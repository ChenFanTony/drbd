# Day 6 — The Receiver Thread: Socket I/O, Dispatch & `receive_Data()`

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_receiver.c` (~5000 lines — the largest file in the codebase)

---

## 1. Overview: Two Receiver Contexts

DRBD has **two distinct receive contexts**, both in `drbd_receiver.c`:

| Context | Thread | Socket | Handles |
|---|---|---|---|
| Data receiver | `connection->receiver` thread | DATA socket | P_DATA, P_RS_DATA_REPLY, P_DATA_REPLY |
| Meta receiver | `connection->receiver` thread (same!) | META socket | P_BARRIER, P_WRITE_ACK, P_STATE, P_PING … |

Wait — it is the **same thread** but it multiplexes between the two sockets. The thread uses `drbd_recv_header()` on each socket in turn (or uses `select`/`poll`). The actual multiplexing mechanism:

```bash
grep -n "drbd_recv_header\|sock_recvmsg\|drbd_recv\b\|transport->ops->recv" drbd/drbd_receiver.c | head -20
```

---

## 2. The Receiver Thread Main Loop

```bash
grep -n -A 80 "^int drbd_receiver\b" drbd/drbd_receiver.c
```

```c
int drbd_receiver(struct drbd_thread *thi)
{
    struct drbd_connection *connection = thi->connection;

    // Phase 1: Establish TCP connection (or wait for incoming)
    if (drbd_connect(connection) != 0)
        goto out;

    // Phase 2: Handshake — exchange features, UUIDs
    err = drbd_do_handshake(connection);

    // Phase 3: Main dispatch loop
    while (get_t_state(thi) == RUNNING) {
        struct packet_info pi;

        // Read next header from DATA or META socket
        err = drbd_recv_header_maybe_unplug(connection, &pi);
        if (err)
            break;

        // Dispatch to handler
        err = drbd_dispatch(connection, &pi);
        if (err)
            break;
    }

out:
    conn_disconnect(connection);
    return 0;
}
```

---

## 3. `struct packet_info` — The Dispatch Context

```bash
grep -n "struct packet_info {" drbd/drbd_int.h
```

```c
struct packet_info {
    enum drbd_packet cmd;    // packet type (P_DATA, P_BARRIER, etc.)
    unsigned int size;       // payload length in bytes
    int vnr;                 // volume number (from p_header100.volume)
    void *data;              // pointer to payload buffer (already received)
};
```

---

## 4. `drbd_recv_header()` — Reading Bytes from the Socket

```bash
grep -n -A 40 "^static int drbd_recv_header\b" drbd/drbd_receiver.c
```

```c
static int drbd_recv_header(struct drbd_connection *connection,
                             struct packet_info *pi)
{
    void *buffer = connection->data.rbuf;  // receive buffer, 4 KiB
    int err;

    // Step 1: Read exactly sizeof(p_header80) bytes
    err = drbd_recv_all_warn(connection, DATA_STREAM, buffer, sizeof(p_header80));
    if (err)
        return err;

    // Step 2: Determine header version from magic
    if (header->h80.magic == cpu_to_be32(DRBD_MAGIC)) {
        pi->cmd  = be16_to_cpu(header->h80.command);
        pi->size = be16_to_cpu(header->h80.length);
    } else if (header->h95.magic == cpu_to_be16(DRBD_MAGIC2)) {
        pi->cmd  = be16_to_cpu(header->h95.command);
        pi->size = be32_to_cpu(header->h95.length);
    } else if (...DRBD_MAGIC3...) {
        pi->vnr  = be16_to_cpu(header->h100.volume);
        pi->cmd  = be16_to_cpu(header->h100.command);
        pi->size = be32_to_cpu(header->h100.length);
    } else {
        return -EINVAL;  // unknown magic → disconnect
    }

    return 0;
}
```

`drbd_recv_all_warn()` loops calling `transport->ops->recv()` until exactly N bytes are read. It handles EAGAIN, short reads, and signals.

```bash
grep -n "drbd_recv_all_warn\b\|drbd_recv_all\b" drbd/drbd_receiver.c | head -10
```

---

## 5. The Dispatch Table

```bash
grep -n "drbd_cmd_handler\|struct drbd_handler\|cmd_handler\b" drbd/drbd_receiver.c | head -20
```

DRBD uses two dispatch tables — one for data-socket packets and one for meta-socket ACKs:

```c
// Data channel handlers (called from receiver thread, large payloads)
static struct drbd_cmd_handler drbd_cmd_handler[] = {
    [P_DATA]           = { "Data",          receive_Data         },
    [P_RS_DATA_REPLY]  = { "RSDataReply",   receive_RSDataReply  },
    [P_DATA_REPLY]     = { "DataReply",     receive_DataReply    },
    [P_BARRIER]        = { "Barrier",       receive_Barrier      },
    [P_STATE]          = { "State",         receive_state        },
    [P_UUIDS110]       = { "Uuids110",      receive_uuids110     },
    // ... ~30 more entries
};

// Meta channel ACK handlers (called from got_* functions, small payloads)
static struct drbd_cmd_handler drbd_ack_handler[] = {
    [P_WRITE_ACK]   = { "WriteAck",   got_WriteAck   },
    [P_RECV_ACK]    = { "RecvAck",    got_RecvAck    },
    [P_NEG_ACK]     = { "NegAck",     got_NegAck     },
    [P_BARRIER_ACK] = { "BarrierAck", got_BarrierAck },
    [P_PING_ACK]    = { "PingAck",    got_PingAck    },
    // ...
};
```

The dispatch call:
```c
static int drbd_dispatch(struct drbd_connection *connection,
                          struct packet_info *pi)
{
    handler = get_handler(pi->cmd);   // lookup in dispatch table
    if (!handler)
        return -EINVAL;

    // If payload, receive it first
    if (pi->size > 0)
        drbd_recv_all(connection, handler->data_buffer, pi->size);

    return handler->fn(connection, pi);
}
```

---

## 6. `receive_Data()` — The Most Important Receiver Handler

This function processes an incoming replicated write on the secondary:

```bash
grep -n -A 200 "^static int receive_Data\b" drbd/drbd_receiver.c
```

Annotated call chain:

```
receive_Data(connection, pi)
│
├─ 1. Resolve peer_device from pi->vnr
│   └── peer_device = idr_find(&connection->peer_devices, pi->vnr)
│
├─ 2. Parse p_data header (already in pi->data)
│   ├── sector    = be64_to_cpu(p->sector)
│   ├── block_id  = p->block_id   (will be echoed in ACK)
│   ├── dp_flags  = be32_to_cpu(p->dp_flags)
│   └── size      = pi->size - sizeof(p_data)  [data payload bytes]
│
├─ 3. Allocate drbd_peer_request from mempool
│   └── peer_req = drbd_alloc_peer_req(peer_device, block_id, sector, size, ...)
│           → mempool_alloc(drbd_ee_mempool, GFP_NOIO)
│           → allocate page list for data storage
│
├─ 4. Read data payload from socket into peer_req->pages
│   └── drbd_recv_all_warn(connection, DATA_STREAM,
│                           peer_req->pages, size)
│       // This is a large read — may be up to 1 MiB per packet
│
├─ 5. Data integrity check (if enabled)
│   └── if (connection->integrity_tfm)
│           drbd_csum_bio(connection->integrity_tfm, bio, digest)
│           compare digest with p->dp_flags carrying expected hash
│
├─ 6. Check for overlapping in-flight writes (interval tree)
│   └── drbd_wait_for_any_resource(device, sector, size)
│           → drbd_find_overlap(&device->write_requests, sector, size)
│           → if overlap: wait_event(device->misc_wait, ...)
│               // Must wait: can't write B before A completes
│
├─ 7. Insert into device->write_requests interval tree
│   └── drbd_insert_interval(&device->write_requests, &peer_req->i)
│
├─ 8. Insert into connection->active_ee list
│   └── spin_lock(&connection->active_ee_lock)
│       list_add_tail(&peer_req->w.list, &connection->active_ee)
│       spin_unlock(...)
│
├─ 9. Assign to epoch
│   └── peer_req->epoch = connection->current_epoch
│       atomic_inc(&peer_req->epoch->epoch_size)
│
├─ 10. Maybe send P_RECV_ACK immediately (protocol B)
│    └── if (dp_flags & DP_SEND_RECEIVE_ACK)
│            drbd_send_ack(peer_device, P_RECV_ACK, peer_req)
│
└─ 11. Submit to local disk
     └── drbd_submit_peer_request(device, peer_req, REQ_OP_WRITE, ...)
             → peer_req->w.cb = e_end_block
             → bio = bio_alloc_bioset(...)
             → bio->bi_end_io = drbd_peer_request_endio
             → generic_make_request(bio)
```

---

## 7. `drbd_peer_request_endio()` — Secondary Write Completion

```bash
grep -n -A 60 "^void drbd_peer_request_endio\b" drbd/drbd_receiver.c
```

```c
void drbd_peer_request_endio(struct bio *bio)
{
    struct drbd_peer_request *peer_req = bio->bi_private;
    struct drbd_connection *connection  = peer_req->peer_device->connection;

    if (bio->bi_status)
        set_bit(EE_WAS_ERROR, &peer_req->flags);

    // Move to done_ee list
    spin_lock_irqsave(&connection->active_ee_lock, flags);
    list_move_tail(&peer_req->w.list, &connection->done_ee);
    spin_unlock_irqrestore(...);

    // Wake the ack_sender workqueue to process done_ee
    // (send P_WRITE_ACK or handle error)
    // NOTE: ACKs are sent from a dedicated kthread_worker (connection->ack_sender),
    //       NOT from the regular sender thread. This decouples ACK latency from
    //       large data sends.
    wake_up(&connection->ee_wait);
    queue_work(connection->ack_sender, &connection->send_acks_work);
}
```

Then the **ack_sender workqueue** dispatches `drbd_send_acks_wf()`, which walks `done_ee` and calls `peer_req->w.cb` = `e_end_block()`:

```bash
grep -n -A 50 "^static int e_end_block\b" drbd/drbd_receiver.c
```

```c
static int e_end_block(struct drbd_work *w, int cancel)
{
    struct drbd_peer_request *peer_req = container_of(w, ...);

    if (test_bit(EE_WAS_ERROR, &peer_req->flags)) {
        // Write failed on secondary disk
        drbd_send_ack(peer_device, P_NEG_ACK, peer_req);
        // Mark OOS in bitmap
        drbd_set_out_of_sync(peer_device, peer_req->i.sector, peer_req->i.size);
    } else {
        // Write succeeded
        if (do_send_write_ack)
            drbd_send_ack(peer_device, P_WRITE_ACK, peer_req);

        // Check if this completes an epoch
        drbd_may_finish_epoch(connection, peer_req->epoch, EV_PUT);
    }

    // Free the peer_request
    drbd_free_peer_req(device, peer_req);
    return 0;
}
```

---

## 8. `drbd_send_ack()` — Building and Sending P_WRITE_ACK

```bash
grep -n -A 30 "^int drbd_send_ack\b" drbd/drbd_sender.c
```

```c
int drbd_send_ack(struct drbd_peer_device *peer_device,
                   enum drbd_packet cmd,
                   struct drbd_peer_request *peer_req)
{
    struct p_block_ack p;

    p.sector   = cpu_to_be64(peer_req->i.sector);
    p.block_id = peer_req->block_id;  // echo back the original block_id
    p.blksize  = cpu_to_be32(peer_req->i.size);
    p.seq_num  = cpu_to_be32(atomic_inc_return(&peer_device->packet_seq));

    return drbd_send_command(peer_device, CONTROL_STREAM, cmd, &p, sizeof(p));
    // Uses CONTROL_STREAM = META socket
}
```

---

## 9. Error Handling in the Receiver

If `drbd_recv_all()` returns an error (connection broken):

```bash
grep -n "conn_request_state\|C_DISCONNECTING\|C_NETWORK_FAILURE" drbd/drbd_receiver.c | head -10
```

```c
// drbd_receiver.c — main loop error path
if (err) {
    // Trigger state machine: move to Disconnecting
    change_cstate(connection, C_DISCONNECTING, CS_HARD);
    break;  // exit main loop
}
// After loop: conn_disconnect(connection) runs cleanup
```

`conn_disconnect()` then:
1. Drains `active_ee`, `sync_ee`, `done_ee` lists
2. Calls `drbd_flush_workqueue()` to finish pending work
3. Calls `drbd_finish_peer_reqs()` to complete or fail all in-flight peer requests
4. Marks all in-flight primary requests' OOS bits in the bitmap
5. Schedules reconnect attempt via worker thread

---

## 10. Hands-On Exercises (3–4 hours)

### Exercise 1 (60 min): Read `receive_Data()` completely
```bash
grep -n -A 300 "^static int receive_Data\b" drbd/drbd_receiver.c
```
Annotate every `if/else` branch: what condition is being checked and why.

### Exercise 2 (40 min): Find and read all `receive_*` functions
```bash
grep -n "^static int receive_\b" drbd/drbd_receiver.c
```
For each one: what packet type does it handle, what does it do, does it send a reply?

### Exercise 3 (30 min): Find and read all `got_*` functions
```bash
grep -n "^static int got_\b" drbd/drbd_receiver.c
```
These are meta-socket ACK handlers. For each one: what `__req_mod()` event does it trigger?

### Exercise 4 (40 min): Trace `receive_Barrier()` completely
```bash
grep -n -A 60 "^static int receive_Barrier\b" drbd/drbd_receiver.c
```
Questions:
- When is a new `drbd_epoch` allocated?
- When is `P_BARRIER_ACK` sent?
- What is `drbd_may_finish_epoch()` doing?

### Exercise 5 (40 min): Understand the `active_ee` / `done_ee` / `sync_ee` lifecycle
```bash
grep -n "active_ee\|done_ee\|sync_ee\|read_ee" drbd/drbd_receiver.c drbd/drbd_int.h | head -40
```
Draw the lifecycle of a `drbd_peer_request` as it moves between these lists.

---

## Summary

`drbd_receiver.c` is the heart of the secondary node's processing. The receiver thread reads bytes from the TCP socket using `drbd_recv_header()`, dispatches to type-specific handlers via a table, and hands large data reads to `receive_Data()`. `receive_Data()` allocates a `drbd_peer_request`, reads the data payload into pages, checks for write overlaps via the interval tree, submits the bio to local disk, and moves the request through `active_ee` → `done_ee`. Completion triggers `e_end_block()` which sends `P_WRITE_ACK` back to the primary.

**Next:** Day 7 — Week 1 integration: connection setup, handshake, UUID exchange & disconnect/reconnect flow.
