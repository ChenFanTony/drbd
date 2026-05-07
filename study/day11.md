# Day 11 — The Worker Thread, ack_sender Workqueue & Metadata Sync

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_sender.c` (~3800 lines — contains `drbd_worker()` + sender + many `w_*` callbacks), `drbd/drbd_main.c`, `drbd/drbd_int.h`

> **⚠️ Important file-name correction:** Despite what older docs say, **there is no `drbd_worker.c` file.** The function `drbd_worker()` and all the `w_*` callbacks live in `drbd_sender.c`. This was a 9.x-era consolidation. Verify with:
> ```bash
> grep -n "^int drbd_worker\b" drbd/drbd_sender.c
> ```

---

## 1. Threads & Workqueues Per Connection

```bash
grep -n "struct drbd_thread {" drbd/drbd_int.h
grep -n "drbd_thread_start\b\|connection->ack_sender\b" drbd/drbd_main.c drbd/drbd_nl.c drbd/drbd_int.h | head -10
```

```
For each drbd_connection:
  ┌─────────────────────────────────────────────────────────┐
  │  receiver thread (connection->receiver)                 │
  │    reads TCP socket → dispatches to receive_*() or got_*()│
  ├─────────────────────────────────────────────────────────┤
  │  sender thread (connection->sender)                     │
  │    drains sender_work.q → sends P_DATA, P_BARRIER, etc. │
  │    Defined as drbd_sender() in drbd_sender.c            │
  ├─────────────────────────────────────────────────────────┤
  │  ack_sender workqueue (connection->ack_sender)          │
  │    NEW IN DRBD 9: dedicated kthread_worker for ACK sends│
  │    Decoupled from data sends so big P_DATA writes don't │
  │    block urgent ACKs                                    │
  │    Drains via send_acks_work → drbd_send_acks_wf()      │
  ├─────────────────────────────────────────────────────────┤
  │  drbd_worker() thread (resource-level, not per-conn)    │
  │    Defined as drbd_worker() in drbd_sender.c            │
  │    Drains resource->work.q                              │
  │    Handles state-after-change tasks, AL writes, etc.    │
  └─────────────────────────────────────────────────────────┘
```

> **Critical correction from earlier days:** ACKs (`P_WRITE_ACK`, `P_RECV_ACK`, `P_BARRIER_ACK`, …) are sent from the **`ack_sender` workqueue** (a `kthread_worker`), not from the regular sender thread. This is why `e_end_block()` and friends `queue_work(connection->ack_sender, ...)` instead of using `sender_work.q`.

---

## 2. `struct drbd_work` and `struct drbd_work_queue`

```bash
grep -n "struct drbd_work {" drbd/drbd_int.h
grep -n "struct drbd_work_queue {" drbd/drbd_int.h
```

```c
struct drbd_work {
    struct list_head list;              // node in work queue
    int (*cb)(struct drbd_work *, int); // callback; int=cancel flag
};

struct drbd_work_queue {
    struct list_head q;                 // the actual work list
    spinlock_t q_lock;                  // protects q
    wait_queue_head_t q_wait;           // sender/worker blocks here
};
```

Enqueuing a work item:
```c
static inline void drbd_queue_work(struct drbd_work_queue *q,
                                    struct drbd_work *w)
{
    spin_lock_irq(&q->q_lock);
    list_add_tail(&w->list, &q->q);
    spin_unlock_irq(&q->q_lock);
    wake_up(&q->q_wait);
}
```

```bash
grep -n "drbd_queue_work\b" drbd/drbd_sender.c drbd/drbd_receiver.c drbd/drbd_req.c drbd/drbd_state.c | head -30
```

---

## 3. The Worker/Sender Thread Loop — Detailed

```bash
grep -n "^int drbd_worker\b\|^int drbd_sender\b" drbd/drbd_sender.c
```

The generic work queue draining loop:

```c
// Typical pattern in drbd_sender() or drbd_worker():
while (get_t_state(thi) == RUNNING) {

    // Wait for work or stop signal
    wait_event_interruptible(connection->sender_work.q_wait,
        !list_empty(&connection->sender_work.q)
        || !list_empty(&resource->work.q)
        || get_t_state(thi) != RUNNING
        || ...);

    // Drain resource-level work queue
    while (!list_empty(&resource->work.q)) {
        struct drbd_work *w = list_first_entry(&resource->work.q, ...);
        list_del_init(&w->list);
        w->cb(w, 0);  // call work function
    }

    // Drain connection-level sender work queue
    while (!list_empty(&connection->sender_work.q)) {
        struct drbd_work *w = list_first_entry(&connection->sender_work.q, ...);
        list_del_init(&w->list);
        if (w->cb(w, 0) != 0) {
            // Error in work item — trigger disconnect
            change_cstate(connection, C_NETWORK_FAILURE, CS_HARD);
            break;
        }
    }
}

// Shutdown: drain with cancel=1
while (!list_empty(&connection->sender_work.q)) {
    w = list_first_entry(...);
    list_del_init(&w->list);
    w->cb(w, 1);  // cancel flag
}
```

---

## 4. Complete Inventory of Work Item Callbacks

Find every `w_` prefixed function in `drbd_sender.c`:

```bash
grep -n "^static int w_\|^int w_" drbd/drbd_sender.c
```

For each one below, the full trace:

---

### 4.1 `make_resync_request` (SyncSource burst — covered Day 10)
```bash
grep -n "^static int make_resync_request\b" drbd/drbd_sender.c
```
- **Queued by:** `resync_timer_fn()` (periodic timer)
- **Purpose:** Find next OOS bit, read local disk, enqueue send
- **Re-queues itself** via `mod_timer(&peer_device->resync_timer, ...)`

---

### 4.2 `w_resync_timer` (timer bounce — if separate from above)
```bash
grep -n "w_resync_timer\b" drbd/drbd_sender.c drbd/drbd_int.h | head -5
```
Some versions separate the timer bounce from the actual request work.

---

### 4.3 `w_e_send_resync_data` (SyncSource: send read data)
```bash
grep -n -A 40 "^static int w_e_send_resync_data\b\|w_e_send_resync_data" drbd/drbd_sender.c drbd/drbd_receiver.c
```
- **Queued by:** `drbd_peer_request_endio()` after local read completes
- **Purpose:** Send the just-read block as `P_RS_DATA_REPLY` to SyncTarget
- **On read error:** sends `P_NEG_RS_DREPLY` instead

---

### 4.4 `w_send_bitmap` (bitmap exchange)
```bash
grep -n -A 40 "^static int w_send_bitmap\b" drbd/drbd_sender.c
```
- **Queued by:** `__after_state_change()` when entering `L_WF_BITMAP_S` state
- **Purpose:** Send the local OOS bitmap to the peer (compressed RLE+LZ4)
- **Calls:** `drbd_send_bitmap()` in `drbd_sender.c`

```bash
grep -n "drbd_send_bitmap\b" drbd/drbd_sender.c | head -5
grep -n -A 60 "^int drbd_send_bitmap\b" drbd/drbd_sender.c
```

---

### 4.5 `w_send_out_of_sync` (Ahead→Behind OOS notification)
```bash
grep -n -A 30 "^static int w_send_out_of_sync\b" drbd/drbd_sender.c
```
- **Queued by:** When in `L_AHEAD` mode, application writes that can't be replicated
- **Purpose:** Send `P_OUT_OF_SYNC` to tell the Behind node which blocks to mark OOS
- **Use case:** Congestion management (Ahead/Behind)

---

### 4.6 `w_e_end_data_req` (secondary: send write ACK)
```bash
grep -n -A 40 "^static int w_e_end_data_req\b\|e_end_block\b" drbd/drbd_sender.c drbd/drbd_receiver.c
```
Actually this is `e_end_block()` in most versions:
- **Queued by:** `drbd_peer_request_endio()` after secondary write completes
- **Purpose:** Send `P_WRITE_ACK` (or `P_NEG_ACK` on error) to primary
- **Also:** `drbd_may_finish_epoch()` — close epoch if all writes done

---

### 4.7 `w_e_end_ov_req` (online verify: SyncSource side)
```bash
grep -n -A 40 "^static int w_e_end_ov_req\b" drbd/drbd_sender.c
```
- **Purpose:** Compute hash of local block, send as `P_OV_REPLY` to verifier
- **Part of:** `drbdadm verify` flow

---

### 4.8 `w_e_end_ov_reply` (online verify: SyncTarget side)
```bash
grep -n -A 40 "^static int w_e_end_ov_reply\b" drbd/drbd_sender.c
```
- **Purpose:** Compare received peer hash with local hash; if mismatch, set OOS bit + send `P_OV_RESULT`

---

### 4.9 `w_send_uuids` (UUID advertise)
```bash
grep -n "w_send_uuids\b\|drbd_send_uuids\b" drbd/drbd_sender.c drbd/drbd_sender.c | head -10
```
- **Queued by:** State changes that require re-advertising UUID to peer
- **Purpose:** Build and send `P_UUIDS110` packet

---

### 4.10 `w_md_sync` (deferred metadata flush)
```bash
grep -n -A 20 "^static int w_md_sync\b" drbd/drbd_sender.c
```
- **Queued by:** `drbd_md_sync_if_dirty()` which is called after any metadata change
- **Purpose:** Flush in-memory metadata to disk
- **Calls:** `drbd_md_sync()` which does a synchronous write to the metadata device

```bash
grep -n "drbd_md_sync\b\|drbd_md_sync_if_dirty\b" drbd/drbd_main.c drbd/drbd_sender.c | head -20
```

---

### 4.11 `w_restart_disk_io` (retry failed disk I/O)
```bash
grep -n "w_restart_disk_io\b" drbd/drbd_sender.c drbd/drbd_req.c | head -5
```
- **Purpose:** After a temporary disk error, retry queued I/O requests

---

### 4.12 `w_start_resync` (initiate resync after state setup)
```bash
grep -n "w_start_resync\b\|drbd_start_resync\b" drbd/drbd_sender.c drbd/drbd_state.c | head -10
grep -n -A 40 "^static int w_start_resync\b\|^void drbd_start_resync\b" drbd/drbd_sender.c
```
- **Queued by:** `__after_state_change()` when entering `L_SYNC_SOURCE`
- **Purpose:** Final setup for resync: initialise counters, arm timer

---

## 5. Metadata Sync in Depth: `drbd_md_sync()`

Metadata is written lazily. Any metadata change (UUID, AL, bitmap, generation counter) sets a dirty flag. A work item (`w_md_sync`) periodically flushes:

```bash
grep -n "drbd_md_sync\b" drbd/drbd_main.c
grep -n -A 60 "^void drbd_md_sync\b" drbd/drbd_main.c
```

```c
void drbd_md_sync(struct drbd_device *device)
{
    struct drbd_md *md = &device->ldev->md;
    struct meta_data_on_disk *buffer;

    // Serialize concurrent metadata writes
    mutex_lock(&device->ldev->md_mutex);

    // Build on-disk representation
    buffer = drbd_md_get_buffer(device, __func__);

    buffer->la_size_sect = cpu_to_be64(drbd_get_capacity(device->this_bdev));
    buffer->uuid[UI_CURRENT]  = cpu_to_be64(md->uuid[UI_CURRENT]);
    buffer->uuid[UI_BITMAP]   = cpu_to_be64(md->uuid[UI_BITMAP]);
    // ... all UUID slots, flags, al_stripes, al_stripe_size_4k

    // Compute CRC32C of the metadata page
    buffer->magic   = cpu_to_be32(DRBD_MD_MAGIC_09);
    buffer->md_size_sect = cpu_to_be32(md->md_size_sect);
    drbd_md_set_sector_offsets(device, device->ldev);
    buffer->crc32c_checksum = cpu_to_be32(
        crc32c(0, buffer, sizeof(*buffer) - sizeof(u32)));

    // Write the metadata page (synchronous)
    drbd_md_sync_page_io(device, device->ldev,
                          device->ldev->md.md_offset,
                          WRITE);

    drbd_md_put_buffer(device);
    mutex_unlock(&device->ldev->md_mutex);
}
```

---

## 6. The `drbd_md_mark_dirty()` Pattern

Every code path that modifies in-memory metadata should call:

```bash
grep -n "drbd_md_mark_dirty\b" drbd/drbd_main.c drbd/drbd_sender.c drbd/drbd_receiver.c drbd/drbd_nl.c | head -20
```

```c
static inline void drbd_md_mark_dirty(struct drbd_device *device)
{
    if (!test_and_set_bit(MD_DIRTY, &device->flags))
        mod_timer(&device->md_sync_timer, jiffies + HZ);
        // Defer: flush in 1 second (coalesce multiple dirty marks)
}
```

This timer-based coalescing avoids writing metadata to disk on every UUID change. Instead, the flush happens at most once per second.

```bash
grep -n "md_sync_timer\b\|MD_DIRTY\b" drbd/drbd_main.c drbd/drbd_int.h | head -15
```

---

## 7. `drbd_md_sync_page_io()` — The Actual Metadata Disk I/O

```bash
grep -n -A 60 "^int drbd_md_sync_page_io\b" drbd/drbd_main.c
```

```c
int drbd_md_sync_page_io(struct drbd_device *device,
                          struct drbd_backing_dev *bdev,
                          sector_t sector, int rw)
{
    struct bio *bio;

    bio = bio_alloc_bioset(GFP_NOIO, 1, &drbd_md_io_bio_set);
    bio->bi_bdev   = bdev->md_bdev;
    bio->bi_iter.bi_sector = sector;
    bio->bi_end_io  = drbd_md_io_complete;
    bio->bi_private = &device->md_io;

    bio_add_page(bio, device->md_io.page, PAGE_SIZE, 0);
    bio->bi_opf = rw ? REQ_OP_WRITE | REQ_SYNC : REQ_OP_READ;

    atomic_set(&device->md_io.in_use, 1);
    submit_bio(bio);

    // Wait for completion
    wait_event(device->misc_wait,
               !atomic_read(&device->md_io.in_use));

    return device->md_io.error;
}
```

Note: `REQ_SYNC` (not `REQ_FUA` here) — for metadata pages that don't need to be ordered relative to data. For the AL transaction which DOES need ordering, `REQ_PREFLUSH | REQ_FUA` is used.

---

## 8. Hands-On Exercises (3–4 hours)

### Exercise 1 (60 min): Inventory all `w_` functions in `drbd_sender.c`
```bash
grep -n "^static int w_\|^int w_" drbd/drbd_sender.c
```
For each function:
1. What is its purpose?
2. What queues it?
3. What does it do on `cancel=1`?

### Exercise 2 (40 min): Trace the full metadata sync path for a UUID change
UUID is updated (e.g., after primary promotion):
```bash
grep -n "drbd_uuid_set\b\|drbd_uuid_new_current\b" drbd/drbd_main.c drbd/drbd_receiver.c | head -10
```
Starting from `drbd_uuid_new_current()`:
1. What UUID field changes?
2. Is `drbd_md_mark_dirty()` called?
3. When does the metadata actually hit the disk?

### Exercise 3 (40 min): Read `w_send_bitmap()` and `drbd_send_bitmap()`
```bash
grep -n -A 80 "^int drbd_send_bitmap\b" drbd/drbd_sender.c
```
How is the bitmap compressed before sending? What packet type is used? How does the receiver know the bitmap is fully received?

### Exercise 4 (30 min): Find the connection work queue vs resource work queue split
```bash
grep -n "resource->work\b\|connection->sender_work\b" drbd/drbd_sender.c drbd/drbd_state.c | head -20
```
Which work items go on the resource-level queue and which on the connection-level queue? Why the distinction?

### Exercise 5 (40 min): Trace `w_e_end_ov_reply()` for online verify
```bash
grep -n -A 60 "^static int w_e_end_ov_reply\b" drbd/drbd_sender.c
# Also:
grep -n "P_OV_REQUEST\|P_OV_REPLY\|P_OV_RESULT\|ov_left\|ov_out_of_sync" \
    drbd/drbd_receiver.c drbd/drbd_sender.c | head -20
```
Draw the full verify flow: which packets go in which direction, what happens on mismatch?

---

## Summary

The sender/worker thread in DRBD processes a unified work queue of `struct drbd_work` items. Each item has a callback function following the `w_*(w, cancel)` convention. Key items include resync burst (`make_resync_request`), bitmap send (`w_send_bitmap`), write ACK send (`e_end_block`), and metadata flush (`w_md_sync`). Metadata changes are coalesced via `drbd_md_mark_dirty()` / `md_sync_timer` and flushed synchronously by `drbd_md_sync_page_io()`. The cancel mechanism ensures the thread can drain cleanly on shutdown.

**Next:** Day 12 — The interval tree: augmented red-black tree for overlapping I/O detection (`drbd_interval.c`).
