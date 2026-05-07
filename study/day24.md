# Day 24 — End-to-End I/O Walkthrough: One Write, All Locks, All Threads

> **Estimated study time: 3–4 hours**
> **Primary files:** All of `drbd_req.c`, `drbd_actlog.c`, `drbd_sender.c`, `drbd_receiver.c`, `drbd_interval.c`

---

## 1. The Setup

- 2-node cluster: **Primary** (node A) and **Secondary** (node B)
- Replication protocol: **C** (synchronous — wait for peer disk write)
- Local disk: UpToDate on both sides
- Application: PostgreSQL writing a 4 KiB page to `/dev/drbd0`

We trace this single 4 KiB write from the moment the kernel hands it to DRBD until `bio_endio()` is called.

---

## 2. Thread Map

```
Node A (Primary):                    Node B (Secondary):
┌─────────────────────────┐         ┌─────────────────────────┐
│ PostgreSQL process       │         │                         │
│  (user context)          │         │                         │
├─────────────────────────┤         ├─────────────────────────┤
│ __drbd_make_request()    │         │                         │
│  (caller's process ctx)  │         │                         │
├─────────────────────────┤         ├─────────────────────────┤
│ A: sender thread         │──NET──▶│ B: receiver thread      │
│  drbd_sender()           │         │  drbd_receiver()        │
├─────────────────────────┤         ├─────────────────────────┤
│ A: local disk endio      │         │ B: local disk endio     │
│  (softirq / workqueue)   │         │  (softirq / workqueue)  │
├─────────────────────────┤         ├─────────────────────────┤
│ A: receiver thread       │◀──NET──│ B: ack_sender workqueue │
│  (got_WriteAck)          │         │  (drbd_send_acks_wf →   │
│                          │         │   e_end_block →         │
│                          │         │   drbd_send_ack)        │
└─────────────────────────┘         └─────────────────────────┘
```

---

## 3. Phase 1: Application → `__drbd_make_request()` [Node A, process context]

```bash
grep -n "^void __drbd_make_request\b" drbd/drbd_req.c
```

```
[A, process ctx] PostgreSQL: write(fd, buf, 4096)
  → VFS → page cache writeback → struct bio
  → submit_bio(bio)
  → block layer → drbd_submit_bio(bio) → __drbd_make_request(device, bio, ...)

LOCK ACQUIRED: none yet

1. get_ldev_if_state(device, D_UP_TO_DATE)
   → atomic_inc(&device->local_cnt)   ← local_cnt = 1

2. req = mempool_alloc(&drbd_request_mempool, GFP_NOIO)
   req->master_bio    = bio
   req->dagtag_sector = atomic64_inc_return(&resource->dagtag_sector)
   req->i.sector      = bio->bi_iter.bi_sector
   req->i.size        = 4096
   atomic_set(&req->completion_ref, 1)   ← initial ref

LOCK ACQUIRED: spin_lock_irq(&resource->req_lock)

3. drbd_insert_interval(&device->write_requests, &req->i)
   → rb_insert_augmented() — O(log n)

4. list_add_tail(&req->tl_requests, &resource->transfer_log)

5. req->local_rq_state     |= RQ_LOCAL_PENDING | RQ_WRITE
   req->net_rq_state[peer1] |= RQ_NET_PENDING | RQ_EXP_WRITE_ACK
   atomic_inc(&req->completion_ref)   ← +1 for local I/O
   atomic_inc(&req->completion_ref)   ← +1 for peer1
   // completion_ref is now 3 (1 init + 1 local + 1 peer)

LOCK RELEASED: spin_unlock_irq(&resource->req_lock)

6. drbd_al_begin_io_fastpath(device, &req->i) → HIT
   → req->local_rq_state |= RQ_IN_ACT_LOG

7. req->private_bio = bio_alloc_clone(...)
   req->private_bio->bi_end_io = drbd_request_endio
   req->private_bio->bi_private = req

8. submit_bio_noacct(req->private_bio)
   → local disk I/O starts asynchronously

9. drbd_queue_write(peer_device, req)
   → spin_lock(&connection->sender_work.q_lock)
   → list_add_tail(&req->w.list, &connection->sender_work.q)
   → spin_unlock(...)
   → wake_up(&connection->sender_work.q_wait)  ← wakes sender thread

10. atomic_dec(&req->completion_ref)   ← drop initial ref → now 2
    // bio NOT yet complete — completion_ref must hit 0
RETURN
```

---

## 4. Phase 2: Local Disk Write [Node A, softirq/workqueue context]

```bash
grep -n "drbd_request_endio\b" drbd/drbd_req.c
grep -n -A 40 "^void drbd_request_endio\b" drbd/drbd_req.c
```

```
[A, softirq] Local disk driver calls drbd_request_endio(private_bio)

bio->bi_status = BLK_STS_OK

LOCK ACQUIRED: spin_lock_irqsave(&resource->req_lock, flags)

__req_mod(req, COMPLETED_OK, NULL, &m)
  → req->local_rq_state &= ~RQ_LOCAL_PENDING
  → req->local_rq_state |=  RQ_LOCAL_OK | RQ_LOCAL_COMPLETED
  → drbd_al_complete_io(device, &req->i)
       → lc_put(device->act_log, e)   ← refcnt-- (still > 0 if other writes)
  → atomic_dec(&req->completion_ref)   ← now 1 (peer1 still pending)

  // completion_ref > 0 → bio NOT yet complete — waiting for peer ACK

LOCK RELEASED: spin_unlock_irqrestore(...)

m.bio == NULL  → no bio_endio() yet
```

---

## 5. Phase 3: Sender Thread Sends `P_DATA` [Node A, sender thread]

```bash
grep -n "drbd_send_dblock\b" drbd/drbd_sender.c
grep -n -A 80 "^int drbd_send_dblock\b" drbd/drbd_sender.c
```

```
[A, sender thread] woken by wake_up() in step 9

drbd_sender() main loop:
  → drains connection->sender_work.q
  → finds req->w (the drbd_request we queued)
  → calls req->w.cb(req->w, 0)
       = w_send_dblock() or equivalent

drbd_send_dblock(peer_device, req):
  → Build p_data header:
       p.sector   = cpu_to_be64(req->i.sector)   = sector 8 (for 4K at start)
       p.block_id = (u64)(uintptr_t)req            ← THE POINTER TRICK
       p.dp_flags = DP_SEND_WRITE_ACK              ← protocol C
       p.seq_num  = atomic_inc(&peer_device->packet_seq)

  → drbd_send_command(peer_device, DATA_STREAM, P_DATA, &p, sizeof(p))
       → transport->ops->send_page(DATA socket, header page, ...)
       → kernel_sendpage() to TCP socket

  → drbd_send_bio(peer_device, req->master_bio)
       → transport->ops->send_page() for each bio page
       → kernel_sendpage(DATA socket, data_page, 0, 4096, MSG_MORE)

  → atomic_inc(&connection->ap_in_flight)
  → req->net_rq_state[peer1] |= RQ_NET_SENT  (under req->rq_lock)

TCP: 4096 + header bytes on the wire to Node B
```

---

## 6. Phase 4: Receiver Processes `P_DATA` [Node B, receiver thread]

```bash
grep -n "receive_Data\b" drbd/drbd_receiver.c
grep -n -A 200 "^static int receive_Data\b" drbd/drbd_receiver.c | head -100
```

```
[B, receiver thread] reads from DATA socket

drbd_recv_header(connection, &pi)
  → reads 16 bytes: magic=DRBD_MAGIC3, cmd=P_DATA, vol=0, length=4096+28

dispatch → receive_Data(connection, &pi)

1. peer_device = idr_find(&connection->peer_devices, pi->vnr)

2. Parse p_data:
   sector   = 8
   block_id = <address of req on node A>
   dp_flags = DP_SEND_WRITE_ACK

3. peer_req = drbd_alloc_peer_req(peer_device, block_id, sector, 4096)
   → mempool_alloc(drbd_ee_mempool)

4. drbd_recv_all_warn(connection, DATA_STREAM, peer_req->pages, 4096)
   → kernel_recvmsg(DATA_socket, ...) — reads 4096 data bytes

LOCK ACQUIRED: spin_lock_irq(&resource->req_lock)

5. drbd_find_overlap(&device->write_requests, sector, 4096)
   → NULL (no concurrent writes to this range on secondary)

6. drbd_insert_interval(&device->write_requests, &peer_req->i)

LOCK RELEASED: spin_unlock_irq(...)

7. spin_lock(&connection->active_ee_lock)
   list_add_tail(&peer_req->w.list, &connection->active_ee)
   spin_unlock(...)

8. peer_req->epoch = connection->current_epoch
   atomic_inc(&epoch->epoch_size)

9. peer_req->w.cb = e_end_block

10. drbd_submit_peer_request(device, peer_req, REQ_OP_WRITE, ...)
    → bio = bio_alloc_bioset(...)
    → bio_add_page(bio, peer_req->pages, 4096, 0)
    → bio->bi_end_io = drbd_peer_request_endio
    → generic_make_request(bio)   ← local disk write starts
```

---

## 7. Phase 5: Secondary Disk Write Completes [Node B, softirq]

```bash
grep -n "drbd_peer_request_endio\b" drbd/drbd_receiver.c
grep -n -A 40 "^void drbd_peer_request_endio\b" drbd/drbd_receiver.c
```

```
[B, softirq] Local disk calls drbd_peer_request_endio(bio)

bio->bi_status = BLK_STS_OK

spin_lock_irqsave(&connection->active_ee_lock, flags)
list_move_tail(&peer_req->w.list, &connection->done_ee)
spin_unlock_irqrestore(...)

drbd_queue_work(&connection->sender_work, &peer_req->w)
→ wakes B's sender thread
```

---

## 8. Phase 6: Node B Sends `P_WRITE_ACK` [Node B, **ack_sender workqueue**]

> **Important:** ACKs are sent from a dedicated **`ack_sender` workqueue** (a `kthread_worker`), NOT from the regular sender thread. This decouples ACK latency from large data sends.

```bash
grep -n "e_end_block\b" drbd/drbd_receiver.c
grep -n "queue_work.*ack_sender\|drbd_send_acks_wf" drbd/drbd_sender.c drbd/drbd_receiver.c | head -10
grep -n -A 60 "^static int e_end_block\b" drbd/drbd_receiver.c
```

```
[B, ack_sender workqueue] dispatches send_acks_work → drbd_send_acks_wf()
   → walks done_ee, calls peer_req->w.cb (= e_end_block) per item

e_end_block(w, cancel):

  → test_bit(EE_WAS_ERROR, &peer_req->flags) = false (write succeeded)

  → drbd_send_ack(peer_device, P_WRITE_ACK, peer_req)
       → Build p_block_ack:
            p.sector   = cpu_to_be64(8)
            p.block_id = peer_req->block_id  ← original block_id from P_DATA
                       = <address of req on node A>
            p.blksize  = cpu_to_be32(4096)
            p.seq_num  = ...

       → drbd_send_command(peer_device, CONTROL_STREAM, P_WRITE_ACK, &p, sizeof(p))
            → kernel_sendpage(META socket, ...)

  → drbd_may_finish_epoch(connection, epoch, EV_PUT)
       → if all writes in epoch done: maybe send P_BARRIER_ACK

  → drbd_free_peer_req(device, peer_req)

TCP META socket: P_WRITE_ACK on the wire back to Node A
```

---

## 9. Phase 7: Node A Receives `P_WRITE_ACK` and Completes [Node A, receiver thread]

```bash
grep -n "got_WriteAck\b" drbd/drbd_receiver.c
grep -n -A 50 "^static int got_WriteAck\b" drbd/drbd_receiver.c
```

```
[A, receiver thread] reads from META socket

drbd_recv_header(connection, &pi) → cmd=P_WRITE_ACK

dispatch → got_WriteAck(connection, &pi)

1. Parse p_block_ack:
   block_id = <address of original req on node A>
   sector   = 8

2. req = (struct drbd_request *)(uintptr_t)block_id
   ← O(1) lookup: no hash table, no search!

3. Validate: req->device == peer_device->device ✓
             req->i.sector == sector ✓

LOCK ACQUIRED: spin_lock_irq(&resource->req_lock)

4. __req_mod(req, WRITE_ACKED_BY_PEER, peer_device, &m)
     → req->net_rq_state[peer1] &= ~RQ_NET_PENDING
     → req->net_rq_state[peer1] |=  RQ_NET_OK | RQ_NET_DONE
     → atomic_dec(&connection->ap_in_flight)
     → atomic_dec(&req->completion_ref)   ← now 0!

     → drbd_req_complete(req, &m):
          completion_ref hit zero → ALL DONE!

          drbd_remove_interval(&device->write_requests, &req->i)
          list_del_init(&req->tl_requests)
          wake_up(&device->misc_wait)  ← if anyone waiting on this range

          m.bio   = req->master_bio   ← PostgreSQL's original bio
          m.error = 0

          kref_put(&req->kref, drbd_req_destroy)
          // (deferred — peer ack tracking may still hold refs)

LOCK RELEASED: spin_unlock_irq(...)

5. bio_endio(m.bio, 0)
   → PostgreSQL's write() returns 4096
   → PostgreSQL proceeds

put_ldev(device)   ← local_cnt--
```

---

## 10. Complete Timeline

```
Time →

[A process]  submit_bio → __drbd_make_request → local bio submitted → sender queued → return
                                                        │                    │
[A local disk]                              ←── I/O ───┘                    │
                                             local endio (rq_state local done)│
                                                                              ▼
[A sender]                                                         send P_DATA ──────────────────▶
                                                                                                  │
[B receiver]                                                                       receive P_DATA ─┘
                                                                                   submit local write
                                                                                          │
[B local disk]                                                                   ←── I/O ─┘
                                                                                   local endio
                                                                                          │
[B sender]                                                                 send P_WRITE_ACK ────▶
                                                                                                 │
[A receiver]                                                        receive P_WRITE_ACK ─────────┘
                                                                    bio_endio(master_bio) ← DONE!

Total latency = A_local_write_time + RTT + B_local_write_time
```

---

## 11. Lock Summary

| Lock | Type | Held During |
|---|---|---|
| `resource->req_lock` | `spinlock_t` (irq-safe) | Interval tree ops, transfer_log, multi-request changes |
| `req->rq_lock` | `spinlock_t` | Per-request `local_rq_state` / `net_rq_state[]` updates |
| `connection->active_ee_lock` | `spinlock_t` (irq-save) | Moving peer_req between ee lists |
| `device->al_lock` | `spinlock_t` | Activity log slot reference counting |
| `device->ldev->md_mutex` | `mutex` | Metadata reads/writes |
| `resource->adm_mutex` | `mutex` | Admin command serialisation |
| `connection->sender_work.q_lock` | `spinlock_t` | Work queue enqueue/dequeue |
| `device->bitmap->bm_lock` | `spinlock_t` | Bitmap bit ops, `bm_set` count |

---

## 12. Hands-On Exercises (3–4 hours)

### Exercise 1 (60 min): Annotate `__drbd_make_request()` with lock regions
```bash
grep -n -A 200 "^void __drbd_make_request\b" drbd/drbd_req.c
```
For every `spin_lock` / `spin_unlock` / `mutex_lock` / `mutex_unlock`, draw a bracket showing the locked region and what is protected.

### Exercise 2 (40 min): Trace the protocol A fast path
In Protocol A, `bio_endio()` fires as soon as the P_DATA is **sent** (not when ACK received). Find:
```bash
grep -n "RQ_EXP_WRITE_ACK\|RQ_EXP_RECEIVE_ACK\|DRBD_PROT_A\|DRBD_PROT_B\|DRBD_PROT_C" drbd/drbd_req.c | head -20
```
What `rq_state[peer]` bits are set in protocol A? When is `RQ_NET_PENDING` cleared?

### Exercise 3 (40 min): Trace a READ request
```bash
grep -n "REQ_OP_READ\|drbd_read_remote\|P_DATA_REQUEST\b" drbd/drbd_req.c | head -20
```
For a Primary with UpToDate local disk: is the read sent to the local disk or to the peer?
For a diskless Primary: trace the full path through `P_DATA_REQUEST` → `P_DATA_REPLY`.

### Exercise 4 (35 min): What happens if the peer disconnects mid-write?
Starting from `tl_walk()` being called with `CONNECTION_LOST_WHILE_PENDING`:
```bash
grep -n "CONNECTION_LOST_WHILE_PENDING\|tl_walk\b\|SEND_CANCELED\b" drbd/drbd_req.c drbd/drbd_main.c | head -20
```
Does `bio_endio()` get called? With what error? What happens to the OOS bitmap?

### Exercise 5 (25 min): Verify the block_id pointer trick is safe
```bash
grep -n "block_id\|uintptr_t" drbd/drbd_sender.c drbd/drbd_receiver.c | head -20
```
On a 64-bit system, a kernel pointer is 8 bytes and `u64` is 8 bytes. On a 32-bit system, a pointer is 4 bytes but `u64` is 8 bytes — does this cause problems? What assumptions does DRBD make?

---

## Summary

A single Protocol-C write traverses 7 phases across 4+ threads on 2 nodes. The critical path is: `__drbd_make_request()` sets up `local_rq_state` + per-peer `net_rq_state[]` and increments `req->completion_ref` for local I/O and each pending peer → local bio submitted → P_DATA sent by sender thread → secondary writes locally → `e_end_block()` (running in the **ack_sender workqueue**, not the sender thread) sends `P_WRITE_ACK` → primary's `got_WriteAck()` decrements `completion_ref` → when it hits 0, `drbd_req_complete()` calls `bio_endio()`. The `block_id` pointer trick achieves O(1) request lookup on ACK. The `req_lock` spinlock serialises interval tree and transfer log; per-request `rq_lock` serialises individual state-bit updates.

**Next:** Day 25 — Performance tuning: AL sizing, `resync-rate`, TCP buffer tuning, `no-disk-barrier`, write-ordering modes, and benchmarking methodology.
