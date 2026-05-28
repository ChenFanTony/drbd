# Day 12 — Interval Tree: Augmented Red-Black Tree for I/O Overlap Detection

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_interval.c`, `drbd/drbd_interval.h`, `drbd/drbd_req.c`, `drbd/drbd_receiver.c`

---

## 1. The Problem: Why Interval Trees?

In a replicated storage system, two writes can target overlapping sector ranges. If they arrive on the secondary in a different order than on the primary, data consistency breaks. DRBD must:

1. **On the primary:** Detect if a new write overlaps an in-flight write to the same range
2. **On the secondary:** Before processing a received write, check if an earlier write to the same range is still in flight
3. **During resync:** Skip blocks currently being written by the application

A hash table keyed by sector would only find exact matches. A sorted linked list would find overlaps in O(n). DRBD uses an **augmented interval tree** (augmented red-black tree) to find overlaps in **O(log n)**.

---

## 2. The Augmented Red-Black Tree — Theory

A standard red-black tree (Linux `struct rb_root`) stores nodes sorted by a key. An **augmented** red-black tree adds extra information to each subtree — here, the **maximum endpoint** in the subtree — enabling O(log n) overlap queries:

```
Nodes sorted by start sector:
         [40, 80]  subtree_max_end=200
        /          \
   [10, 50]         [100, 200]
   subtree_max=50   subtree_max=200

Query: does anything overlap [30, 70)?
  → check left subtree: max_end=50 > 30  → might overlap
  → [10,50] overlaps? 10 < 70 AND 50 > 30 → YES
```

The `end` augmentation is maintained automatically on insert/delete using the
Linux `RB_DECLARE_CALLBACKS_MAX` macro.

---

## 3. `struct drbd_interval` — The Node Type

```bash
grep -n "struct drbd_interval {" drbd/drbd_interval.h
```

```c
// drbd_interval.h
struct drbd_interval {
    struct rb_node rb;       // red-black tree node (MUST be first)
    sector_t sector;         // start sector of this I/O range
    sector_t end;            // augmentation: max 'end' in subtree (maintained by callbacks)
    unsigned int size;       // size in bytes
    enum drbd_interval_type type;  // what kind of I/O this is (see below)
    unsigned long flags;     // INTERVAL_* flag bits (see below)

    unsigned int partially_in_al_next_enr; // for resuming partial al_begin_io
};
```

### Interval types

```bash
grep -n "enum drbd_interval_type" drbd/drbd_interval.h
```

| Type | Who sets it | Meaning |
|---|---|---|
| `INTERVAL_LOCAL_WRITE` | primary, `drbd_req.c` | application write on this node |
| `INTERVAL_PEER_WRITE` | secondary, `drbd_receiver.c` | write received from a peer primary |
| `INTERVAL_LOCAL_READ` | primary, `drbd_req.c` | application read on this node |
| `INTERVAL_PEER_READ` | secondary, `drbd_receiver.c` | read request from peer |
| `INTERVAL_RESYNC_WRITE` | sync target, `drbd_receiver.c` | resync block being written |
| `INTERVAL_RESYNC_READ` | sync source, `drbd_sender.c` | resync block being read |
| `INTERVAL_OV_READ_SOURCE/TARGET` | verify, `drbd_sender/receiver.c` | online verify read |
| `INTERVAL_PEERS_IN_SYNC_LOCK` | `drbd_receiver.c` | peers-in-sync lock region |

### Flag bits

```bash
grep -n "enum drbd_interval_flags" drbd/drbd_interval.h
```

| Flag | Meaning |
|---|---|
| `INTERVAL_READY_TO_SEND` | resync: this peer request may now be sent |
| `INTERVAL_SENT` | resync read: sending path has released its reference |
| `INTERVAL_RECEIVED` | resync read: reply received; receiving path released its reference |
| `INTERVAL_SUBMIT_CONFLICT_QUEUED` | queued to `submit_conflict` workqueue after blocking |
| `INTERVAL_SUBMITTED` | **submitted to backing device** — the key gate for conflict ordering |
| `INTERVAL_BACKING_COMPLETED` | backing device bio has completed |
| `INTERVAL_COMPLETED` | fully done; kept in tree for remote reply lookup |
| `INTERVAL_CONFLICT` | verify request: has conflicts (verify is cancelled, not blocked) |
| `INTERVAL_CANCELED` | resync request: cancelled while waiting for conflict resolution |

`drbd_interval` is **embedded** in both `drbd_request` and `drbd_peer_request`:

```bash
grep -n "struct drbd_interval i;" drbd/drbd_req.h drbd/drbd_int.h
```

```c
// drbd_req.h:
struct drbd_request {
    struct drbd_interval i;    // i.type = INTERVAL_LOCAL_WRITE / INTERVAL_LOCAL_READ
    ...
};

// drbd_int.h:
struct drbd_peer_request {
    struct drbd_interval i;    // i.type = INTERVAL_PEER_WRITE / INTERVAL_RESYNC_WRITE / ...
    ...
};
```

Recover the parent struct with `container_of()`:
```c
struct drbd_request     *req      = container_of(interval, struct drbd_request, i);
struct drbd_peer_request *peer_req = container_of(interval, struct drbd_peer_request, i);
```

### Inline type predicates

```c
drbd_interval_is_application(i)  // LOCAL_WRITE, PEER_WRITE, LOCAL_READ, PEER_READ
drbd_interval_is_write(i)        // LOCAL_WRITE, PEER_WRITE, RESYNC_WRITE
drbd_interval_is_resync(i)       // RESYNC_WRITE, RESYNC_READ
drbd_interval_is_verify(i)       // OV_READ_SOURCE, OV_READ_TARGET
drbd_interval_is_local(i)        // LOCAL_READ, LOCAL_WRITE
```

---

## 4. The Three Tree Operations

```bash
grep -n "^bool drbd_insert_interval\|^void drbd_remove_interval\|^struct drbd_interval \*drbd_find_overlap" \
    drbd/drbd_interval.c
```

### 4.1 `drbd_insert_interval()` — O(log n)

```bash
grep -n -A 40 "^bool drbd_insert_interval\b" drbd/drbd_interval.c
```

```c
bool drbd_insert_interval(struct rb_root *root, struct drbd_interval *this)
{
    struct rb_node **new = &root->rb_node, *parent = NULL;
    sector_t this_end = this->sector + (this->size >> 9);

    while (*new) {
        struct drbd_interval *here = rb_entry(*new, struct drbd_interval, rb);

        parent = *new;
        // Maintain augmentation: update max end on the way down
        if (here->end < this_end)
            here->end = this_end;

        if (this->sector < here->sector)
            new = &(*new)->rb_left;
        else if (this->sector > here->sector)
            new = &(*new)->rb_right;
        else if (this < here)
            new = &(*new)->rb_left;
        else if (this > here)
            new = &(*new)->rb_right;
        else
            return false;  // already inserted
    }

    this->end = this_end;
    rb_link_node(&this->rb, parent, new);
    rb_insert_augmented(&this->rb, root, &augment_callbacks);
    // augment_callbacks (from RB_DECLARE_CALLBACKS_MAX) recomputes 'end'
    // on any rotations/recolouring during rebalance
    return true;
}
```

### 4.2 `drbd_find_overlap()` — O(log n)

```bash
grep -n -A 50 "^struct drbd_interval \*drbd_find_overlap\b" drbd/drbd_interval.c
```

`drbd_find_overlap()` is the low-level primitive — it returns **any** overlapping
interval without filtering by flags.  Higher-level callers (`drbd_find_conflict`)
add policy on top.

```c
struct drbd_interval *drbd_find_overlap(struct rb_root *root,
                                         sector_t sector,
                                         unsigned int size)
{
    struct rb_node *node = root->rb_node;
    struct drbd_interval *overlap = NULL;
    sector_t end = sector + (size >> 9);

    while (node) {
        struct drbd_interval *here = rb_entry(node, struct drbd_interval, rb);

        if (node->rb_left && sector < interval_end(node->rb_left)) {
            // left subtree's max end > our start → overlap must be on left
            node = node->rb_left;
        } else if (here->sector < end &&
                   sector < here->sector + (here->size >> 9)) {
            // this node overlaps
            overlap = here;
            break;
        } else if (sector >= here->sector) {
            node = node->rb_right;
        } else {
            break;
        }
    }
    return overlap;
}
```

`drbd_next_overlap()` continues from a found node, enabling the macro:

```c
#define drbd_for_each_overlap(i, root, sector, size)   \
    for (i = drbd_find_overlap(root, sector, size);    \
         i;                                             \
         i = drbd_next_overlap(i, sector, size))
```

### 4.3 `drbd_remove_interval()` — O(log n)

```bash
grep -n -A 10 "^void drbd_remove_interval\b" drbd/drbd_interval.c
```

```c
void drbd_remove_interval(struct rb_root *root, struct drbd_interval *this)
{
    if (drbd_interval_empty(this))
        return;
    rb_erase_augmented(&this->rb, root, &augment_callbacks);
    // augment_callbacks recomputes 'end' on affected ancestors
}
```

---

## 5. Augmented Callbacks — Maintaining `end`

```bash
grep -n "RB_DECLARE_CALLBACKS_MAX\|NODE_END\|augment_callbacks" drbd/drbd_interval.c | head -5
```

The actual augmentation is a one-liner using the kernel macro:

```c
// drbd_interval.c
#define NODE_END(node) ((node)->sector + ((node)->size >> 9))
RB_DECLARE_CALLBACKS_MAX(static, augment_callbacks,
    struct drbd_interval, rb, sector_t, end, NODE_END);
```

This macro generates the `propagate`, `copy`, and `rotate` callbacks that keep
each node's `end` field equal to the maximum endpoint in its subtree.  No manual
`rb_max_end` bookkeeping is needed.

---

## 6. Where the Interval Trees Are Used

There are **two interval trees per device**:

```bash
grep -n "rb_root read_requests\|rb_root requests\b" drbd/drbd_int.h
```

```c
// drbd_int.h: struct drbd_device
struct rb_root read_requests;  // local application reads only
struct rb_root requests;       // everything else: local writes, peer writes,
                               // resync writes/reads, verify reads
```

All trees are protected by `device->interval_lock` (spinlock, taken with
`spin_lock_irq()`).

### 6.1 Primary: Application Write — `drbd_conflict_submit_write()`

When `drbd_make_request()` receives a write bio (`drbd_req.c:2207`):

```c
void __drbd_make_request(struct drbd_device *device, struct bio *bio, ...)
{
    req = drbd_request_prepare(device, bio, ...);
    if (rw == WRITE)
        drbd_conflict_submit_write(req);     // drbd_req.c:2107
    else
        drbd_send_and_submit(req);
}
```

Inside `drbd_conflict_submit_write()`:

```c
static void drbd_conflict_submit_write(struct drbd_request *req)
{
    struct drbd_interval *conflict;

    spin_lock_irq(&device->interval_lock);
    clear_bit(INTERVAL_SUBMIT_CONFLICT_QUEUED, &req->i.flags);
    conflict = drbd_find_conflict(device, &req->i, 0);
    if (drbd_interval_empty(&req->i))
        drbd_insert_interval(&device->requests, &req->i);
    if (!conflict)
        set_bit(INTERVAL_SUBMITTED, &req->i.flags);
    spin_unlock_irq(&device->interval_lock);

    if (!conflict)
        drbd_send_and_submit(req);           // proceed immediately
    // else: wait for conflict to clear via drbd_release_conflicts()
}
```

### 6.2 Secondary: Peer Write — `drbd_conflict_submit_peer_write()`

In `receive_Data()` (`drbd_receiver.c:3390`):

```c
spin_lock_irq(&device->interval_lock);
conflict = drbd_find_conflict(device, &peer_req->i, 0);
drbd_insert_interval(&device->requests, &peer_req->i);
if (!conflict)
    set_bit(INTERVAL_SUBMITTED, &peer_req->i.flags);
spin_unlock_irq(&device->interval_lock);

if (!conflict)
    submit_peer_request_activity_log(peer_req);   // proceed
// else: submit deferred via drbd_release_conflicts()
```

### 6.3 Conflict Resolution — Deferred Submission

This is the **key mechanism** — not a blocking wait loop.  When a conflict is
detected, the new request is left in the tree without `INTERVAL_SUBMITTED` set.
It is submitted asynchronously once the blocking interval completes.

**Completion side** — when any request finishes its backing device I/O:

```c
// drbd_req.c:266 (request), drbd_receiver.c:1860 (peer_req)
// Set INTERVAL_BACKING_COMPLETED, then:
drbd_release_conflicts(device, &req->i);  // drbd_req.c:452
```

`drbd_release_conflicts()` scans the tree for all intervals that were waiting on
this region and queues them to `device->submit_conflict.wq`:

```c
void drbd_release_conflicts(struct drbd_device *device,
                             struct drbd_interval *release_interval)
{
    drbd_for_each_overlap(i, &device->requests,
                          release_interval->sector, release_interval->size) {
        if (test_bit(INTERVAL_SUBMITTED, &i->flags))
            continue;  // already submitted, not waiting
        if (test_bit(INTERVAL_SUBMIT_CONFLICT_QUEUED, &i->flags))
            continue;  // already queued
        set_bit(INTERVAL_SUBMIT_CONFLICT_QUEUED, &i->flags);
        // queue to appropriate list (writes / peer_writes / resync_writes / reads)
    }
    queue_work(submit_conflict->wq, &submit_conflict->worker);
}
```

The workqueue runs `drbd_do_submit_conflict()`, which calls the appropriate
`drbd_conflict_submit_*()` function for each deferred request.  That function
re-checks `drbd_find_conflict()` — if the conflict is now gone, it sets
`INTERVAL_SUBMITTED` and proceeds; if a new conflict appeared (unlikely but
possible), it defers again.

**Flow summary:**

```
New write B arrives, overlaps in-flight write A:
  → drbd_conflict_submit_write()
    → drbd_find_conflict() → conflict found (A)
    → insert B into tree WITHOUT INTERVAL_SUBMITTED   ← parked, no AL/send/submit
    → return immediately                              ← no sleep, no wait_event()

Write A's local disk I/O completes (INTERVAL_BACKING_COMPLETED set):
  → drbd_release_conflicts()
    → finds B (INTERVAL_SUBMITTED not set), queues to submit_conflict wq

submit_conflict workqueue fires:
  → drbd_do_submit_conflict()
    → drbd_conflict_submit_write() for B
      → re-check drbd_find_conflict()
      → no conflict now  → set INTERVAL_SUBMITTED → drbd_send_and_submit()  ← B proceeds
      → new conflict     → park again, wait for next drbd_release_conflicts()
```

#### Why `INTERVAL_BACKING_COMPLETED`, not `INTERVAL_COMPLETED`?

The unblock trigger is the conflicting write's **local disk I/O completing** — not
its full completion (which, under Protocol C, requires a network ACK from the
peer). This is intentional:

The ordering requirement is purely about the **local disk** seeing writes in the
correct sequence. Once write A's data is committed to the local backing device,
write B can safely be submitted to that same device — the disk will see them in
order. Whether A's peer has acknowledged is irrelevant to disk ordering.

Waiting for the full Protocol C round-trip would unnecessarily stall B for network
latency on top of disk latency, harming performance with no correctness benefit.

`drbd_find_conflict()` implements this by skipping `INTERVAL_BACKING_COMPLETED`
intervals — they are no longer a block for new submissions even though they may
still be waiting for peer ACKs.

### 6.4 Resync: Skipping Blocks With Application Writes

```bash
grep -n "drbd_find_conflict\b" drbd/drbd_sender.c | head -5
```

In the resync sender, before requesting a block:
```c
// drbd_sender.c
conflict = drbd_find_conflict(device, &peer_req->i, 0);
if (!conflict)
    set_bit(INTERVAL_SUBMITTED, &peer_req->i.flags);
// else: parked; drbd_release_conflicts() will wake it later
```

This ensures resync never overwrites a block that has a concurrent application
write in progress.

---

## 7. `drbd_find_conflict()` — Policy Layer on Top of `drbd_find_overlap()`

```bash
grep -n -A 5 "drbd_find_conflict - find" drbd/drbd_int.h
```

`drbd_find_conflict()` (`drbd_int.h:3022`) wraps `drbd_for_each_overlap()` with
filtering rules:

- **Skip `INTERVAL_COMPLETED`**: fully done requests are kept in the tree for
  remote reply lookup but must not block new I/O.
- **Skip `INTERVAL_BACKING_COMPLETED`** (when applicable): backing device done,
  unblocking can proceed.
- **Defer to resync** (`drbd_should_defer_to_resync()`): application writes wait
  for in-flight resync reads (to prevent overwriting data that the source is
  about to send).
- **`CONFLICT_FLAG_APPLICATION_ONLY`**: only look for application writes (used by
  dual-primary conflict detection — see Section 8).

```c
static inline struct drbd_interval *drbd_find_conflict(
    struct drbd_device *device,
    struct drbd_interval *interval,
    unsigned long flags)
{
    lockdep_assert_held(&device->interval_lock);

    drbd_for_each_overlap(i, &device->requests, sector, size) {
        if (i == interval) continue;
        if (exclusive_until_completed && test_bit(INTERVAL_COMPLETED, &i->flags)) continue;
        if (!exclusive_until_completed && test_bit(INTERVAL_BACKING_COMPLETED, &i->flags)) continue;
        if (!drbd_should_defer_to_interval(interval, i, defer_to_resync)) continue;
        // ... other filters ...
        return i;  // conflict found
    }
    return NULL;
}
```

---

## 8. Dual-Primary Write Conflict Detection

In dual-primary mode (`two_primaries = yes`), both primaries accept application
writes.  Each primary also acts as secondary for the other's writes, so DRBD must
handle the case where both primaries write the **same sector simultaneously**.

### Layer 1: `peer_seq` — ordering writes from the same peer

Each `P_DATA` packet carries a monotonically increasing `peer_seq`. In
`receive_Data()`, when `two_primaries` is enabled:

```c
// drbd_receiver.c:3350
if (tp) {
    err = wait_for_and_update_peer_seq(peer_device, d.peer_seq);
```

`wait_for_and_update_peer_seq()` (`drbd_receiver.c:2903`) blocks until `peer_seq − 1`
has been processed, ensuring writes from the same remote primary are applied in
order even if packets arrive out of order.

### Layer 2: `drbd_peer_write_conflicts()` — hard conflict detection

After `peer_seq` passes, `receive_Data()` calls (`drbd_receiver.c:3383`):

```c
if (tp) {
    err = drbd_peer_write_conflicts(peer_req);
    if (err)
        goto out_del_list;   // disconnect
}
```

`drbd_peer_write_conflicts()` (`drbd_receiver.c:3012`) uses
`drbd_find_conflict(device, &peer_req->i, CONFLICT_FLAG_APPLICATION_ONLY)` to
look for a **local application write** (`INTERVAL_LOCAL_WRITE`) overlapping the
incoming peer write.  If found:

```c
drbd_alert(device, "Concurrent writes detected: "
    "local=%llus +%u, remote=%llus +%u\n", ...);
return -EBUSY;
```

**DRBD disconnects on true dual-primary write overlap.** There is no silent
last-writer-wins resolution.  The `after-sb` / `rr-conflict` policy then governs
reconnection behaviour.

### Layer 3: interval tree deferred submission for non-overlapping regions

If the two primaries write to **different** sectors, the deferred-submission
mechanism (Section 6.3) still serialises any accidental overlap (e.g. a peer
write arrives slightly after a local write to the same region starts):

```
NodeA writes sector 1000        NodeB writes sector 2000
  → both succeed independently

NodeA receives NodeB's write to 2000:
  → drbd_peer_write_conflicts(): no INTERVAL_LOCAL_WRITE at 2000 → ok
  → drbd_find_conflict(): no conflict → INTERVAL_SUBMITTED → submit

NodeB receives NodeA's write to 1000:
  → same: no conflict → submit
```

---

## 9. The Locking Protocol

All interval tree operations must hold `device->interval_lock` (spinlock):

```bash
grep -n "interval_lock\b" drbd/drbd_int.h | head -3
grep -rn "spin_lock.*interval_lock\|spin_unlock.*interval_lock" drbd/drbd_req.c drbd/drbd_receiver.c | head -10
```

`device->interval_lock` serialises:
- `drbd_insert_interval()` / `drbd_remove_interval()`
- `drbd_find_conflict()` / `drbd_find_overlap()`
- Setting `INTERVAL_SUBMITTED`, `INTERVAL_BACKING_COMPLETED`, `INTERVAL_COMPLETED` bits

It is taken as `spin_lock_irq()` because `drbd_request_endio()` (which calls
`drbd_release_conflicts()`) can run from IRQ/softirq context.

---

## 10. INTERVAL_COMPLETED — Why Completed Intervals Stay in the Tree

Application request intervals are **retained** in the tree even after completion
(`INTERVAL_COMPLETED` set) for one reason: the primary needs to look up pending
network replies (P_WRITE_ACK, P_RECV_ACK) that arrive after local completion.
The `block_id` in the ACK packet is a pointer to the `drbd_request`; the tree
keeps the request alive so the ACK handler can locate it.

`drbd_find_conflict()` skips `INTERVAL_COMPLETED` intervals so they do not block
new writes.

---

## 11. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Read `drbd_interval.c` completely
~200 lines. For every function, understand: what are its preconditions, what lock
must be held, what does it return?

### Exercise 2 (40 min): Manually trace `drbd_find_overlap()` on a sample tree
Build a mental tree with these intervals already inserted:
```
[0, 100], [50, 150], [200, 400], [300, 500]
```
Draw the RB tree and `end` augmentation values. Trace `drbd_find_overlap(root, 120, 100)` step by step.

### Exercise 3 (40 min): Find every caller of `drbd_insert_interval()`
```bash
grep -rn "drbd_insert_interval\b" drbd/
```
For each call site:
- What struct is being inserted (`drbd_request` or `drbd_peer_request`)?
- What `type` is set on `i.type`?
- What lock is held at the call?

### Exercise 4 (40 min): Trace the full deferred-submission flow
```bash
grep -n "INTERVAL_SUBMIT_CONFLICT_QUEUED\|drbd_release_conflicts\b\|drbd_do_submit_conflict\b" \
    drbd/drbd_req.c drbd/drbd_receiver.c | head -20
```
Starting from a write request that finds a conflict in `drbd_conflict_submit_write()`:
1. Where is `INTERVAL_SUBMIT_CONFLICT_QUEUED` set?
2. Which event triggers `drbd_release_conflicts()`?
3. What workqueue processes deferred requests?
4. What function finally calls `drbd_send_and_submit()`?

### Exercise 5 (30 min): Understand the `INTERVAL_COMPLETED` lifetime
```bash
grep -rn "INTERVAL_COMPLETED\b" drbd/drbd_req.c drbd/drbd_receiver.c | head -15
```
When is `INTERVAL_COMPLETED` set? When is `drbd_remove_interval()` called? Are
they always in the same place, or can the interval linger completed-but-in-tree?

### Exercise 6 (30 min): Trace dual-primary conflict detection
```bash
grep -n "drbd_peer_write_conflicts\b\|wait_for_and_update_peer_seq\b\|two_primaries\b" \
    drbd/drbd_receiver.c | head -15
```
Given two primaries writing to the same sector at the same time:
1. What detects the conflict on each node?
2. What is the consequence (`goto out_del_list` leads where)?
3. What configuration policy controls reconnection after split-brain?

---

## Summary

`drbd_interval.c` implements a 200-line augmented red-black tree where each node
represents a sector range `[sector, sector+size/512)`.  Each `drbd_interval`
carries a `type` (enum) and a `flags` bitmask; the key flag is `INTERVAL_SUBMITTED`,
which gates whether a request may proceed to the disk.  The `end` field is the
augmentation value — the maximum endpoint in the subtree — maintained via
`RB_DECLARE_CALLBACKS_MAX` so `drbd_find_overlap()` can prune entire subtrees in
O(log n).

Conflict resolution is **non-blocking and deferred**: a new write that finds a
conflicting in-flight write is inserted into `device->requests` without
`INTERVAL_SUBMITTED` set; when the conflicting write's backing I/O completes,
`drbd_release_conflicts()` queues the deferred write to the `submit_conflict`
workqueue, which re-checks and submits.  There is no `wait_event()` in the hot
path.

In dual-primary mode, `wait_for_and_update_peer_seq()` serialises writes from the
same remote primary, and `drbd_peer_write_conflicts()` detects true concurrent
overlapping writes from two primaries — resulting in disconnect rather than silent
data corruption.

**Next:** Day 13 — LRU cache (`lru_cache.c`): generic cache used by the activity log.
