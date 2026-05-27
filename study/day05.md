# Day 5 — Network Protocol: Packet Format, Sender Thread & Write Ordering

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd-headers/drbd_protocol.h`, `drbd/drbd_sender.c`, `drbd/drbd_int.h`

---

## 1. The Wire Protocol: Two-Socket Architecture

Each DRBD connection uses **two TCP sockets** (or two RDMA QPs):

```
Primary node                        Secondary node
┌────────────────────────────┐      ┌────────────────────────────┐
│  DATA socket               │──────│  DATA socket               │
│  (P_DATA, P_RS_DATA_REPLY) │      │  (bulk data, one direction)│
├────────────────────────────┤      ├────────────────────────────┤
│  META socket               │──────│  META socket               │
│  (P_BARRIER, P_WRITE_ACK,  │      │  (control messages, ACKs,  │
│   P_STATE, P_PING, ...)    │      │   both directions)         │
└────────────────────────────┘      └────────────────────────────┘
```

Why two sockets?
- Large data transfers on DATA socket cannot block urgent control messages (ACKs, pings)
- Each socket has its own TCP send/receive buffer
- Avoids head-of-line blocking

```bash
grep -n "DATA_STREAM\|CONTROL_STREAM\|DRBD_STREAM" drbd/drbd-headers/drbd_transport.h drbd/drbd_int.h
```

---

## 2. Packet Header Formats

### `p_header80` — original header (DRBD 8 compat)
```c
// drbd/drbd-headers/drbd_protocol.h
struct p_header80 {
    u32 magic;    // DRBD_MAGIC = 0x835a
    u16 command;  // enum drbd_packet — packet type
    u16 length;   // payload length (EXCLUDES header)
} __packed;       // 8 bytes total
```

### `p_header95` — extended header for large payloads
```c
struct p_header95 {
    u16 magic;    // DRBD_MAGIC2 = 0x8620
    u16 command;
    u32 length;   // 32-bit length field — up to 4 GiB payload
} __packed;       // 8 bytes total (same size, different layout)
```

### `p_header100` — DRBD 9 multi-volume header
```c
struct p_header100 {
    u32 magic;    // DRBD_MAGIC3 = 0x835a4600
    u16 volume;   // which volume this packet belongs to
    u16 command;
    u32 length;
    u32 pad;      // alignment
} __packed;       // 16 bytes total
```

```bash
grep -n "struct p_header\|DRBD_MAGIC\|p_header80\|p_header95\|p_header100" drbd/drbd-headers/drbd_protocol.h
```

The receiver checks `magic` first to determine which header format to parse.

---

## 3. `enum drbd_packet` — All Packet Types

```bash
grep -n "enum drbd_packet {" drbd/drbd-headers/drbd_protocol.h
# Read the entire enum — approximately 60 values in DRBD 9
```

Key packets grouped by function:

**Data replication:**
```
P_DATA              Primary→Secondary: replicated write data + header
P_DATA_REPLY        Secondary→Primary: response to P_DATA_REQUEST (peer read)
P_RS_DATA_REPLY     SyncSource→SyncTarget: resync data block
```

**Write ordering / barriers:**
```
P_BARRIER           Primary→Secondary: epoch boundary, sequence number
P_BARRIER_ACK       Secondary→Primary: all writes in epoch committed
```

**Acknowledgements (per replication protocol):**
```
P_WRITE_ACK         Secondary→Primary: write durable on peer disk (protocol C)
P_RECV_ACK          Secondary→Primary: write received into peer RAM (protocol B)
P_NEG_ACK           Secondary→Primary: write failed on peer disk
P_NEG_RS_DREPLY     SyncTarget→SyncSource: resync write failed
P_RS_WRITE_ACK      SyncTarget→SyncSource: resync write OK
```

**Resync control:**
```
P_RS_REQUEST        SyncTarget→SyncSource: request specific block
P_RS_CANCEL         Cancel a pending resync request
P_OV_REQUEST        Verify: request hash of block
P_OV_REPLY          Verify: response with hash
P_OV_RESULT         Verify: result (match or mismatch)
P_OUT_OF_SYNC       Ahead→Behind: notify peer which blocks are OOS
```

**Handshake / connection setup:**
```
P_CONNECTION_FEATURES  First packet: exchange capabilities
P_AUTH_CHALLENGE       HMAC challenge
P_AUTH_RESPONSE        HMAC response
P_INITIAL_META         Metadata version exchange
P_INITIAL_DATA         Data version exchange
P_UUIDS               UUID exchange (DRBD 8 compat)
P_UUIDS110            UUID exchange (DRBD 9, up to 110 history entries)
P_PEER_DAGTAG         Exchange latest written dagtag
```

**State / config:**
```
P_STATE             Advertise local role+disk+repl state to peer
P_STATE_CHG_REQ     Request peer to change its state (demote, etc.)
P_STATE_CHG_REPLY   Response to state change request
P_PROTOCOL          Negotiate replication protocol (A/B/C)
P_TWOPC_PREPARE     Two-phase commit prepare (quorum/promote)
P_TWOPC_COMMIT      Two-phase commit commit
P_TWOPC_ABORT       Two-phase commit abort
```

**Keep-alive:**
```
P_PING              Keep-alive probe
P_PING_ACK          Keep-alive response
```

---

## 4. The `P_DATA` Packet — Structure In Detail

```bash
grep -n "struct p_data {" drbd/drbd-headers/drbd_protocol.h
```

```c
struct p_data {
    struct p_header100 head;   // 16 bytes

    u64 sector;     // starting sector on the secondary (little-endian)
    u64 block_id;   // opaque ID — echoed in P_WRITE_ACK
                    // set to pointer value of drbd_request on primary
                    // used to find the request when ACK arrives
    u32 seq_num;    // write sequence number within connection
    u32 dp_flags;   // DP_HARDBARRIER | DP_RW_SYNC | DP_SEND_WRITE_ACK |
                    // DP_SEND_RECEIVE_ACK | DP_MAY_SET_IN_SYNC
} __packed;         // 16 + 28 = 44 bytes
// Immediately followed by: data pages (bio->bi_iter.bi_size bytes)
```

```bash
grep -n "dp_flags\|DP_HARDBARRIER\|DP_SEND_WRITE_ACK\|DP_MAY_SET_IN_SYNC" drbd/drbd-headers/drbd_protocol.h
```

**`block_id` is a pointer cast to u64.** This is a common trick in DRBD: the primary stores `(u64)(uintptr_t)req` as the block_id in P_DATA, so when the secondary sends P_WRITE_ACK back with the same block_id, the primary can find the drbd_request in O(1) without any hash table lookup.

---

## 5. The Sender Thread — `drbd_sender()`

> **Important architectural note:** DRBD has **two** thread-like producers of network packets per connection:
> 1. **`drbd_sender()` thread** (in `drbd_sender.c`) — sends bulk data: `P_DATA`, `P_RS_DATA_REPLY`, `P_BARRIER`
> 2. **`ack_sender` workqueue** (`connection->ack_sender` — a `kthread_worker`) — sends ACKs: `P_WRITE_ACK`, `P_RECV_ACK`, `P_BARRIER_ACK`, `P_PING_ACK`
>
> This separation prevents large data sends from delaying urgent ACKs. The two share access to the META socket (CONTROL_STREAM) but use separate work queues.

```bash
grep -n "ack_sender\b\|send_acks_work\|drbd_send_acks_wf" drbd/drbd_sender.c drbd/drbd_int.h drbd/drbd_main.c | head -10
```

```bash
grep -n -A 60 "^int drbd_sender\b" drbd/drbd_sender.c
```

Structure of the sender thread main loop:

```c
int drbd_sender(struct drbd_thread *thi)
{
    struct drbd_connection *connection = thi->connection;
    struct drbd_work *w;
    int intr;

    while (1) {
        // Block until work arrives or thread should stop
        intr = wait_event_interruptible(
            connection->sender_work.q_wait,
            !list_empty(&connection->sender_work.q)
            || test_bit(DEVICE_WORK_PENDING, ...)
            || get_t_state(thi) != RUNNING);

        if (get_t_state(thi) != RUNNING)
            break;

        // Drain the work queue
        while (!list_empty(&connection->sender_work.q)) {
            w = list_first_entry(&connection->sender_work.q,
                                  struct drbd_work, list);
            list_del_init(&w->list);
            // release sender_work lock before calling
            if (w->cb(w, 0) == 0)
                continue;
            // cb returned error → disconnect
            break;
        }
    }

    // Flush remaining work with cancel=1
    while (!list_empty(&connection->sender_work.q)) {
        w = list_first_entry(...);
        w->cb(w, 1);  // cancel all pending work
    }
}
```

---

## 6. `drbd_send_dblock()` — Sending a Write to the Secondary

This is the function that actually transmits `P_DATA` over the wire:

```bash
grep -n -A 80 "^int drbd_send_dblock\b" drbd/drbd_sender.c
```

```c
int drbd_send_dblock(struct drbd_peer_device *peer_device,
                     struct drbd_request *req)
{
    struct drbd_connection *connection = peer_device->connection;
    struct p_data p;

    // Build the header
    p.head = ... (filled by drbd_prepare_command())
    p.sector    = cpu_to_be64(req->i.sector);
    p.block_id  = (u64)(uintptr_t)req;    // ← key trick
    p.seq_num   = cpu_to_be32(atomic_inc_return(&peer_device->packet_seq));
    p.dp_flags  = drbd_dp_flags(peer_device, req);
    // dp_flags includes: DP_SEND_WRITE_ACK if protocol C,
    //                    DP_SEND_RECEIVE_ACK if protocol B

    // Send header over DATA socket
    err = drbd_send_command(peer_device, DATA_STREAM, &p, sizeof(p));

    // Send data pages (zero-copy if possible)
    err = drbd_send_bio(peer_device, req->master_bio);
    //     → calls connection->transport->ops->send_page()
    //     → TCP: sock_sendpage() per page (splice / sendfile path)

    return err;
}
```

```bash
grep -n "drbd_dp_flags\|DP_SEND_WRITE_ACK\|drbd_send_bio\b" drbd/drbd_sender.c
```

---

## 7. The Work Queue Entry for a Write Request

When `__drbd_make_request()` finishes setting up a `drbd_request`, it enqueues it for the sender:

```bash
grep -n "drbd_queue_write\|drbd_send_writes\|sender_work" drbd/drbd_req.c drbd/drbd_sender.c | head -20
```

The `drbd_request` itself is the work item — its `drbd_work` member is embedded:

```c
// In drbd_req.h:
struct drbd_request {
    ...
    struct drbd_work w;         // embedded work item
    ...
};
// The work callback:
// w.cb = drbd_send_dblock_work (or similar)
```

When the sender thread dequeues this work item, it calls `drbd_send_dblock()`.

---

## 8. Epochs and Barriers — Write Ordering on the Secondary

### 8.1 Why epochs exist — not the same job as the transfer log

The transfer log and epochs solve **different problems**:

- **Transfer log**: ensures the sender thread sends writes in dagtag sequence (initiator-side ordering). It is a local in-memory structure and says nothing about durability at the peer.
- **Epochs**: answer the question *how does the initiator know writes are durably stored on the peer?*

For **Protocol C/B**, every write gets an individual `P_WRITE_ACK` / `P_RECV_ACK`, so each request knows independently when it is done. Epochs matter less there.

For **Protocol A**, there is **no per-write acknowledgment**. The only durability signal is `P_BARRIER_ACK`. Epochs are the grouping mechanism: when the peer sends `P_BARRIER_ACK`, every write in that epoch is confirmed on stable storage. Without epochs, a `P_BARRIER_ACK` has no defined scope.

`__req_mod()` for `BARRIER_ACKED` (`drbd_req.c:1322`) shows this:
```c
case BARRIER_ACKED:
    /* As this is called for all requests within a matching epoch,
     * we need to filter, and only set RQ_NET_DONE for those that
     * have actually been on the wire. */
    if (req->net_rq_state[idx] & RQ_NET_MASK)
        mod_rq_state(req, m, peer_device, 0, RQ_NET_DONE);
```
`tl_release()` walks all requests in the epoch and fires `BARRIER_ACKED` on each — the **only** completion path for Protocol A writes.

### 8.2 Epoch structure (`drbd_int.h:419`)

```c
struct drbd_epoch {
    struct drbd_connection *connection;
    struct list_head list;          // node in connection->epochs
    unsigned int barrier_nr;        // sequence number
    atomic_t epoch_size;            // writes added to this epoch
    atomic_t active;                // writes not yet completed on peer disk
    atomic_t confirmed;             // adjusted for P_CONFIRM_STABLE
    unsigned long flags;            // DE_BARRIER_IN_NEXT_EPOCH_ISSUED, etc.
};
```

### 8.3 When a new epoch starts

`start_new_tl_epoch()` (`drbd_req.c:388`) increments `resource->current_tle_nr` and wakes
the sender. It is called from two places:

- `drbd_req.c:606` — `REQ_PREFLUSH` / `REQ_FUA` / `blkdev_issue_flush()` arrives; the
  flush becomes an epoch boundary
- `drbd_req.c:1184` — `current_tle_writes >= max_epoch_size` (flow control)

### 8.4 Sender: detecting the boundary (`drbd_sender.c:3444`)

```c
static bool should_send_barrier(struct drbd_connection *connection, unsigned int epoch)
{
    if (!connection->send.seen_any_write_yet)
        return false;
    return connection->send.current_epoch_nr != epoch;
}
static void maybe_send_barrier(struct drbd_connection *connection, unsigned int epoch)
{
    if (should_send_barrier(connection, epoch)) {
        if (connection->send.current_epoch_writes)
            drbd_send_barrier(connection);       // sends P_BARRIER on CONTROL_STREAM
        connection->send.current_epoch_nr = epoch;
    }
}
```
Before sending a data packet, the sender calls `maybe_send_barrier()`. If the request's
epoch number has advanced, `P_BARRIER` is sent first.

### 8.5 Receiver: `receive_Barrier()` (`drbd_receiver.c:1945`)

On the secondary, receiving `P_BARRIER`:
1. Assigns the barrier number to the current epoch object
2. Depending on `write_ordering` mode:
   - `WO_BDEV_FLUSH` / `WO_DRAIN_IO`: drains all active peer requests and issues a disk flush before sending `P_BARRIER_ACK`
   - `WO_BIO_BARRIER` / `WO_NONE`: recycles epoch immediately
3. Allocates a new `drbd_epoch` for incoming writes after this barrier

### 8.6 Flow control via `max_epoch_size`

When `current_tle_writes >= max_epoch_size`, `start_new_tl_epoch()` forces a new epoch
(`drbd_req.c:1184`). This bounds outstanding writes before the peer must flush, limiting
memory pressure on both sides. `max_epoch_size` is configurable per connection.

### 8.7 Summary

| Mechanism | Scope | Solves |
|---|---|---|
| Transfer log | Initiator sender thread | Write send ordering (dagtag sequence) |
| Epochs | Network round-trip | Protocol A bulk durability ack + peer disk flush ordering + flow control |

---

## 9. `P_WRITE_ACK` Flow — ACK Handling on the Primary

When the secondary sends P_WRITE_ACK:

```bash
grep -n -A 40 "^static int got_WriteAck\b" drbd/drbd_receiver.c
```

```c
static int got_WriteAck(struct drbd_connection *connection,
                         struct packet_info *pi)
{
    struct p_block_ack *p = pi->data;
    struct drbd_peer_device *peer_device = ...;
    struct drbd_request *req;
    sector_t sector = be64_to_cpu(p->sector);
    u64 block_id    = p->block_id;   // ← this is the drbd_request pointer!

    // Find the request by block_id
    req = (struct drbd_request *)(uintptr_t)block_id;
    // Validate: req->device == peer_device->device, sector matches

    // Drive the state machine
    spin_lock_irq(&device->resource->req_lock);
    __req_mod(req, WRITE_ACKED_BY_PEER, peer_device, &m);
    spin_unlock_irq(&device->resource->req_lock);

    // If m.bio is set, complete it outside the lock
    if (m.bio)
        bio_endio(m.bio, m.error);
}
```

---

## 10. Hands-On Exercises (3–4 hours)

### Exercise 1 (45 min): Read `drbd_send_dblock()` completely
```bash
grep -n -A 100 "drbd_send_dblock\b" drbd/drbd_sender.c
```
Annotate every line: what is being built, what is being sent, in what order.

### Exercise 2 (40 min): Trace a barrier through the system
1. Find where `REQ_PREFLUSH` or `REQ_FUA` causes a barrier in `drbd_req.c`
2. Find `drbd_send_barrier()` in `drbd_sender.c`
3. Find `receive_Barrier()` in `drbd_receiver.c`
4. Find where `P_BARRIER_ACK` is sent
5. Find `got_BarrierAck()` back on the primary

### Exercise 3 (30 min): Understand the `block_id` pointer trick
```bash
grep -n "block_id\|uintptr_t" drbd/drbd_sender.c drbd/drbd_receiver.c | head -30
```
Why is casting a kernel pointer to u64 safe here? What assumptions does this require? (Hint: look at `p_block_ack` and `got_WriteAck`)

### Exercise 4 (40 min): Read the full `enum drbd_packet`
```bash
grep -n "P_[A-Z_]\+\s*=" drbd/drbd-headers/drbd_protocol.h
```
For every packet type, write: who sends it, who receives it, and what the payload struct is.

### Exercise 5 (45 min): Find every `drbd_send_*` function in `drbd_sender.c`
```bash
grep -n "^int drbd_send_\|^static int drbd_send_" drbd/drbd_sender.c
```
For each one: which packet type does it build, which socket (DATA or META) does it use, when is it called?

---

## Summary

DRBD's wire protocol uses two sockets per connection and a fixed-size header with a magic number and packet type discriminator. `P_DATA` carries the sector address, a `block_id` (the `drbd_request` pointer for O(1) ACK lookup), and the data payload. **Two producers** put packets on the wire per connection: the `drbd_sender()` thread for bulk data (drains `sender_work.q`) and the `ack_sender` workqueue for acknowledgements (drains via `send_acks_work`). Epochs group writes for ordering; `P_BARRIER` / `P_BARRIER_ACK` ensure the secondary commits epoch data before the primary considers the barrier done. `got_WriteAck()` uses `block_id` to find and complete the original request.

**Next:** Day 6 — The receiver thread: socket I/O, packet dispatch, and `receive_Data()`.
