# Day 4 — The Write Request Path: `__drbd_make_request()` to `bio_endio()`

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_req.c`, `drbd/drbd_req.h`, `drbd/drbd_main.c`

---

## 1. The Entry Point

Every block I/O submitted to `/dev/drbdN` enters DRBD via the `submit_bio` callback set in `drbd_ops` (Day 1). The entry chain is:

```bash
grep -n "drbd_submit_bio\b\|__drbd_make_request\b" drbd/drbd_req.c drbd/drbd_main.c | head -10
```

```c
// drbd/drbd_req.c
void drbd_submit_bio(struct bio *bio);
    // → minimal wrapper, eventually calls:
void __drbd_make_request(struct drbd_device *device,
                          struct bio *bio,
                          ktime_t start_kt,
                          unsigned long start_jif);
    // ← this is where the real work happens
```

The `struct bio` carries:
- `bio->bi_iter.bi_sector` — starting sector (512-byte units)
- `bio->bi_iter.bi_size`   — byte count
- `bio->bi_opf`            — operation flags (REQ_OP_WRITE, REQ_OP_READ, REQ_FUA, REQ_PREFLUSH…)
- `bio->bi_io_vec[]`       — scatter-gather list of data pages

> **Note:** Older DRBD versions used `__drbd_make_request()` registered via `blk_queue_make_request()`. In DRBD 9.2 + modern kernels, the block layer calls `.submit_bio` directly. Throughout this day, when you see `__drbd_make_request()` in older docs/code, mentally substitute `__drbd_make_request()`.

---

## 2. `struct drbd_request` — The Core Object

```bash
grep -n "struct drbd_request {" drbd/drbd_req.h
# Read every field
```

```c
struct drbd_request {
    struct drbd_device *device;        // which volume
    struct drbd_interval i;            // sector + size (for interval tree)
                                       // i.sector, i.size, i.local, i.completed
    struct bio *master_bio;            // the original bio from the application
    struct bio *private_bio;           // cloned bio submitted to local disk
    //                                 // NULL if device is diskless

    spinlock_t rq_lock;                // ← per-request lock (in addition to req_lock)
    unsigned int local_rq_state;       // ← LOCAL I/O state bits (single u32)
    u16 net_rq_state[DRBD_NODE_ID_MAX];// ← per-peer network state (one slot per node)

    // ── Note: the actual struct in DRBD 9.2 splits state across these fields,
    //         NOT a single rq_state[1+DRBD_PEERS_MAX] array.

    u64 dagtag_sector;                 // Data Generation Tag — global write sequence
                                       // used to enforce ordering on secondaries

    struct list_head tl_requests;      // node in resource->transfer_log
    struct list_head req_pending_master_completion;
    struct list_head req_pending_local; // waiting for local disk ack

    unsigned int epoch;                // which write epoch this belongs to

    atomic_t completion_ref;           // when 0: master_bio may be completed
    struct kref kref;                  // when 0: drbd_request may be freed

    // timestamps for latency tracking
    ktime_t start_kt;
    unsigned long start_jif;
    unsigned long pre_submit_jif;
    unsigned long in_actlog_jif;
};
```
```

---

## 3. State Bits — Every Bit Explained

```bash
grep -n "RQ_LOCAL_PENDING\|RQ_NET_PENDING\|RQ_NET_SENT\|RQ_NET_OK\|RQ_LOCAL_OK\|RQ_WRITE\|RQ_COMPLETION_SUSP\|RQ_EXP_BARR_ACK\|RQ_EXP_WRITE_ACK\|RQ_EXP_RECEIVE_ACK" \
    drbd/drbd_req.h
```

State bits split across two fields:

**`local_rq_state` bits (one per request):**

| Bit | Meaning |
|---|---|
| `RQ_LOCAL_PENDING` | Local disk write submitted, not yet completed |
| `RQ_LOCAL_OK` | Local disk write completed successfully |
| `RQ_LOCAL_ABORTED` | Local disk write failed |
| `RQ_LOCAL_COMPLETED` | Local I/O processing finished (OK or aborted) |
| `RQ_WRITE` | This is a write request (not a read) |
| `RQ_IN_ACT_LOG` | This request has reserved an activity log slot |
| `RQ_POSTPONED` | Postponed for any reason (overlap, suspended I/O) |
| `RQ_COMPLETION_SUSP` | Completion is suspended (fencing active) |
| `RQ_UNPLUG` | This request triggers a queue unplug |

**`net_rq_state[node_id]` bits (one set per peer):**

| Bit | Meaning |
|---|---|
| `RQ_NET_PENDING` | Waiting for network acknowledgement from this peer |
| `RQ_NET_SENT` | Data sent over network (at least into TCP buffer) |
| `RQ_NET_OK` | Peer confirmed the write (protocol C: written to peer disk) |
| `RQ_NET_DONE` | Network processing complete (either OK or failed) |
| `RQ_EXP_WRITE_ACK` | Expecting P_WRITE_ACK (protocol C) |
| `RQ_EXP_RECEIVE_ACK` | Expecting P_RECV_ACK (protocol B) |
| `RQ_EXP_BARR_ACK` | Waiting for barrier acknowledgement |

**The rule:** The master bio is completed (and `bio_endio()` called) only when `req->completion_ref` drops to zero. That refcount is decremented by both local completion and per-peer ACK paths.

---

## 4. Full Write Path — Annotated Call Chain

```bash
grep -n -A 200 "^void __drbd_make_request\b" drbd/drbd_req.c
```

Step-by-step with exact function calls (note: state-bit notation below uses `local_rq_state` and `net_rq_state[idx]` per Section 3):

```
drbd_submit_bio(bio)
  └── __drbd_make_request(device, bio, start_kt, start_jif)
│
├─ 1. Sanity checks
│   ├── if (!get_ldev_if_state(device, D_UP_TO_DATE)) → bio_endio(EIO)
│   │       // get_ldev_if_state atomically increments device->local_cnt
│   │       // and verifies disk_state >= required state
│   └── if (bio_data_dir(bio) == WRITE && !drbd_suspended(device)) ...
│
├─ 2. Allocate drbd_request from mempool
│   └── req = drbd_req_new(device, bio)
│           → mempool_alloc(&drbd_request_mempool, GFP_NOIO)
│           → req->master_bio    = bio
│           → req->device        = device
│           → resource->dagtag_sector += req->i.size >> 9  (writes only)
│           → req->dagtag_sector = resource->dagtag_sector
│
├─ 3. Activity log reservation (if write)
│   └── drbd_al_begin_io_fastpath(device, &req->i)        ← cache HIT path
│        OR drbd_al_begin_io_nonblock(device, &req->i)    ← async MISS path
│           → on miss: queue an AL transaction; the write is parked
│             until drbd_al_begin_io_commit() runs
│           → sets RQ_IN_ACT_LOG in local_rq_state on success
│
├─ 4. Conflict check + insert into interval tree
│   └── drbd_conflict_submit_write(req)          ← drbd_req.c:2107
│           → drbd_find_conflict(&device->requests, &req->i)
│           → drbd_insert_interval(&device->requests, &req->i)
│           → if no conflict: set INTERVAL_SUBMITTED, proceed
│           → if conflict found: park in tree, deferred via submit_conflict wq
│
├─ 5. Insert into transfer log (write ordering)
│   └── list_add_tail(&req->tl_requests, &resource->transfer_log)
│           // held under resource->req_lock spinlock
│
├─ 6. Determine required acks based on replication protocol
│   └── for each peer_device in Established state:
│         net_rq_state[peer_idx] |= RQ_NET_PENDING
│         if protocol == C: net_rq_state[peer_idx] |= RQ_EXP_WRITE_ACK
│         if protocol == B: net_rq_state[peer_idx] |= RQ_EXP_RECEIVE_ACK
│         atomic_inc(&req->completion_ref)   // one ref per pending peer
│
├─ 7. Submit to local disk (if device has disk)
│   ├── req->private_bio = bio_alloc_clone(...)
│   ├── req->private_bio->bi_end_io = drbd_request_endio
│   ├── local_rq_state |= RQ_LOCAL_PENDING
│   ├── atomic_inc(&req->completion_ref)    // ref for local I/O
│   └── submit_bio_noacct(req->private_bio)
│
├─ 8. Enqueue for network send
│   └── drbd_queue_write(device, req)
│           // queues the request for the connection's sender thread
│           // actual send happens in drbd_sender.c (drbd_send_dblock and friends)
│
└─ 9. Return
       // bio is NOT yet complete — completion happens asynchronously
       // when req->completion_ref drops to 0
```

---

## 5. Local Completion: `drbd_request_endio()`

Called when the local disk write finishes (interrupt/softirq context → workqueue):

```bash
grep -n -A 60 "^void drbd_request_endio\b" drbd/drbd_req.c
```

```c
void drbd_request_endio(struct bio *bio)
{
    struct drbd_request *req = bio->bi_private;
    struct drbd_device *device = req->device;

    // Was there an I/O error?
    if (bio->bi_status) {
        req->private_bio = ERR_PTR(-EIO);
        __req_mod(req, READ_COMPLETED_WITH_ERROR, ...);  // or WRITE_COMPLETED_WITH_ERROR
    } else {
        __req_mod(req, WRITE_COMPLETED_WITH_ERROR == 0, ...);
        // Actually: calls req_mod() with appropriate completion event
    }
}
```

Then `__req_mod()` clears `RQ_LOCAL_PENDING`, sets `RQ_LOCAL_OK`, and calls `drbd_req_complete()` if all bits are now clear.

---

## 6. `__req_mod()` — The Request State Machine Driver

This is the most important function in `drbd_req.c`. It drives the state machine for each request. **Note:** The actual mechanism in DRBD 9.2 uses `mod_rq_state()` and `req_mod()` together, with completion driven by `req->completion_ref` (an atomic counter) rather than by checking individual bits.

### 6.1 Caller Wrappers

Three wrappers exist with different locking contracts:

```
__req_mod()   ← raw; caller already holds state_rwlock or is in irq context
_req_mod()    ← calls __req_mod then complete_master_bio() outside any lock
req_mod()     ← acquires read_lock_irq(state_rwlock), calls __req_mod, then complete_master_bio()
```

`req_mod()` (`drbd_req.h:319`) is used by the ack receiver thread; `_req_mod()` is used inside
sections that already hold the transfer-log spinlock; `__req_mod()` is used directly in bio endio
and sender paths that save/restore irq flags themselves.

### 6.2 Call Sites by Lifecycle Phase

#### Phase 1 — Request setup (`drbd_req.c`, inside `tl_update_lock`)

| Event | Line | Meaning |
|---|---|---|
| `TO_BE_SUBMITTED` | `drbd_req.c:2068` | About to submit bio to local disk |
| `NEW_NET_WRITE` | `drbd_req.c:1690` | Write queued for network send |
| `NEW_NET_READ` | `drbd_req.c:2029` | Read routed to remote peer |
| `NEW_NET_OOS` | `drbd_req.c:1692` | No live connection — will send P_OUT_OF_SYNC instead |
| `ADDED_TO_TRANSFER_LOG` | `drbd_req.c:1705` | Request enqueued to sender thread |
| `BARRIER_SENT` | `drbd_req.c:1663` | Empty flush — mark barrier sent to peer |

#### Phase 2 — Local disk completion (`drbd_sender.c:359`, bio endio callback)

Called from the bio completion handler. Uses `__req_mod` directly (not `_req_mod`) because
irq flags are already saved with `read_lock_irqsave`.

| Event | Condition |
|---|---|
| `COMPLETED_OK` | disk read/write succeeded |
| `WRITE_COMPLETED_WITH_ERROR` | disk write failed |
| `READ_COMPLETED_WITH_ERROR` | disk read failed |
| `READ_AHEAD_COMPLETED_WITH_ERROR` | readahead failed (non-fatal to upper layer) |
| `DISCARD_COMPLETED_NOTSUPP` / `DISCARD_COMPLETED_WITH_ERROR` | discard failed |

#### Phase 3 — Network send completion (`drbd_sender.c:3588`, after `drbd_send_dblock()`)

Sender thread calls `__req_mod` under `read_lock_irq(state_rwlock)` after each send attempt:

| Event | Condition |
|---|---|
| `HANDED_OVER_TO_NETWORK` | data packet sent successfully |
| `SEND_FAILED` | send returned error |
| `OOS_HANDED_TO_NETWORK` | `P_OUT_OF_SYNC` sent to peer |
| `SEND_CANCELED` | `drbd_sender.c:3701` — connection dropped while request was queued |

#### Phase 4 — Network ACK reception (`drbd_receiver.c` via `validate_req_change_req_state()`)

Ack receiver thread calls `req_mod()` (locking wrapper) when reply packets arrive. The request
is found by sector address in the interval tree, then driven forward:

| Packet received | Event passed to `__req_mod` |
|---|---|
| `P_WRITE_ACK` | `WRITE_ACKED_BY_PEER` — Protocol C write confirmed durable on peer |
| `P_WRITE_ACK_IN_SYNC` | `WRITE_ACKED_BY_PEER_AND_SIS` — durable + cleared from OOS bitmap |
| `P_RECV_ACK` | `RECV_ACKED_BY_PEER` — Protocol B: peer received data |
| `P_NEG_ACK` | `NEG_ACKED` — peer rejected or lost the write |
| (read reply data) | `DATA_RECEIVED` — remote read data received (`drbd_receiver.c:2646`) |

#### Phase 5 — Barrier acknowledgment (`drbd_main.c:448`, in `tl_release()`)

`tl_release()` is called when `P_BARRIER_ACK` arrives. It walks the matching epoch in the
transfer log and calls `req_mod(req, BARRIER_ACKED, ...)` for every request in that epoch.
This is the Protocol A durability point — the peer confirms all preceding writes are stable.

#### Phase 6 — Connection loss (`drbd_main.c:501`, via `__tl_walk()`)

On disconnect, DRBD walks the entire transfer log and applies `CONNECTION_LOST` or
`CONNECTION_LOST_WHILE_SUSPENDED` to every request still holding `RQ_NET_PENDING`. This
drives OOS bitmap marking for all in-flight writes that never reached the peer.

### 6.3 State machine pattern (simplified)

```c
void __req_mod(struct drbd_request *req, enum drbd_req_event what,
               struct drbd_peer_device *peer_device,
               struct bio_and_error *m)
{
    // m is set to {master_bio, error} when completion is triggered
    switch (what) {

    case WRITE_COMPLETED_WITH_ERROR:
        // local disk write failed
        // → mark request failed, possibly detach disk
        mod_rq_state(req, m, RQ_LOCAL_PENDING, RQ_LOCAL_COMPLETED|RQ_LOCAL_ABORTED);
        // → atomic_dec(&req->completion_ref)

    case COMPLETED_OK:
        // local disk write OK
        mod_rq_state(req, m, RQ_LOCAL_PENDING, RQ_LOCAL_COMPLETED|RQ_LOCAL_OK);
        if (local_rq_state & RQ_IN_ACT_LOG)
            drbd_al_complete_io(device, &req->i);
        atomic_dec(&req->completion_ref);
        break;

    case WRITE_ACKED_BY_PEER:         // P_WRITE_ACK received
        net_rq_state[peer_idx] &= ~RQ_NET_PENDING;
        net_rq_state[peer_idx] |=  RQ_NET_OK | RQ_NET_DONE;
        atomic_dec(&req->completion_ref);
        break;

    case RECV_ACKED_BY_PEER:          // P_RECV_ACK received
        net_rq_state[peer_idx] &= ~RQ_NET_PENDING;
        net_rq_state[peer_idx] |=  RQ_NET_OK;
        atomic_dec(&req->completion_ref);
        break;

    case BARRIER_ACKED:               // P_BARRIER_ACK received
        // All writes before this barrier are now durable on peer
        net_rq_state[peer_idx] |= RQ_NET_DONE;
        break;

    case SEND_CANCELED:               // peer disconnected before send
        net_rq_state[peer_idx] &= ~RQ_NET_PENDING;
        net_rq_state[peer_idx] |=  RQ_NET_DONE;
        atomic_dec(&req->completion_ref);
        break;
    }

    // After every state change: check if completion_ref hit zero
    // If so: drbd_req_complete() → bio_endio() of master_bio
}
```

---

## 7. Completion via `req->completion_ref`

In DRBD 9.2, completion is driven by a refcount, not by checking individual bits:

```bash
grep -n "completion_ref\|drbd_req_complete\|req_destroy_after_send_acks" drbd/drbd_req.c | head -20
```

The pattern:

```c
struct drbd_request {
    ...
    atomic_t completion_ref;   // when 0: master_bio may be completed
    struct kref kref;          // when 0: drbd_request itself may be freed
    ...
};
```

Each ref-taking event (initial creation, local I/O submission, per-peer pending) increments `completion_ref`. Each completion event (local endio, P_WRITE_ACK, etc.) decrements it.

When `completion_ref` reaches zero, `drbd_req_complete()` triggers the master bio:

```c
if (atomic_dec_and_test(&req->completion_ref)) {
    // All pending work done
    drbd_remove_interval(&device->requests, &req->i);
    list_del_init(&req->tl_requests);
    wake_up(&device->misc_wait);

    m->bio   = req->master_bio;
    m->error = (local_rq_state & RQ_LOCAL_ABORTED) ? -EIO : 0;
    // The caller (outside the lock) calls bio_endio(m->bio, m->error)

    // Then the kref drops:
    kref_put(&req->kref, drbd_req_destroy);
    // (deferred — peer-ack tracking may still hold references)
}
```

This ref-based design lets DRBD cleanly handle: local + N peers in protocol C, multiple peer ACK paths, peer ACK forwarding (Day 22), and crash-time cleanup.

---

## 8. The Replication Protocols: When Does `bio_endio()` Fire?

| Protocol | `RQ_LOCAL_PENDING` must clear | `RQ_NET_PENDING` must clear | When? |
|---|---|---|---|
| **A** (async) | Yes | Yes, but cleared when data *sent* (not acked) | Very fast; data loss risk |
| **B** (semi-sync) | Yes | Yes, cleared on P_RECV_ACK | Data in peer RAM |
| **C** (sync) | Yes | Yes, cleared on P_WRITE_ACK | Data on peer disk — default |

```bash
grep -n "RQ_EXP_WRITE_ACK\|RQ_EXP_RECEIVE_ACK\|drbd_prot_C\|prot_A\|prot_B" \
    drbd/drbd_req.c drbd/drbd_int.h
grep -n "on_no_data\|wire_protocol\|dp_flags" drbd/drbd_int.h | head -20
```

### Dual-primary with Protocol C

`two_primaries = yes` is enforced to require Protocol C (`drbd_nl.c:3949`).  In
this mode each node simultaneously runs the write path above for its **own**
writes, AND runs `receive_Data()` for the peer's writes.  The `completion_ref`
for a write on NodeA starts at 2 (local + NodeB peer), and only drops to zero
when both NodeA's local disk completes **and** NodeB sends `P_WRITE_ACK`.
NodeB's own writes run the same path in reverse.

The critical extra step on the **receiver** side (NodeB receiving NodeA's write):
```c
// drbd_receiver.c receive_Data(), two_primaries path
err = wait_for_and_update_peer_seq(peer_device, d.peer_seq); // in-order delivery
err = drbd_peer_write_conflicts(peer_req);  // detect same-sector concurrent write
```
If `drbd_peer_write_conflicts()` finds NodeB has a `INTERVAL_LOCAL_WRITE` at the
same sector, it returns `-EBUSY` and DRBD disconnects.  See day12 Section 8 and
day22 Section 8 for the full treatment.

---

## 9. The Transfer Log and Write Ordering

```bash
grep -n "transfer_log\|tl_update_lock\|tl_requests\|dagtag_sector" drbd/drbd_req.c drbd/drbd_sender.c | head -30
```

`resource->transfer_log` is a linked list of all in-flight `drbd_request` objects. It is the mechanism by which DRBD enforces identical write ordering on both the local disk and all peer nodes.

### 9.1 Why ordering matters

The Linux block layer I/O scheduler reorders writes before submission (sorting by LBA, merging adjacent requests, etc.) to improve throughput. By the time the block layer calls `drbd_make_request()`, it has already chosen a submission order. DRBD's job is to ensure both the local disk and every peer see writes in **that same order** — not some random order determined by network or disk latency.

### 9.2 Step 1 — dagtag assignment and list insertion are serialized (`drbd_req.c:1988`)

```bash
grep -n "tl_update_lock\|dagtag_sector\|list_add_tail_rcu.*tl_requests" drbd/drbd_req.c
```

```c
spin_lock(&resource->tl_update_lock);   /* serializes all drbd_make_request() calls */
    resource->dagtag_sector += req->i.size >> 9;   /* monotonically increasing */
    req->dagtag_sector = resource->dagtag_sector;
    ...
    list_add_tail_rcu(&req->tl_requests, &resource->transfer_log);
spin_unlock(&resource->tl_update_lock);
```

`tl_update_lock` ensures every write gets a unique, strictly increasing dagtag and is appended to the **tail** of the transfer log. The list is always in submission order.

### 9.3 Step 2 — the sender never skips a pending request (`drbd_sender.c:3248`)

```bash
grep -n -A 20 "^static struct drbd_request \*__next_request_for_connection" drbd/drbd_sender.c
```

```c
static struct drbd_request *__next_request_for_connection(struct drbd_connection *connection)
{
    list_for_each_entry_rcu(req, &resource->transfer_log, tl_requests) {
        unsigned s = req->net_rq_state[connection->peer_node_id];

        /* Pending but not yet queued: STOP — do not skip over it */
        if (unlikely(s & RQ_NET_PENDING && !(s & (RQ_NET_QUEUED|RQ_NET_SENT))))
            return NULL;   /* sender goes back to sleep */

        if (s & RQ_NET_QUEUED)
            return req;    /* ready: send this one next */
    }
    return NULL;
}
```

The critical line is `return NULL` — the sender **stops and sleeps** if it encounters a request that needs to go to the network but is not yet queued. It never jumps over request N to send request N+1. The peer therefore always receives `P_DATA` packets in list order — which is dagtag order — which is submission order.

### 9.4 Local disk completion order does not affect send order

The sender walks the transfer log by **list position**, not by local disk completion. The local disk may complete write N+1 before write N (elevator reordering inside the backing device), but that does not move any request's position in the transfer log. The network path and the local disk path are completely independent in terms of ordering.

### 9.5 Single-primary vs dual-primary

In **single-primary** mode: ordering is fully enforced by the sender (steps above). The receiver processes `P_DATA` packets as they arrive — in order — with no extra buffering needed.

In **dual-primary** mode: two primaries send independently to each other. Each primary maintains its own transfer log and dagtag space, so `P_DATA` streams from A→B and B→A are independent. The receiver uses `wait_for_and_update_peer_seq()` (`drbd_receiver.c:2903`) to detect and resolve conflicts between concurrent writes from both primaries — this only runs when `RESOLVE_CONFLICTS` is set.

### 9.6 Second purpose: recovery after disconnect

When a peer disconnects, DRBD walks the transfer log to find all requests that have `RQ_NET_PENDING` set for that peer (i.e., sent but not yet acknowledged, or not yet sent). Those sectors are marked out-of-sync in the peer's bitmap so they will be resynced on reconnect.

```bash
grep -n "RQ_NET_PENDING\|set_bit.*OUT_OF_SYNC\|drbd_set_out_of_sync" drbd/drbd_req.c | head -15
```

---

## 10. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Read `drbd_req.c` fully
It is ~1800 lines. For every function, write its name and one-line description. Pay special attention to:
- `__drbd_make_request()` — the entry point
- `__req_mod()` — the state machine
- `drbd_req_complete()` — the completion check

### Exercise 2 (30 min): Enumerate all `enum drbd_req_event` values
```bash
grep -n "enum drbd_req_event\b" drbd/drbd_req.h
```
For each event, identify: who sends it, what `rq_state` bits it clears/sets.

### Exercise 3 (40 min): Trace a failed local write
Start from `drbd_request_endio()` being called with `bio->bi_status != 0`. What state bits change? Does `bio_endio()` still get called? Does DRBD detach the disk? Under what conditions?

```bash
grep -n "WRITE_COMPLETED_WITH_ERROR\|drbd_handle_failed_mirror\|drbd_detach" drbd/drbd_req.c
```

### Exercise 4 (40 min): Read `get_ldev_if_state()` and understand `local_cnt`
```bash
grep -n "get_ldev_if_state\|put_ldev\|atomic.*local_cnt" drbd/drbd_int.h drbd/drbd_req.c
```
Why does DRBD need `local_cnt`? What would happen if a disk detach ran concurrently with an in-flight write?

### Exercise 5 (30 min): Find all callers of `__req_mod()`
```bash
grep -n "__req_mod\b" drbd/drbd_req.c drbd/drbd_receiver.c drbd/drbd_sender.c
```
For each call site, identify: what event is being sent, from which file/function, and why.

---

## Summary

`__drbd_make_request()` wraps each incoming bio in a `drbd_request`, reserves an activity log slot, inserts the request into the interval tree and transfer log, submits a cloned bio to the local disk, and enqueues the request for the sender thread. The `rq_state` bitmask tracks what must complete before `bio_endio()` is called. `__req_mod()` drives the request state machine from both the local disk completion callback and the network ACK handler. The replication protocol (A/B/C) determines which network events are required.

**Next:** Day 5 — The network protocol: packet format, sender thread, barriers and epochs.
