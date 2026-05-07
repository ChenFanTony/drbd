# Day 27 — Epoch Management, Flow Control & Handling a Slow Secondary

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_req.c`, `drbd/drbd_receiver.c`, `drbd/drbd_sender.c`, `drbd/drbd_int.h`

---

## 1. What Are Epochs?

An **epoch** is a group of writes between two barrier points. A barrier is generated when:
- Application issues `fsync()` / `fdatasync()` → `REQ_PREFLUSH` or `REQ_FUA` bio flag
- Application calls `ioctl(BLKFLSBUF)`
- `max-epoch-size` limit is reached (forced barrier)

```
Writes:   W1  W2  W3 | W4  W5 | W6  W7  W8
              epoch1  |  epoch2 |    epoch3
                      ↑         ↑
                   barrier    barrier (fsync)
```

The secondary must commit all writes in epoch N to durable storage **before** acknowledging the barrier for epoch N. This is how DRBD guarantees write ordering on the secondary mirrors the primary's intent.

---

## 2. `struct drbd_epoch` — In-Memory Representation

```bash
grep -n "struct drbd_epoch {" drbd/drbd_int.h
# Read every field
```

```c
struct drbd_epoch {
    struct drbd_connection *connection;
    struct list_head list;           // node in connection->epochs list
    unsigned int barrier_nr;         // sequence number (monotonically increasing)
    atomic_t epoch_size;             // number of writes submitted in this epoch
    atomic_t active;                 // writes not yet completed on secondary disk
    unsigned long flags;             // EF_* flag bits
};
```

```bash
grep -n "EF_BARRIER_IN_NEXT_EPOCH_ISSUED\|EF_BARRIER_IN_NEXT_EPOCH_DONE\|EF_EPOCH_DONE\|EF_NET_DONE" \
    drbd/drbd_int.h | head -10
```

| Flag | Meaning |
|---|---|
| `EF_BARRIER_IN_NEXT_EPOCH_ISSUED` | A `P_BARRIER` for this epoch has been sent |
| `EF_BARRIER_IN_NEXT_EPOCH_DONE` | The barrier ACK for this epoch received |
| `EF_EPOCH_DONE` | All writes in this epoch completed on secondary disk |

---

## 3. Epoch Lifecycle on the Primary

### Creating a New Epoch

```bash
grep -n "drbd_may_finish_epoch\b\|drbd_advance_rs_marks\b\|new.*epoch\|alloc.*epoch\|drbd_epoch\b" \
    drbd/drbd_req.c drbd/drbd_sender.c drbd/drbd_receiver.c | head -20
```

A new epoch is created when a barrier is sent:

```c
// drbd_sender.c: drbd_send_barrier()
static int drbd_send_barrier(struct drbd_connection *connection)
{
    struct drbd_epoch *epoch = connection->current_epoch;
    struct p_barrier p;

    // Send P_BARRIER on META (CONTROL) socket
    p.barrier = cpu_to_be32(epoch->barrier_nr);
    p.pad     = 0;
    drbd_send_command(connection, CONTROL_STREAM, P_BARRIER, &p, sizeof(p));

    // Allocate next epoch
    struct drbd_epoch *new_epoch = kzalloc(sizeof(*new_epoch), GFP_NOIO);
    new_epoch->barrier_nr  = epoch->barrier_nr + 1;
    atomic_set(&new_epoch->epoch_size, 0);
    atomic_set(&new_epoch->active, 0);

    spin_lock(&connection->epoch_lock);
    list_add(&new_epoch->list, &epoch->list);
    connection->current_epoch = new_epoch;
    connection->epochs++;
    spin_unlock(&connection->epoch_lock);

    return 0;
}
```

### When Does a Barrier Get Sent?

```bash
grep -n "drbd_send_barrier\b\|REQ_PREFLUSH\|REQ_FUA\|maybe_send_barrier\b" \
    drbd/drbd_req.c drbd/drbd_sender.c | head -20
```

In the sender thread, when processing a write request with flush/FUA flags:

```c
// drbd_sender.c or drbd_req.c:
if (bio_op(req->master_bio) == REQ_OP_WRITE &&
    (req->master_bio->bi_opf & (REQ_PREFLUSH | REQ_FUA))) {
    // This write must be a barrier point
    drbd_send_barrier(connection);
}

// Also triggered by max-epoch-size limit:
if (atomic_read(&connection->current_epoch->epoch_size) >=
    net_conf->max_epoch_size) {
    drbd_send_barrier(connection);
}
```

---

## 4. Epoch Lifecycle on the Secondary

### Receiving `P_BARRIER`

```bash
grep -n -A 60 "^static int receive_Barrier\b" drbd/drbd_receiver.c
```

```c
static int receive_Barrier(struct drbd_connection *connection,
                             struct packet_info *pi)
{
    struct p_barrier *p = pi->data;
    unsigned int barrier_nr = be32_to_cpu(p->barrier);

    // Close the current epoch (no more writes will arrive for it)
    set_bit(EF_BARRIER_IN_NEXT_EPOCH_ISSUED,
            &connection->current_epoch->flags);

    // Transition: current_epoch → a new epoch starts
    // All new writes go into next epoch
    epoch = alloc_next_epoch(connection);
    connection->current_epoch = epoch;
    connection->epochs++;

    // Trigger drbd_may_finish_epoch() to check if old epoch can be closed
    drbd_may_finish_epoch(connection, connection->current_epoch->prev,
                          EV_BARRIER_DONE);
    return 0;
}
```

### `drbd_may_finish_epoch()` — The Epoch Completion Logic

```bash
grep -n -A 80 "^static enum finish_epoch drbd_may_finish_epoch\b\|^enum finish_epoch drbd_may_finish_epoch\b" \
    drbd/drbd_receiver.c
```

```c
enum finish_epoch drbd_may_finish_epoch(
    struct drbd_connection *connection,
    struct drbd_epoch *epoch,
    enum epoch_event ev)
{
    // Called with event: EV_PUT (write completed), EV_BARRIER_DONE, EV_BECAME_LAST

    switch (ev) {
    case EV_PUT:
        atomic_dec(&epoch->active);  // one more write done
        break;
    case EV_BARRIER_DONE:
        set_bit(EF_BARRIER_IN_NEXT_EPOCH_DONE, &epoch->flags);
        break;
    }

    // Can we finish this epoch?
    if (atomic_read(&epoch->active) == 0 &&
        test_bit(EF_BARRIER_IN_NEXT_EPOCH_DONE, &epoch->flags)) {
        // All writes done AND barrier received → commit epoch
        rv = FE_RECYCLED;

        // Send P_BARRIER_ACK to primary
        drbd_send_b_ack(connection, epoch->barrier_nr,
                        atomic_read(&epoch->epoch_size));

        // Recycle or free the epoch struct
        kfree(epoch);
        connection->epochs--;
    }

    return rv;
}
```

---

## 5. `P_BARRIER_ACK` on the Primary — Write Order Guarantee

```bash
grep -n "got_BarrierAck\b\|P_BARRIER_ACK" drbd/drbd_receiver.c | head -10
grep -n -A 40 "^static int got_BarrierAck\b" drbd/drbd_receiver.c
```

```c
static int got_BarrierAck(struct drbd_connection *connection,
                            struct packet_info *pi)
{
    struct p_barrier_ack *p = pi->data;
    unsigned int barrier_nr = be32_to_cpu(p->barrier);
    unsigned int set_size   = be32_to_cpu(p->set_size);

    spin_lock_irq(&resource->req_lock);

    // Walk transfer log: for all requests in epochs <= barrier_nr,
    // clear RQ_EXP_BARR_ACK (if protocol A, this is how ACKs work)
    tl_epoch_barrier_ack(connection, barrier_nr, set_size);

    spin_unlock_irq(&resource->req_lock);
    return 0;
}
```

In Protocol A: requests are ACKed in bulk by barrier ACK (not individually).
In Protocol B/C: requests have individual `P_RECV_ACK` / `P_WRITE_ACK`.

---

## 6. Flow Control — `max-buffers` and Receiver Backpressure

```bash
grep -n "max_buffers\b\|drbd_alloc_peer_req\b\|ee_wait\b\|EE_CALL_AL_BEGIN_IO" \
    drbd/drbd_receiver.c drbd/drbd_int.h | head -20
```

When the secondary's disk is slow, `connection->active_ee` grows without bound — the secondary accepts all incoming writes into RAM even if it can't write them to disk fast enough.

`max-buffers` limits this:

```c
// drbd_receiver.c: receive_Data()
// Before allocating peer_req, check buffer limit:
if (drbd_may_throttle_rx(connection)) {
    // Too many buffers in use!
    // Sleep until some complete:
    wait_event_interruptible(connection->ee_wait,
                              !drbd_may_throttle_rx(connection));
}

peer_req = drbd_alloc_peer_req(peer_device, ...);
```

```bash
grep -n "drbd_may_throttle_rx\b\|may_throttle\b" drbd/drbd_receiver.c | head -10
```

```c
static bool drbd_may_throttle_rx(struct drbd_connection *connection)
{
    unsigned int used_buffers = atomic_read(&connection->pp_in_use);
    return used_buffers >= net_conf->max_buffers;
}
```

When the receiver thread sleeps in `ee_wait`, the TCP receive window fills up → TCP backpressure propagates to the primary's sender → the primary's `drbd_send_dblock()` blocks in `kernel_sendpage()` → primary I/O latency increases.

This is the designed flow control mechanism: **slow secondary → TCP backpressure → slower primary**.

---

## 7. `ap_in_flight` — Application Write Tracking

```bash
grep -n "ap_in_flight\b" drbd/drbd_req.c drbd/drbd_receiver.c drbd/drbd_int.h | head -15
```

```c
// drbd_int.h: struct drbd_connection
atomic_t ap_in_flight;   // application writes pending ACK from this peer
```

- **Incremented** in sender thread after sending `P_DATA` (`rq_state |= RQ_NET_SENT`)
- **Decremented** on `P_WRITE_ACK` (`got_WriteAck`), `P_RECV_ACK` (`got_RecvAck`), or `P_BARRIER_ACK`

`ap_in_flight` is what `drbd_congested()` checks (Day 21). It's the primary's measure of "how far ahead of the secondary am I?"

---

## 8. The `done_ee` → Sender Pipeline

The secondary's sender thread continuously drains `connection->done_ee`:

```bash
grep -n "done_ee\b\|drbd_finish_peer_reqs\b\|drbd_process_done_ee\b" \
    drbd/drbd_receiver.c drbd/drbd_sender.c | head -15
```

```c
// drbd_sender.c or drbd_receiver.c: process_done_ee()
static int drbd_process_done_ee(struct drbd_connection *connection)
{
    LIST_HEAD(work_list);

    spin_lock_irq(&connection->active_ee_lock);
    list_splice_init(&connection->done_ee, &work_list);
    spin_unlock_irq(&connection->active_ee_lock);

    // For each completed peer_req:
    list_for_each_entry_safe(peer_req, tmp, &work_list, w.list) {
        // Call e_end_block() or e_end_resync_block()
        peer_req->w.cb(&peer_req->w, 0);
        // → sends P_WRITE_ACK or P_RS_WRITE_ACK
        // → frees peer_req
        // → decrements pp_in_use → wakes receiver if was throttled
    }
    return 0;
}
```

This creates a pipeline:
```
Receiver reads P_DATA → drbd_alloc_peer_req (pp_in_use++) → submit local write
Local write endio → move to done_ee
Sender drains done_ee → send P_WRITE_ACK → drbd_free_peer_req (pp_in_use--)
                                                         ↑
                                               wakes receiver if throttled
```

---

## 9. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Read `drbd_may_finish_epoch()` completely
```bash
grep -n -A 100 "^static enum finish_epoch drbd_may_finish_epoch\b" drbd/drbd_receiver.c
```
List all possible `epoch_event` values. For each: when is it called, what does it check, what does it do?

### Exercise 2 (40 min): Trace an `fsync()` through DRBD
Application calls `fsync(fd)` on a file on an ext4 filesystem mounted on `/dev/drbd0`:
1. `fsync()` → VFS → ext4 journal commit
2. ext4 submits `bio` with `REQ_PREFLUSH | REQ_FUA`
3. DRBD's `drbd_make_request()` sees the flags
4. `drbd_send_barrier()` is called
5. Secondary receives `P_BARRIER`
6. Secondary sends `P_BARRIER_ACK` after all writes durable
7. Primary's `fsync()` returns to application

Find each step in the code:
```bash
grep -n "REQ_PREFLUSH\|REQ_FUA\|drbd_send_barrier\b\|P_BARRIER\b" drbd/drbd_req.c drbd/drbd_sender.c | head -20
```

### Exercise 3 (40 min): Measure epoch size under fio workload
```bash
# Run a workload with frequent fsyncs:
fio --name=sync-test --rw=write --bs=4k --size=1G \
    --fsync=1 --ioengine=sync --filename=/dev/drbd0

# Watch epoch count:
watch -n1 "grep 'ep:' /proc/drbd"
```
What is the typical epoch size with `fsync=1` (every write fsynced)? With `fsync=1000`? How does this relate to `max-epoch-size`?

### Exercise 4 (35 min): Understand the `epochs` counter
```bash
grep -n "connection->epochs\b\|connection->epochs\s*[+-]=\|connection->epochs\s*[+-][+-]" \
    drbd/drbd_receiver.c drbd/drbd_sender.c | head -15
```
Under what conditions does `connection->epochs` grow unboundedly? What is the safety limit? What happens if it overflows?

### Exercise 5 (35 min): Trace `drbd_finish_peer_reqs()` on disconnect
```bash
grep -n -A 60 "^static void drbd_finish_peer_reqs\b\|^void drbd_finish_peer_reqs\b" \
    drbd/drbd_receiver.c
```
What happens to partially-completed epochs when the connection drops? Are their writes acknowledged or failed?

---

## Summary

Epochs group writes between barrier points. The primary creates new epochs at each barrier (`P_BARRIER` packet sent over META socket). The secondary tracks which writes belong to each epoch via `epoch->active` refcounting and sends `P_BARRIER_ACK` only when all writes in the epoch are durable on local disk. Flow control works via `max-buffers`: when the secondary's in-use buffer count exceeds this limit, the receiver thread sleeps → TCP receive window fills → primary's sender blocks in `kernel_sendpage()` → natural backpressure without any explicit signaling. The `done_ee` → sender pipeline creates efficient pipelining between secondary disk writes and ACK sends.

**Next:** Day 28 — Error handling and failure scenarios: disk failure, network partition, and DRBD's recovery sequences in code.
