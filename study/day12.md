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
         [40, 80]  rb_max_end=200
        /          \
   [10, 50]         [100, 200]
   rb_max_end=50    rb_max_end=200

Query: does anything overlap [30, 70)?
  → root [40,80] overlaps? 40 < 70 AND 80 > 30  → YES, return [40,80]
  → but first check left: max_end=50 > 30  → left might overlap too
  → [10,50] overlaps? 10 < 70 AND 50 > 30 → YES
```

The `rb_max_end` augmentation is maintained automatically on insert/delete.

---

## 3. `struct drbd_interval` — The Node Type

```bash
grep -n "struct drbd_interval {" drbd/drbd_interval.h
```

```c
struct drbd_interval {
    struct rb_node rb;         // red-black tree node (MUST be first)
    sector_t sector;           // start sector of this I/O range
    unsigned int size;         // size in bytes
    sector_t end;              // sector + (size >> 9)  [in 512-byte sectors]
    sector_t rb_max_end;       // augmentation: max 'end' in this subtree
                               // maintained by augmented RB tree callbacks
    unsigned int local:1;      // 1 = drbd_request (primary-side write)
                               // 0 = drbd_peer_request (secondary-side write)
    unsigned int waiting:1;    // someone is wait_event()ing for this to complete
    unsigned int completed:1;  // this interval has been removed from the tree
};
```

`drbd_interval` is **embedded** in both `drbd_request` and `drbd_peer_request`:

```bash
grep -n "struct drbd_interval i;" drbd/drbd_req.h drbd/drbd_int.h
```

```c
// drbd_req.h:
struct drbd_request {
    struct drbd_interval i;    // ← interval tree node
    // i.sector = starting sector of this write
    // i.size   = byte count
    // i.local  = 1
    ...
};

// drbd_int.h:
struct drbd_peer_request {
    struct drbd_interval i;    // ← interval tree node
    // i.sector = starting sector
    // i.size   = byte count
    // i.local  = 0
    ...
};
```

Using `container_of()` to recover the parent struct:
```c
struct drbd_request *req =
    container_of(interval, struct drbd_request, i);
struct drbd_peer_request *peer_req =
    container_of(interval, struct drbd_peer_request, i);
```

---

## 4. The Three Tree Operations

```bash
grep -n "^void drbd_insert_interval\|^void drbd_remove_interval\|^struct drbd_interval \*drbd_find_overlap" \
    drbd/drbd_interval.c
```

### 4.1 `drbd_insert_interval()` — O(log n)

```bash
grep -n -A 50 "^void drbd_insert_interval\b" drbd/drbd_interval.c
```

```c
void drbd_insert_interval(struct rb_root *root, struct drbd_interval *this)
{
    struct rb_node **new = &root->rb_node, *parent = NULL;

    // Standard BST insert: sort by sector
    while (*new) {
        struct drbd_interval *here =
            rb_entry(*new, struct drbd_interval, rb);

        parent = *new;
        // Update augmentation as we descend
        if (here->rb_max_end < this->end)
            here->rb_max_end = this->end;

        if (this->sector < here->sector)
            new = &(*new)->rb_left;
        else if (this->sector > here->sector)
            new = &(*new)->rb_right;
        else
            // Tie-break by size (multiple requests can start at same sector)
            new = (this->size < here->size) ?
                    &(*new)->rb_left : &(*new)->rb_right;
    }

    // Compute end sector for augmentation
    this->end = this->sector + (this->size >> 9);
    this->rb_max_end = this->end;

    // Insert and rebalance
    rb_link_node(&this->rb, parent, new);
    rb_insert_augmented(&this->rb, root, &drbd_interval_augmented_callbacks);
    // The callbacks maintain rb_max_end on rotations
}
```

### 4.2 `drbd_find_overlap()` — O(log n)

```bash
grep -n -A 60 "^struct drbd_interval \*drbd_find_overlap\b" drbd/drbd_interval.c
```

```c
struct drbd_interval *drbd_find_overlap(struct rb_root *root,
                                         sector_t sector,
                                         unsigned int size)
{
    struct rb_node *node = root->rb_node;
    struct drbd_interval *ret = NULL;
    sector_t end = sector + (size >> 9);

    while (node) {
        struct drbd_interval *here =
            rb_entry(node, struct drbd_interval, rb);

        // Pruning: if subtree's max_end <= our start, no overlap possible
        if (here->rb_max_end <= sector) {
            break;  // entire subtree ends before our range
        }

        // Check this node for overlap
        // Overlap condition: NOT (here->end <= sector OR here->sector >= end)
        if (here->sector < end && here->end > sector) {
            // OVERLAP FOUND
            if (here->completed)
                goto skip;  // already done, not really blocking
            ret = here;
            break;
        }

skip:
        // Descend left or right
        if (node->rb_left &&
            rb_entry(node->rb_left, struct drbd_interval, rb)->rb_max_end > sector)
            node = node->rb_left;
        else
            node = node->rb_right;
    }

    return ret;
}
```

### 4.3 `drbd_remove_interval()` — O(log n)

```bash
grep -n -A 20 "^void drbd_remove_interval\b" drbd/drbd_interval.c
```

```c
void drbd_remove_interval(struct rb_root *root, struct drbd_interval *this)
{
    rb_erase_augmented(&this->rb, root,
                        &drbd_interval_augmented_callbacks);
    // Augmented erase re-computes rb_max_end on affected nodes
}
```

---

## 5. Augmented Callbacks — Maintaining `rb_max_end`

```bash
grep -n "drbd_interval_augmented_callbacks\|drbd_augment_rb_erase\|drbd_compute_max_end" \
    drbd/drbd_interval.c | head -10
```

The Linux kernel's `rb_insert_augmented()` / `rb_erase_augmented()` call user-provided callbacks whenever a rotation or rebalancing touches a node:

```c
static void drbd_augment_propagate(struct rb_node *rb, struct rb_node *stop)
{
    // Walk from rb up to stop, recomputing rb_max_end
    while (rb != stop) {
        struct drbd_interval *node = rb_entry(rb, struct drbd_interval, rb);
        sector_t max_end = node->end;

        if (node->rb.rb_left) {
            struct drbd_interval *left =
                rb_entry(node->rb.rb_left, struct drbd_interval, rb);
            if (left->rb_max_end > max_end)
                max_end = left->rb_max_end;
        }
        if (node->rb.rb_right) {
            struct drbd_interval *right =
                rb_entry(node->rb.rb_right, struct drbd_interval, rb);
            if (right->rb_max_end > max_end)
                max_end = right->rb_max_end;
        }
        if (node->rb_max_end == max_end)
            break;  // unchanged — no need to propagate further
        node->rb_max_end = max_end;
        rb = rb_parent(&node->rb);
    }
}

static const struct rb_augment_callbacks drbd_interval_augmented_callbacks = {
    .propagate = drbd_augment_propagate,
    .copy      = drbd_augment_copy,
    .rotate    = drbd_augment_rotate,
};
```

---

## 6. Where the Interval Trees Are Used

There are **two interval trees per device**:

```bash
grep -n "write_requests\|read_requests" drbd/drbd_int.h | head -5
```

```c
struct drbd_device {
    struct rb_root read_requests;   // pending READ requests from application
    struct rb_root write_requests;  // pending WRITE requests (local + peer)
    ...
};
```

### 6.1 Primary: Inserting Application Writes

```bash
grep -n "drbd_insert_interval\b" drbd/drbd_req.c
```

In `drbd_make_request()`:
```c
// drbd_req.c
req->i.sector = bio->bi_iter.bi_sector;
req->i.size   = bio->bi_iter.bi_size;
req->i.local  = 1;

spin_lock_irq(&device->resource->req_lock);
drbd_insert_interval(&device->write_requests, &req->i);
spin_unlock_irq(&device->resource->req_lock);
```

### 6.2 Secondary: Inserting Peer Writes

```bash
grep -n "drbd_insert_interval\b" drbd/drbd_receiver.c
```

In `receive_Data()`:
```c
// drbd_receiver.c
peer_req->i.sector = sector;
peer_req->i.size   = size;
peer_req->i.local  = 0;

spin_lock_irq(&device->resource->req_lock);
drbd_insert_interval(&device->write_requests, &peer_req->i);
spin_unlock_irq(&device->resource->req_lock);
```

### 6.3 Overlap Check Before Processing

```bash
grep -n "drbd_find_overlap\b\|drbd_wait_for_requests\b\|wait_for_*overlap\|drbd_overlap_requests\b" \
    drbd/drbd_receiver.c drbd/drbd_req.c drbd/drbd_worker.c | head -20
```

In `receive_Data()` on the secondary, before submitting the write:
```c
// Wait for any overlapping writes to complete
for (;;) {
    struct drbd_interval *i;
    spin_lock_irq(&device->resource->req_lock);
    i = drbd_find_overlap(&device->write_requests,
                           peer_req->i.sector, peer_req->i.size);
    if (!i) {
        spin_unlock_irq(...);
        break;  // no overlap, safe to proceed
    }
    // Mark that we're waiting
    i->waiting = 1;
    spin_unlock_irq(...);

    // Sleep until the overlapping write completes
    wait_event(device->misc_wait, i->completed);
}
```

The `misc_wait` wakeup:
```bash
grep -n "misc_wait\|wake_up.*misc_wait" drbd/drbd_req.c drbd/drbd_receiver.c | head -10
```

In `drbd_req_complete()` and `drbd_remove_interval()`, after removing from tree:
```c
if (req->i.waiting)
    wake_up(&device->misc_wait);
```

### 6.4 Resync: Overlap with Application Writes

```bash
grep -n "drbd_find_overlap\b" drbd/drbd_worker.c | head -5
```

In `w_make_resync_request()`:
```c
spin_lock_irq(&device->resource->req_lock);
if (drbd_find_overlap(&device->write_requests, sector, BM_BLOCK_SIZE)) {
    spin_unlock_irq(...);
    // Application write in progress for this block
    // Mark OOS again so resync will retry later
    drbd_bm_set_bits(device, peer_device_idx,
                     BM_SECT_TO_BIT(sector),
                     BM_SECT_TO_BIT(sector) + 1 - 1);
    continue;  // skip this block in this burst
}
spin_unlock_irq(&device->resource->req_lock);
```

---

## 7. The Locking Protocol

All interval tree operations must be done under `device->resource->req_lock` (spinlock):

```bash
grep -n "req_lock\b" drbd/drbd_req.c drbd/drbd_receiver.c | grep "spin_lock\|spin_unlock" | head -20
```

This spinlock serialises:
- Insertion of new requests
- Overlap queries
- Removal on completion
- `rq_state` bit changes (they also access the transfer log)

The spinlock is taken as `spin_lock_irq()` / `spin_unlock_irq()` to disable interrupts, since `drbd_request_endio()` can be called from IRQ/softirq context.

---

## 8. Correctness: Why `completed` Flag is Needed

When `drbd_req_complete()` fires:
1. It sets `req->i.completed = 1`
2. It calls `drbd_remove_interval()` to remove from the tree
3. It calls `wake_up(&device->misc_wait)`

But a waiter in `receive_Data()` might be woken, see the interval still in the tree (because removal hasn't happened yet under that lock hold), and go back to sleep. The `completed` flag prevents `drbd_find_overlap()` from returning a completed-but-not-yet-removed interval as a real overlap:

```c
if (here->completed)
    goto skip;  // ignore this one, it's done
```

---

## 9. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Read `drbd_interval.c` completely
~200 lines. For every function, understand: what are its preconditions, what lock must be held, what does it return?

### Exercise 2 (40 min): Manually trace `drbd_find_overlap()` on a sample tree
Build a mental tree with these intervals already inserted:
```
[0, 100], [50, 150], [200, 400], [300, 500]
```
Draw the RB tree and `rb_max_end` values. Then trace `drbd_find_overlap(root, 120, 100)` step by step.

### Exercise 3 (40 min): Find every caller of `drbd_insert_interval()`
```bash
grep -rn "drbd_insert_interval\b" drbd/
```
For each call site:
- What struct is being inserted?
- What lock is held?
- What are the sector and size values set from?

### Exercise 4 (35 min): Find every caller of `drbd_remove_interval()`
```bash
grep -rn "drbd_remove_interval\b" drbd/
```
For each: when is removal triggered, and is `wake_up()` always called after?

### Exercise 5 (35 min): Understand the `read_requests` tree
```bash
grep -n "read_requests\b" drbd/drbd_req.c drbd/drbd_receiver.c | head -15
```
When are READ requests inserted into `read_requests`? When are they queried? What race do they prevent?

---

## Summary

`drbd_interval.c` implements a 200-line augmented red-black tree where each node represents a sector range `[sector, sector+size/512)`. The `rb_max_end` augmentation enables O(log n) overlap queries: `drbd_find_overlap()` can prune entire subtrees using the max-end field. The tree is used in `device->write_requests` to prevent out-of-order writes on the secondary, and to allow resync to skip blocks with concurrent application writes. All operations are performed under `device->resource->req_lock` which also serialises request state changes and the transfer log.

**Next:** Day 13 — LRU cache (`lru_cache.c`): generic cache used by the activity log.
