# Day 10 — The Resync Engine: Burst Scheduling, SyncSource & SyncTarget

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_worker.c`, `drbd/drbd_receiver.c`, `drbd/drbd_sender.c`

---

## 1. Resync State Machine Entry Points

```bash
grep -n "L_SYNC_SOURCE\|L_SYNC_TARGET\|L_WF_BITMAP_S\|L_WF_BITMAP_T" drbd/drbd_state.c | head -20
```

Resync is triggered from `__after_state_change()` in `drbd_state.c`:

```bash
grep -n "__after_state_change\b\|after_state_change\b" drbd/drbd_state.c | head -10
grep -n "L_SYNC_SOURCE\|drbd_start_resync\|resync_timer" drbd/drbd_state.c | head -10
```

When `repl_state` transitions to `L_SYNC_SOURCE`:
```c
// drbd_state.c: __after_state_change()
if (repl_state[NEW] == L_SYNC_SOURCE) {
    // Initialise resync
    peer_device->rs_total = drbd_bm_total_weight(peer_device);
    peer_device->rs_left  = peer_device->rs_total;
    peer_device->rs_failed = 0;

    // Arm the resync timer → triggers w_make_resync_request burst
    mod_timer(&peer_device->resync_timer, jiffies + SLEEP_TIME);

    drbd_rs_controller_reset(peer_device);
}
```

---

## 2. The Resync Timer and Work Item

The resync burst cycle:
```
resync_timer fires
    → resync_timer_fn()
        → drbd_queue_work(connection->sender_work,
                          &peer_device->resync_work)
            → sender thread wakes
                → calls peer_device->resync_work.cb
                    = w_make_resync_request()
```

```bash
grep -n "resync_timer\b\|resync_timer_fn\b" drbd/drbd_worker.c drbd/drbd_int.h | head -10
grep -n -A 15 "^static void resync_timer_fn\b" drbd/drbd_worker.c
```

```c
static void resync_timer_fn(struct timer_list *t)
{
    struct drbd_peer_device *peer_device =
        from_timer(peer_device, t, resync_timer);

    // Only queue work if we are still SyncSource
    if (peer_device->repl_state[NOW] == L_SYNC_SOURCE ||
        peer_device->repl_state[NOW] == L_PAUSED_SYNC_S) {
        drbd_queue_work(&peer_device->connection->sender_work,
                        &peer_device->resync_work);
    }
}
```

---

## 3. `w_make_resync_request()` — The Core Resync Burst Function

```bash
grep -n -A 200 "^static int w_make_resync_request\b" drbd/drbd_worker.c
```

Full annotated trace:

```
w_make_resync_request(w, cancel)
│
├─ 0. Cancellation check
│   └── if (cancel) return 0;  // thread shutting down
│
├─ 1. State check
│   └── if (repl_state != L_SYNC_SOURCE) return 0;
│       if (test_bit(RESYNC_AFTER_NEG, ...)) ...  // resync paused
│
├─ 2. Compute how many sectors to send this burst
│   └── number = drbd_rs_controller(peer_device, 0)
│       // returns: sectors_to_send_now
│       // (studied in detail in section 4 below)
│
├─ 3. Resync burst loop: send up to `number` sectors
│   └── while (number > 0) {
│       │
│       ├─ a. Find next OOS bit
│       │   └── sector = BM_BIT_TO_SECT(
│       │               drbd_bm_find_next(peer_device, peer_device->bm_resync_fo))
│       │       if (sector == DRBD_END_OF_BITMAP) goto requeue;
│       │       peer_device->bm_resync_fo = BM_SECT_TO_BIT(sector) + 1;
│       │
│       ├─ b. Compute burst size (may be limited by OOS run length)
│       │   └── size = drbd_rs_request_size(peer_device)
│       │           // how many consecutive OOS blocks in this run
│       │           // capped at BM_BLOCK_SIZE * 128 = 512 KiB typically
│       │
│       ├─ c. Check for overlap with concurrent application writes
│       │   └── drbd_overlap_requests(peer_device->device, sector, size)
│       │           → drbd_find_overlap(&device->write_requests, sector, size)
│       │           if overlap: skip this block (application write takes priority)
│       │               drbd_bm_set_bits(sector, size) // re-mark OOS; it'll be cleared by the write
│       │
│       ├─ d. Submit read from local disk
│       │   └── peer_req = drbd_alloc_peer_req(peer_device, ...)
│       │       peer_req->w.cb = w_e_send_resync_data
│       │       drbd_submit_peer_request(device, peer_req, REQ_OP_READ, ...)
│       │           → bio_alloc, bio_add_page for each page
│       │           → bio->bi_end_io = drbd_peer_request_endio
│       │           → submit_bio()
│       │
│       └─ e. Accounting
│           number -= size / 512;
│           atomic_add(size >> 9, &peer_device->rs_in_flight);
│   }
│
├─ 4. Re-arm timer for next burst
│   └── mod_timer(&peer_device->resync_timer, jiffies + SLEEP_TIME)
│
└─ 5. requeue: check if resync is done
    └── if (drbd_bm_total_weight(peer_device) == 0)
            drbd_resync_finished(peer_device, D_UP_TO_DATE)
```

---

## 4. `drbd_rs_controller()` — Throughput Rate Control

```bash
grep -n -A 100 "^static int drbd_rs_controller\b" drbd/drbd_worker.c
```

The rate controller uses a **token bucket / planning algorithm**:

```c
static int drbd_rs_controller(struct drbd_peer_device *peer_device,
                               unsigned int sect_in)
{
    struct peer_device_conf *pdc = peer_device->conf;
    unsigned int want;    // desired sectors for this burst
    int scheduled_for;    // what we're committing to

    // 1. How many sectors per timer tick to meet target rate?
    want = max(SLEEP_TIME * pdc->resync_rate / HZ, 1U);
    // resync_rate in KiB/s, SLEEP_TIME in jiffies (typically HZ/4)

    // 2. How many sectors are currently in-flight (pending ACK)?
    unsigned in_flight = atomic_read(&peer_device->rs_in_flight);

    // 3. If in-flight < c_fill_target: send more
    //    If in-flight > c_fill_target: send less (throttle)
    scheduled_for = want - in_flight + pdc->c_fill_target;

    // 4. Apply per-burst limits
    scheduled_for = clamp(scheduled_for,
                           DRBD_RESYNC_MIN_SECTORS,
                           DRBD_RESYNC_MAX_SECTORS);

    return scheduled_for;  // sectors to send this burst
}
```

Configuration parameters that feed `drbd_rs_controller()`:

```bash
grep -n "resync_rate\|c_plan_ahead\|c_fill_target\|c_delay_target\|c_max_rate" \
    drbd/drbd_int.h drbd/drbd_worker.c | head -20
```

| Config param | Default | Meaning |
|---|---|---|
| `resync-rate` | 250K | Target resync rate in KiB/s |
| `c-fill-target` | 0 | Desired in-flight sectors (0=disabled) |
| `c-delay-target` | 1 | Target round-trip delay in 0.1s units |
| `c-plan-ahead` | 20 | Planning horizon in 0.1s units |
| `c-max-rate` | 250K | Hard cap on resync rate |

---

## 5. SyncSource: After Local Read — `w_e_send_resync_data()`

After the local disk read completes, `drbd_peer_request_endio()` is called, which queues the peer_request back to the sender thread via `peer_req->w.cb = w_e_send_resync_data`:

```bash
grep -n -A 40 "^static int w_e_send_resync_data\b" drbd/drbd_worker.c
```

```c
static int w_e_send_resync_data(struct drbd_work *w, int cancel)
{
    struct drbd_peer_request *peer_req = container_of(w, ...);

    if (cancel || test_bit(EE_WAS_ERROR, &peer_req->flags)) {
        // Read failed on our own disk — we are the SyncSource!
        // → RS data reply error → peer marks this range as failed
        drbd_send_ack(peer_device, P_NEG_RS_DREPLY, peer_req);
        drbd_rs_failed_io(peer_device, peer_req->i.sector, peer_req->i.size);
        goto out;
    }

    // Send P_RS_DATA_REPLY: the actual data
    err = drbd_send_block(peer_device, P_RS_DATA_REPLY, peer_req);

    // Accounting: decrement rs_in_flight
    atomic_sub(peer_req->i.size >> 9, &peer_device->rs_in_flight);

out:
    drbd_free_peer_req(device, peer_req);
    return err;
}
```

```bash
grep -n "drbd_send_block\b" drbd/drbd_sender.c | head -5
grep -n -A 40 "^int drbd_send_block\b" drbd/drbd_sender.c
```

`drbd_send_block()` builds a `P_RS_DATA_REPLY` packet (similar to `P_DATA` but different command byte) and sends it over the DATA socket.

---

## 6. SyncTarget: Receiving Resync Data — `receive_RSDataReply()`

```bash
grep -n -A 100 "^static int receive_RSDataReply\b" drbd/drbd_receiver.c
```

```
receive_RSDataReply(connection, pi)
│
├─ 1. Parse header: sector, block_id, size
├─ 2. Allocate drbd_peer_request
├─ 3. Read data from DATA socket into peer_req->pages
│       drbd_recv_all_warn(connection, DATA_STREAM, pages, size)
│
├─ 4. Check: is this block still OOS?
│       if (drbd_bm_test_bit(peer_device, BM_SECT_TO_BIT(sector)) == 0)
│           // Already in-sync (concurrent write cleared it) — skip write
│           goto skip;
│
├─ 5. Submit write to local disk
│       peer_req->w.cb = e_end_resync_block
│       drbd_submit_peer_request(device, peer_req, REQ_OP_WRITE, ...)
│
└─ 6. Accounting: rs_in_flight++ on target side
```

After local disk write completes, `e_end_resync_block()` is called:

```bash
grep -n -A 60 "^static int e_end_resync_block\b" drbd/drbd_receiver.c drbd/drbd_worker.c
```

```c
static int e_end_resync_block(struct drbd_work *w, int cancel)
{
    struct drbd_peer_request *peer_req = container_of(w, ...);

    if (test_bit(EE_WAS_ERROR, &peer_req->flags)) {
        // Write failed on SyncTarget's disk
        drbd_send_ack(peer_device, P_NEG_RS_DREPLY, peer_req);
        peer_device->rs_failed += peer_req->i.size >> 9;
    } else {
        // Write succeeded
        // Clear OOS bits in bitmap
        drbd_bm_clear_bits(device, peer_device->bitmap_index,
                            BM_SECT_TO_BIT(sector),
                            BM_SECT_TO_BIT(sector + size/512) - 1);

        // Update progress counter
        peer_device->rs_left -= peer_req->i.size >> 9;

        // Periodically write updated bitmap to disk
        if (peer_device->rs_left % (BM_BITS_PER_PAGE * BM_BLOCK_SIZE/512) == 0)
            drbd_bm_write_range(peer_device->device, ...);

        // Send P_RS_WRITE_ACK to SyncSource
        drbd_send_ack(peer_device, P_RS_WRITE_ACK, peer_req);
    }

    drbd_free_peer_req(device, peer_req);
    return 0;
}
```

---

## 7. SyncSource Receives P_RS_WRITE_ACK — `got_RSWriteAck()`

```bash
grep -n -A 40 "^static int got_RSWriteAck\b" drbd/drbd_receiver.c
```

```c
static int got_RSWriteAck(struct drbd_connection *connection,
                           struct packet_info *pi)
{
    struct p_block_ack *p = pi->data;
    sector_t sector  = be64_to_cpu(p->sector);
    int size         = be32_to_cpu(p->blksize);

    // Clear OOS bits on source side too (symmetry)
    drbd_bm_clear_bits(device, peer_device->bitmap_index,
                        BM_SECT_TO_BIT(sector),
                        BM_SECT_TO_BIT(sector + size/512) - 1);

    // Decrement in-flight count
    atomic_sub(size >> 9, &peer_device->rs_in_flight);

    // Update progress
    peer_device->rs_left -= size >> 9;

    // If rs_left == 0 → done
    if (peer_device->rs_left == 0)
        drbd_resync_finished(peer_device, D_UP_TO_DATE);

    return 0;
}
```

---

## 8. Resync Completion: `drbd_resync_finished()`

```bash
grep -n -A 80 "^void drbd_resync_finished\b\|^static void drbd_resync_finished\b" drbd/drbd_worker.c
```

```c
void drbd_resync_finished(struct drbd_peer_device *peer_device,
                           enum drbd_disk_state new_peer_disk_state)
{
    // 1. Final bitmap write (persist all cleared bits)
    drbd_bm_write(device, peer_device);

    // 2. Update metadata (UUIDs, al-bitmap)
    drbd_uuid_set_bm(peer_device, 0);     // clear bitmap UUID
    drbd_md_sync(device);                 // flush metadata

    // 3. Change replication state: SyncSource → Established
    begin_state_change(resource, &irq_flags, CS_VERBOSE);
    __change_repl_state(peer_device, L_ESTABLISHED);
    // 4. Change disk state of peer: Inconsistent → UpToDate
    __change_peer_disk_state(peer_device, new_peer_disk_state);
    end_state_change(resource, &irq_flags, "resync-finished");

    // 5. Log and notify
    drbd_info(peer_device, "Resync done (total %lu K synced, %lu K failed)\n",
              peer_device->rs_total >> 1,
              peer_device->rs_failed >> 1);

    // 6. Run after-resync user hook
    drbd_kobject_uevent(device);
}
```

---

## 9. Resync Pause and Resume

```bash
grep -n "L_PAUSED_SYNC_S\|L_PAUSED_SYNC_T\|drbd_pause_after\|drbd_resume_next" \
    drbd/drbd_worker.c drbd/drbd_state.c | head -20
```

Resync is paused:
- **Manually:** `drbdadm pause-sync r0` → `change_repl_state(L_PAUSED_SYNC_S)`
- **Automatically:** due to `resync-after` dependency (another resource must finish first)
- **On congestion:** if application I/O is very heavy (Ahead/Behind mode)

The resync timer still fires during pause but `w_make_resync_request()` checks the pause flag and returns immediately without sending data.

---

## 10. Resync Interaction with Application Writes

During `L_SYNC_SOURCE`, the primary continues to accept application writes. These writes go both to local disk AND are replicated. The interaction:

```bash
grep -n "drbd_overlap_requests\|drbd_rs_set_out_of_sync\|C_SKIP_UNSYNCED" \
    drbd/drbd_worker.c drbd/drbd_req.c | head -15
```

**Case 1: Application write to an already-resynced block**
- Write is replicated normally
- Bitmap bit stays clear (in-sync)

**Case 2: Application write to an OOS block that resync hasn't reached**
- Write clears the bitmap bit on both nodes
- When resync cursor reaches this block, `drbd_bm_find_next()` skips it (bit already clear)

**Case 3: Application write to a block currently being resynced**
- Detected by `drbd_find_overlap(&device->write_requests, sector, size)` in `w_make_resync_request()`
- Resync skips this block and re-marks it OOS: the application write will clear it
- Note: the re-mark + clear is slightly racy but safe because DRBD treats any ambiguity as OOS

---

## 11. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Read `w_make_resync_request()` completely
```bash
grep -n -A 250 "^static int w_make_resync_request\b" drbd/drbd_worker.c
```
Count: how many OOS sectors can be submitted per timer tick at the default `resync-rate 250K`? How long would a 100 GiB full resync take?

### Exercise 2 (40 min): Trace `drbd_rs_controller()` step by step
```bash
grep -n -A 100 "^static int drbd_rs_controller\b" drbd/drbd_worker.c
```
Manually compute the return value for:
- `rs_in_flight = 0`, `resync_rate = 250K`, `SLEEP_TIME = HZ/4`
- `rs_in_flight = 5000` (congested), same parameters

### Exercise 3 (40 min): Find all callers of `drbd_resync_finished()`
```bash
grep -n "drbd_resync_finished\b" drbd/drbd_worker.c drbd/drbd_receiver.c
```
Under what conditions is it called with `D_UP_TO_DATE` vs `D_INCONSISTENT` vs `D_DISKLESS`?

### Exercise 4 (35 min): Trace `e_end_resync_block()` on write failure
```bash
grep -n "EE_WAS_ERROR\|rs_failed\|P_NEG_RS_DREPLY" drbd/drbd_receiver.c drbd/drbd_worker.c | head -20
```
What does `got_NegRSDReply()` do on the SyncSource side when it receives a write-failure ACK?

### Exercise 5 (35 min): Understand the resync-after dependency chain
```bash
grep -n "resync_after\|drbd_pause_after\|drbd_resume_next" drbd/drbd_worker.c | head -20
grep -n -A 30 "drbd_pause_after\b" drbd/drbd_worker.c
```
If `resource r1` has `resync-after r0`, what prevents r1 from starting resync while r0 is still resyncing?

---

## Summary

The resync engine is timer-driven: `resync_timer_fn()` fires periodically and queues `w_make_resync_request()` on the sender thread. The burst function uses `drbd_bm_find_next()` as a cursor into the OOS bitmap, reads data from local disk, and sends it as `P_RS_DATA_REPLY`. The SyncTarget writes received data, clears bitmap bits, and sends `P_RS_WRITE_ACK`. Rate control is handled by `drbd_rs_controller()` which implements a token-bucket-like algorithm respecting `resync-rate` and the `c-*` congestion parameters. Resync completes when `rs_left == 0`, triggering a final bitmap flush, UUID update, and state transition to `Established / UpToDate`.

**Next:** Day 11 — The worker thread: all work item types, the work queue, deferred resync, metadata sync.
