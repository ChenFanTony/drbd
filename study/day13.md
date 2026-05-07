# Day 13 — The LRU Cache: Generic Cache Used by the Activity Log

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd-kernel-compat/lru_cache.c`, `drbd/drbd-kernel-compat/linux/lru_cache.h`, `drbd/drbd_actlog.c`

> **⚠️ File location correction:** Earlier days referenced `drbd/drbd-kernel-compat/lru_cache.c` — this file does NOT exist at that path. The LRU cache implementation is in **`drbd/drbd-kernel-compat/lru_cache.c`** (and the header at `drbd/drbd-kernel-compat/linux/lru_cache.h`). It is a private copy of the upstream kernel's `lib/lru_cache.c` (which DRBD originally contributed). DRBD ships its own copy in the compat dir to avoid depending on whether the running kernel exports it.
> ```bash
> ls drbd/drbd-kernel-compat/lru_cache.c drbd/drbd-kernel-compat/linux/lru_cache.h
> ```

---

## 1. Purpose and Design

`lru_cache.c` is a **generic, intrusive LRU cache** used by DRBD's activity log (Day 8). It is not DRBD-specific — it is a clean, reusable data structure that maps integer keys to fixed-size elements with LRU eviction policy.

Key design decisions:

| Decision | Rationale |
|---|---|
| Fixed number of slots (`nr_elements`) | Allocated at creation; no dynamic growth |
| Intrusive: elements embed `struct lc_element` | No separate node allocation; element IS the cache slot |
| Hash table + LRU list | O(1) lookup + O(1) LRU touch |
| Explicit transaction model | Caller controls when evictions are persisted to disk |
| Single pending change at a time | Simplifies crash recovery (one AL transaction per slot change) |

---

## 2. Data Structures

```bash
grep -n "struct lru_cache {" drbd/drbd-kernel-compat/linux/lru_cache.h
grep -n "struct lc_element {" drbd/drbd-kernel-compat/linux/lru_cache.h
```

### `struct lc_element` — a single cache slot

```c
struct lc_element {
    struct hlist_node colision;   // in lru_cache->lc_slot[] hash table
    struct list_head list;        // in one of: lru / free / to_be_changed
    unsigned int lc_number;       // current key (extent number), or LC_FREE
    unsigned int lc_new_number;   // pending new key (during eviction transaction)
    unsigned int lc_index;        // slot index in [0, nr_elements)
    unsigned int refcnt;          // reference count; element evictable only if 0
};
#define LC_FREE  (~0U)            // sentinel: this slot is unallocated
```

### `struct lru_cache` — the cache descriptor

```c
struct lru_cache {
    // Configuration (set at lc_create time, immutable after)
    unsigned int nr_elements;    // total capacity
    unsigned int element_size;   // sizeof the user-embedded element
    unsigned int  lc_max_pending_changes; // max pending changes before flush needed

    // Current state
    unsigned int used;           // currently allocated (refcnt > 0 or in lru)
    unsigned int hits;           // cache hit counter
    unsigned int misses;         // cache miss counter
    unsigned int starving;       // times a free slot wasn't available
    unsigned int dirty;          // elements needing write-back (to_be_changed list)

    // Data structures
    struct list_head lru;            // LRU list (MRU at head, LRU at tail)
    struct list_head free;           // free (never used) elements
    struct list_head to_be_changed;  // pending evictions needing disk transaction

    struct hlist_head *lc_slot;      // hash table: key → lc_element
                                     // size: next_power_of_2(nr_elements)
    struct lc_element **lc_element;  // flat array: lc_index → lc_element*
                                     // for O(1) slot→element lookup

    // Locking (caller is responsible for external locking)
    unsigned long flags;         // LC_DIRTY, LC_STARVING, LC_LOCKED, LC_CHANGING
};
```

```bash
grep -n "#define LC_DIRTY\|#define LC_STARVING\|#define LC_LOCKED\|#define LC_CHANGING\|#define LC_FREE" \
    drbd/drbd-kernel-compat/linux/lru_cache.h
```

---

## 3. `lc_create()` — Allocating a Cache

```bash
grep -n -A 80 "^struct lru_cache \*lc_create\b" drbd/drbd-kernel-compat/lru_cache.c
```

```c
struct lru_cache *lc_create(const char *name,
                              struct kmem_cache *cache,
                              unsigned int max_pending_changes,
                              unsigned int e_count,    // nr_elements
                              size_t e_size,           // sizeof(user struct)
                              size_t e_off)            // offsetof(user, lc_element)
{
    struct lru_cache *lc = kzalloc(sizeof(*lc), GFP_KERNEL);

    // Allocate hash table (power-of-2 size)
    unsigned int h = roundup_pow_of_two(e_count);
    lc->lc_slot = kzalloc(sizeof(struct hlist_head) * h, GFP_KERNEL);

    // Allocate flat element array
    lc->lc_element = kzalloc(sizeof(struct lc_element *) * e_count, GFP_KERNEL);

    // Allocate all elements from caller's kmem_cache
    for (unsigned int i = 0; i < e_count; i++) {
        // Allocate user-struct (which embeds lc_element)
        void *p = kmem_cache_alloc(cache, GFP_KERNEL);

        // Get the embedded lc_element
        struct lc_element *e = (struct lc_element *)((char *)p + e_off);
        e->lc_index     = i;
        e->lc_number    = LC_FREE;
        e->lc_new_number = LC_FREE;
        e->refcnt       = 0;

        // Start on the free list
        list_add_tail(&e->list, &lc->free);
        lc->lc_element[i] = e;
    }

    lc->nr_elements              = e_count;
    lc->lc_max_pending_changes   = max_pending_changes;
    // ...
    return lc;
}
```

**How drbd_actlog.c calls it:**
```bash
grep -n "lc_create\b" drbd/drbd_actlog.c drbd/drbd_nl.c
```

```c
// drbd_nl.c (during disk attach):
device->act_log = lc_create("act_log",
    drbd_al_ext_cache,           // kmem_cache for struct lc_element (really: al_extent)
    AL_UPDATES_PER_TRANSACTION,  // max pending changes before forced flush
    nc->al_extents,              // nr_elements (from drbd.conf al-extents setting)
    sizeof(struct lc_element),   // element size
    0);                          // offsetof = 0 (lc_element is the first member)
```

```bash
grep -n "AL_UPDATES_PER_TRANSACTION\b" drbd/drbd_actlog.c
grep -n "al_extents\b" drbd/drbd_nl.c drbd/drbd_int.h | head -10
```

---

## 4. `lc_find()` — Pure Lookup, No Side Effects

```bash
grep -n -A 25 "^struct lc_element \*lc_find\b" drbd/drbd-kernel-compat/lru_cache.c
```

```c
struct lc_element *lc_find(struct lru_cache *lc, unsigned int enr)
{
    struct lc_element *e;
    unsigned int h = lc_hash_fn(lc, enr);   // h = enr & (hash_table_size - 1)

    hlist_for_each_entry(e, &lc->lc_slot[h], colision) {
        if (e->lc_number == enr)
            return e;
    }
    return NULL;
}
```

- **No LRU list update** — this is a pure read with no side effects
- Used after `lc_get()` returns to double-check, or to inspect without acquiring
- Called by `drbd_al_complete_io()` to find the element before calling `lc_put()`

---

## 5. `lc_get()` — The Core Operation

```bash
grep -n -A 100 "^struct lc_element \*lc_get\b" drbd/drbd-kernel-compat/lru_cache.c
```

This is the most important function. Full annotated trace:

```
lc_get(lc, enr)
│
├─ CASE 1: Cache HIT (extent already active)
│   ├── e = lc_find(lc, enr)   → hash lookup O(1)
│   ├── e != NULL
│   ├── e->refcnt++
│   ├── list_move(&e->list, &lc->lru)   // move to MRU head
│   ├── lc->hits++
│   └── return e    ← FAST PATH, no disk I/O needed
│
├─ CASE 2: Slot being changed (LC_CHANGING flag set)
│   └── return NULL   // caller must wait and retry
│       // (only one slot change in flight at a time)
│
├─ CASE 3: Cache MISS — need to allocate/evict a slot
│   │
│   ├─ Try to get a FREE slot first (never used)
│   │   └── e = list_first_entry_or_null(&lc->free, ...)
│   │
│   ├─ No free slot: evict LRU slot
│   │   ├── Walk lru list from TAIL (LRU end)
│   │   ├── Find first e with e->refcnt == 0
│   │   ├── If none found: set LC_STARVING, return NULL
│   │   │   // All slots are in use — caller must wait for lc_put()
│   │   └── e = lru tail entry (victim)
│   │
│   ├── Remove e from hash table (old key)
│   │   └── hlist_del_init(&e->colision)
│   │
│   ├── Set pending change
│   │   ├── e->lc_new_number = enr    // target key
│   │   ├── e->lc_number = LC_FREE   // mark as "in transition"
│   │   ├── set LC_CHANGING on lc
│   │   └── list_move(&e->list, &lc->to_be_changed)
│   │
│   ├── lc->misses++
│   └── return e   ← MISS PATH, caller MUST call lc_committed()
│                    after writing the AL transaction to disk
```

```bash
# See the actual implementation:
grep -n -A 120 "^struct lc_element \*lc_get\b" drbd/drbd-kernel-compat/lru_cache.c
```

---

## 6. `lc_committed()` — Finalising a Slot Change

After `lc_get()` returns a miss (new or evicted slot), and after the caller has written the AL transaction to disk, the caller must call:

```bash
grep -n -A 40 "^void lc_committed\b" drbd/drbd-kernel-compat/lru_cache.c
```

```c
void lc_committed(struct lru_cache *lc)
{
    // Move all elements from to_be_changed → active
    list_for_each_entry_safe(e, tmp, &lc->to_be_changed, list) {
        // Finalise the key change
        e->lc_number = e->lc_new_number;

        // Insert into hash table under new key
        unsigned int h = lc_hash_fn(lc, e->lc_number);
        hlist_add_head(&e->colision, &lc->lc_slot[h]);

        // Set refcnt = 1 (caller holds one reference)
        e->refcnt = 1;

        // Move to LRU list (MRU head)
        list_move(&e->list, &lc->lru);
    }

    // Clear the changing flag
    clear_bit(LC_CHANGING, &lc->flags);

    // Decrement dirty count
    lc->dirty--;
}
```

In `drbd_actlog.c`, `lc_committed()` is called after `drbd_al_write_transaction()` successfully completes:

```bash
grep -n "lc_committed\b" drbd/drbd_actlog.c
```

---

## 7. `lc_put()` — Releasing a Reference

```bash
grep -n -A 30 "^void lc_put\b\|^unsigned int lc_put\b" drbd/drbd-kernel-compat/lru_cache.c
```

```c
unsigned int lc_put(struct lru_cache *lc, struct lc_element *e)
{
    BUG_ON(e->lc_number == LC_FREE);  // can't put a free element
    BUG_ON(e->refcnt == 0);           // double-put is a bug

    if (--e->refcnt == 0) {
        // Element is now evictable
        // It stays in hash table and lru list —
        // just no longer held by any in-flight I/O
        // Will be evicted on next cache miss that needs a slot
    }

    return e->refcnt;
}
```

After `lc_put()` drops refcnt to 0, `wake_up(&device->al_wait)` is called to wake
any thread that was waiting for a free slot (LC_STARVING case):

```bash
grep -n "al_wait\b\|LC_STARVING\b" drbd/drbd_actlog.c | head -10
```

---

## 8. `lc_try_lock_for_transaction()` and `lc_unlock()`

The activity log needs to prevent new `lc_get()` calls during the disk write of the AL transaction (to avoid inconsistency between in-memory and on-disk state):

```bash
grep -n "lc_try_lock_for_transaction\b\|lc_unlock\b\|LC_LOCKED\b" drbd/drbd-kernel-compat/lru_cache.c drbd/drbd_actlog.c | head -15
```

```c
// Prevent new lc_get() while writing transaction
bool lc_try_lock_for_transaction(struct lru_cache *lc)
{
    return !test_and_set_bit(LC_LOCKED, &lc->flags);
}

// Allow lc_get() again after transaction written
void lc_unlock(struct lru_cache *lc)
{
    clear_bit(LC_LOCKED, &lc->flags);
    wake_up(&lc->locked_waiters);
}
```

Usage in `drbd_al_write_transaction()`:
```c
// drbd_actlog.c
lc_try_lock_for_transaction(device->act_log);
    // ← build and write the transaction sector here
    drbd_md_sync_page_io(device, ...);  // synchronous disk write
lc_unlock(device->act_log);
lc_committed(device->act_log);         // finalise slot change
```

---

## 9. The Starving Case — When All Slots Are In Use

```bash
grep -n "LC_STARVING\|al_wait\|starving\b" drbd/drbd_actlog.c drbd/drbd-kernel-compat/lru_cache.c | head -20
```

If `lc_get()` returns NULL with `LC_STARVING` set, `drbd_al_begin_io_fastpath/_nonblock/_commit()` must wait:

```c
// drbd_actlog.c — drbd_al_begin_io_nonblock() slow path
wait_event(device->al_wait,
    (e = lc_get(device->act_log, enr)) != NULL
    || !get_ldev_if_state(device, D_FAILED));
// Woken when: lc_put() drops any element's refcnt to 0
```

This is why DRBD can **stall application I/O** if all `al-extents` slots are simultaneously in use. With default `al-extents = 1024`, this means 1024 concurrent writes to distinct 4 MiB extents — very unlikely in practice.

---

## 10. `lc_seq_printf_stats()` and `lc_seq_dump_details()` — Debugging

```bash
grep -n "lc_seq_printf_stats\b\|lc_seq_dump_details\b" drbd/drbd-kernel-compat/lru_cache.c drbd/drbd_debugfs.c | head -10
```

These are used by DebugFS to expose AL statistics:
```bash
# On a running DRBD system:
cat /sys/kernel/debug/drbd/resources/r0/volumes/0/act_log
# Shows: hits, misses, starving, dirty, used counts
# and a hex dump of slot→extent mappings
```

```bash
grep -n -A 30 "^void lc_seq_printf_stats\b" drbd/drbd-kernel-compat/lru_cache.c
```

---

## 11. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Read `lru_cache.c` completely
~650 lines. Every function — write its name, signature, and purpose. Note all functions that modify `lc->flags` and which bit they set/clear.

### Exercise 2 (40 min): Trace the full `lc_get()` miss path
For an activity log with `al-extents = 4` (tiny, for testing), and current state:
```
slot 0: extent 10  refcnt=1  (write in flight)
slot 1: extent 20  refcnt=0  (MRU head of lru list)
slot 2: extent 30  refcnt=0
slot 3: extent 40  refcnt=0  (LRU tail)
```
Trace `lc_get(lc, 99)` step by step:
1. Hash lookup for extent 99 → miss
2. No free slots
3. Walk lru from tail → victim = slot 3 (extent 40, refcnt=0)
4. Remove slot 3 from hash table
5. Set `slot3->lc_new_number = 99`, move to `to_be_changed`
6. Return slot 3

Then trace the caller writing the AL transaction and calling `lc_committed()`.

### Exercise 3 (35 min): Read `drbd_al_begin_io_prepare()` and `drbd_al_begin_io_commit()`
```bash
grep -n "drbd_al_begin_io_prepare\b\|drbd_al_begin_io_commit\b" drbd/drbd_actlog.c
grep -n -A 60 "^void drbd_al_begin_io_prepare\b\|^static.*drbd_al_begin_io_prepare\b" drbd/drbd_actlog.c
```
How do these two functions divide the work? Why is it split into prepare and commit?

### Exercise 4 (35 min): Understand the `LC_LOCKED` / `LC_CHANGING` interaction
```bash
grep -n "LC_LOCKED\|LC_CHANGING\|lc_try_lock\|lc_committed\b" drbd/drbd-kernel-compat/lru_cache.c drbd/drbd_actlog.c | head -20
```
Draw a timeline:
```
Thread A: lc_get() → miss → sets LC_CHANGING
Thread B: lc_get() → sees LC_CHANGING → returns NULL
Thread A: writes AL transaction → lc_committed() → clears LC_CHANGING
Thread B: retries lc_get() → ...
```
What prevents thread B from evicting the same slot A is changing?

### Exercise 5 (40 min): Map `lc_element` fields to the on-disk AL transaction
```bash
grep -n "struct al_transaction_on_disk {" drbd/drbd_actlog.c
grep -n "drbd_al_write_transaction\b" drbd/drbd_actlog.c
grep -n -A 80 "^static void drbd_al_write_transaction\b" drbd/drbd_actlog.c
```
For a single slot change (one `to_be_changed` element):
- Which field of `lc_element` becomes `updates[0].slot`?
- Which field becomes `updates[0].extent`?
- What is the `context[]` array populated from?

---

## Summary

`lru_cache.c` implements a generic LRU cache with O(1) lookup (hash table), O(1) LRU touch (doubly-linked list), and a transactional eviction model. `lc_get()` returns immediately on hit; on miss it identifies a victim (LRU element with `refcnt==0`), marks the pending change, and returns the element in transition. The caller must serialise this pending change to disk before calling `lc_committed()` to finalise. `lc_put()` decrements the refcount; when it drops to zero the element becomes evictable. The `LC_STARVING` case stalls callers until a slot becomes available via `lc_put()`. This clean separation between cache policy (lru_cache.c) and persistence (drbd_actlog.c) makes both components testable independently.

**Next:** Day 14 — Backing device management: disk attach/detach lifecycle, local I/O error handling, `drbd_backing_dev`.
