# Day 8 — Activity Log: On-Disk Format, Transactions & Crash Recovery

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_actlog.c`, `drbd/drbd-kernel-compat/lru_cache.c`, `drbd/drbd_int.h`

---

## 1. The Problem the Activity Log Solves

A write to DRBD involves two steps that are not atomic:
1. Write data to the local disk
2. Replicate to the secondary

If the primary crashes between steps 1 and 2 (or in the middle of step 1), after reboot we don't know which blocks are inconsistent. Scanning the entire device to find differences could take hours on a large disk.

The **Activity Log (AL)** is a small, fixed-size structure that tracks, at 4 MiB granularity, which **extents** (regions of the device) were being written at crash time. On recovery, only those extents need re-synchronising.

**Guarantee:** The AL on disk always reflects a *superset* of active writes. Any write that starts, the AL is updated *before* the write hits the disk.

---

## 2. Extent Size and AL Slots

```bash
grep -n "AL_EXTENT_SHIFT\|AL_EXTENT_SIZE\|AL_UPDATES_PER_TRANSACTION\|DRBD_AL_EXTENTS_DEF\|DRBD_AL_EXTENTS_MAX" \
    drbd/drbd_int.h drbd/drbd_actlog.c
```

```
AL_EXTENT_SHIFT = 22          (bits)
AL_EXTENT_SIZE  = 1 << 22     = 4 MiB per extent
sectors_per_extent = 4MiB / 512 = 8192 sectors

Default al-extents = 1024     → AL covers up to 4 GiB of active writes
Maximum al-extents = 65536    → AL covers up to 256 GiB of active writes
```

The **extent number** for a given sector:
```c
#define BM_SECT_TO_EXT(sect)  ((sect) >> (AL_EXTENT_SHIFT - 9))
// i.e., sector / 8192
```

The AL maps **slot number → extent number**. There are `al-extents` slots (default 1024). When all slots are occupied and a new extent is needed, one slot is evicted (LRU policy).

---

## 3. On-Disk Structure

The AL is stored in the metadata area as a **circular transaction log**. Each transaction is exactly 512 bytes (one sector), aligned on sector boundaries:

```bash
grep -n "struct al_transaction_on_disk {" drbd/drbd_actlog.c
```

```c
struct al_transaction_on_disk {
    be32 magic;            // AL_MAGIC = 0x23DDE8 or similar
    be32 tr_number;        // monotonically increasing transaction number
    be32 crc32c;           // checksum of entire transaction (for validation)
    be16 n_context;        // number of "context" (slot→extent) entries
    be16 n_updates;        // number of "update" (slot change) entries
    // context entries: up to AL_CONTEXT_SIZE
    struct {
        be16 slot;
        be32 extent;
    } context[AL_CONTEXT_HIGHWATER];
    // update entries
    struct {
        be16 slot;
        be32 extent;
    } updates[AL_UPDATES_PER_TRANSACTION];
    be32 context_start_slot;
    be32 context_size;
    u8   pad[...];         // zero padding to 512 bytes
} __packed;
```

**Context entries:** A snapshot of the current AL state (subset of slots, enough to reconstruct). Written on every transaction to allow recovery without reading old transactions.

**Update entries:** The actual changes in this transaction (which slot is now mapped to which extent).

```bash
grep -n "AL_CONTEXT_SIZE\|AL_CONTEXT_HIGHWATER\|AL_UPDATES_PER_TRANSACTION\|AL_MAGIC" drbd/drbd_actlog.c
```

---

## 4. The In-Memory AL: `struct lru_cache`

The in-memory activity log is managed by `lru_cache.c`:

```bash
grep -n "struct lru_cache {" drbd/drbd-kernel-compat/lru_cache.c drbd/drbd-kernel-compat/linux/lru_cache.h
```

```c
struct lru_cache {
    unsigned int nr_elements;    // total slots = al-extents config value
    unsigned int used;           // currently allocated slots
    unsigned int hits;           // cache hit counter (extent already active)
    unsigned int misses;         // cache miss counter (new extent needed)
    unsigned int starving;       // times we had to wait for a free slot
    unsigned int dirty;          // slots needing a write-back
    unsigned long *lc_slot_in_use; // bitmap: which slots are occupied
    struct hlist_head *lc_slot;  // hash table: extent_nr → lc_element
    struct list_head lru;        // LRU list of elements
    struct list_head free;       // free (unallocated) elements
    struct list_head to_be_changed; // pending slot changes (need disk write)
    struct lc_element *changing_element; // element currently being changed
    unsigned int flags;          // LC_DIRTY, LC_STARVING, LC_LOCKED
};
```

Each slot is a `struct lc_element`:

```bash
grep -n "struct lc_element {" drbd/drbd-kernel-compat/linux/lru_cache.h
```

```c
struct lc_element {
    struct hlist_node colision;   // in lru_cache->lc_slot hash
    struct list_head list;        // in lru or to_be_changed or free list
    unsigned int lc_number;       // the extent number (key)
    unsigned int refcnt;          // how many writes are using this slot
    unsigned int lc_index;        // slot index [0, nr_elements)
    unsigned int lc_new_number;   // pending new extent number (during eviction)
};
```

### 4.1 Why LRU? — design rationale (`lru_cache.h:23–118`)

Each slot maps one 4MB extent. There are `al-extents` slots (default 1024). When all slots
are occupied and a new extent is needed, one slot must be **evicted** — which requires a
synchronous `REQ_FUA` disk transaction. LRU is the right eviction policy because:

- **Locality of reference**: most workloads write repeatedly to a hot set of regions. LRU
  keeps those regions warm so subsequent writes to the same extent are cache hits — no disk
  transaction needed.
- **Cache hit = fast path**: `lc_get()` finds the extent in the hash table, increments
  `refcnt`, returns immediately. Zero disk I/O.
- **Cache miss = slow path**: extent is new; evict the LRU slot; write a disk transaction
  recording the old→new slot mapping; proceed. The disk transaction is the bottleneck.

### 4.2 Three internal lists

Each `lc_element` lives on exactly one list at all times:

| List | Condition | Meaning |
|---|---|---|
| `in_use` | `refcnt > 0` | Extent has active writes — **cannot be evicted** |
| `lru` | `refcnt == 0`, `lc_number != LC_FREE` | Extent cooled down — evictable, ordered by recency |
| `free` | `lc_number == LC_FREE` | Slot never used — cheapest to allocate, no transaction needed |

A hash table (`lc_slot`) maps extent number → `lc_element` for O(1) lookup.

### 4.3 Lifecycle for a single write

```
Write to sector S:
  extent_nr = S >> (AL_EXTENT_SHIFT - 9)   // S / 8192

  lc_get(extent_nr):
    CACHE HIT  → refcnt++, return element   ← no disk I/O (fast path)
    CACHE MISS → pick slot from free list, or evict LRU element
               → write disk transaction (old extent evicted, new extent recorded)
               → return element

  req->local_rq_state |= RQ_IN_ACT_LOG
  Submit write to local disk + replicate to peer

  Write completes (drbd_al_complete_io):
    lc_put(element) → refcnt--
    if refcnt == 0: move to lru list  ← extent "cools down", slot now evictable
```

### 4.4 Crash recovery

On crash, every slot in both `in_use` and `lru` lists is still recorded in the on-disk AL
transaction ring. `drbd_al_apply_to_bm()` reads all those slots on reconnect and sets the
corresponding bits in the OOS bitmap. Only those extents are resynced — not the full
device. The hard limit of `al-extents` slots directly bounds the worst-case resync scope:
`1024 slots × 4MB = 4GB` maximum crash-recovery resync with default settings.

---

## 5. AL Reservation API — `drbd_al_begin_io_*()` Family

In DRBD 9.2, AL reservation is split across **three functions** (not one), to support both fast-path and async paths:

```bash
grep -n "^bool drbd_al_begin_io_fastpath\|^int drbd_al_begin_io_nonblock\|^void drbd_al_begin_io_commit" drbd/drbd_actlog.c
```

```c
// Fast path — try to get the slot without sleeping or doing disk I/O
bool drbd_al_begin_io_fastpath(struct drbd_device *device,
                                struct drbd_interval *i);
// Returns: true  → got the slot, can proceed immediately
//          false → must use nonblock + commit path

// Slow path part 1 — try to get the slot, may need disk transaction
int drbd_al_begin_io_nonblock(struct drbd_device *device,
                               struct drbd_interval *i);
// Returns: 0      → got it
//          -EBUSY → must wait for transaction to be written

// Slow path part 2 — commit the AL transaction to disk
void drbd_al_begin_io_commit(struct drbd_device *device);
// Called after disk transaction completes (or directly if no I/O needed)

// Always paired with:
bool drbd_al_complete_io(struct drbd_device *device,
                          struct drbd_interval *i);
// Returns: true if extent was active (refcnt > 0), false if not
```

This is called from `__drbd_make_request()` before submitting any write.

Full annotated trace of the typical path:

```
__drbd_make_request() needs to write to extent E
│
├─ try drbd_al_begin_io_fastpath(device, &req->i)
│   │
│   ├─ HIT: extent E already in cache → fast path returns true
│   │   → req->local_rq_state |= RQ_IN_ACT_LOG
│   │   → proceed with local write submission
│   │
│   └─ MISS: extent E not in cache → fast path returns false
│       → fall through to nonblock path
│
├─ drbd_al_begin_io_nonblock(device, &req->i)
│   → tries to evict an LRU slot for E
│   → if a transaction needs to be written: returns -EBUSY
│   → request is parked on device->pending_completion list
│
└─ Later: drbd_al_write_transaction() finishes
   → drbd_al_begin_io_commit(device) is called
   → parked requests are released and proceed
```

```bash
grep -n "drbd_al_write_transaction\b" drbd/drbd_actlog.c
grep -n "lc_get\b\|lc_try_get\b\|lc_get_cumulative\b" drbd/drbd_actlog.c drbd/drbd-kernel-compat/lru_cache.c
```

---

## 6. `drbd_al_complete_io()` — Releasing an AL Slot

Called after the write to local disk AND the network ACK have both completed:

```bash
grep -n -A 30 "^bool drbd_al_complete_io\b" drbd/drbd_actlog.c
```

```c
bool drbd_al_complete_io(struct drbd_device *device, struct drbd_interval *i)
{
    unsigned enr = BM_SECT_TO_EXT(i->sector);
    struct lc_element *extent;
    unsigned long flags;
    bool wake = false;

    spin_lock_irqsave(&device->al_lock, flags);
    extent = lc_find(device->act_log, enr);
    if (extent) {
        if (lc_put(device->act_log, extent) == 0)
            wake = true;
        // lc_put() decrements refcnt
        // returns 0 (zero) when refcnt actually drops to 0
    }
    spin_unlock_irqrestore(&device->al_lock, flags);

    if (wake)
        wake_up(&device->al_wait);  // wake anyone waiting for a free slot

    return extent != NULL;   // true if extent was actually active
}
```

> Note the **bool return**: callers can detect "this extent wasn't actually in the AL" — useful for debugging double-completes or BUG detection.

```bash
grep -n "lc_put\b\|lc_find\b" drbd/drbd-kernel-compat/lru_cache.c drbd/drbd_actlog.c
```

---

## 7. Crash Recovery: `drbd_al_apply_to_bm()`

Called during disk attach (`drbd_adm_attach()` in `drbd_nl.c`):

```bash
grep -n "drbd_al_apply_to_bm\b" drbd/drbd_actlog.c drbd/drbd_nl.c
grep -n -A 80 "^void drbd_al_apply_to_bm\b" drbd/drbd_actlog.c
```

```c
void drbd_al_apply_to_bm(struct drbd_device *device)
{
    unsigned int enr;

    // Walk all occupied AL slots
    lc_for_each_slot_in_use(device->act_log, enr) {
        // This extent was being written when we crashed
        // Mark all its bits OOS in the bitmap for all peers
        unsigned long start = (unsigned long)enr * AL_EXTENT_SIZE / (BM_BLOCK_SIZE);
        unsigned long end   = start + AL_EXTENT_SIZE / BM_BLOCK_SIZE;
        drbd_bm_set_many_bits(device, 0 /*all peers*/, start, end);
    }

    // After this, all extents that were active in the AL are marked OOS
    // They will be resynced after reconnect
}
```

```bash
grep -n "lc_for_each\|lc_index_of\|lc_slot_in_use" drbd/drbd-kernel-compat/lru_cache.c drbd/drbd-kernel-compat/linux/lru_cache.h
```

---

## 8. Reading the AL from Disk (`drbd_al_read_log()`)

```bash
grep -n "drbd_al_read_log\b\|al_read_log\b" drbd/drbd_actlog.c
grep -n -A 100 "^static int drbd_al_read_log\b" drbd/drbd_actlog.c
```

```c
static int drbd_al_read_log(struct drbd_device *device,
                              struct drbd_backing_dev *bdev)
{
    // Read all transaction sectors from metadata
    // Walk them in tr_number order (ring buffer)
    // Reconstruct the AL state by applying updates

    // For each transaction:
    //   validate magic + CRC32C
    //   if valid and tr_number is newest seen: apply updates to in-memory AL

    // The "context" entries in each transaction let us skip reading old transactions
    // (context is a snapshot — sufficient to rebuild from last valid transaction)
}
```

Key: the circular transaction log with CRC validation means that even a partial write of the last transaction (power loss mid-transaction) can be detected and the previous valid state used instead.

---

## 9. AL Performance Implications

### The Flush Bottleneck

Every AL miss requires a synchronous `REQ_PREFLUSH | REQ_FUA` write to the metadata device. On spinning disks this is ~10ms. On NVMe it is ~100µs.

For **sequential writes**: few AL misses (extents are reused). Fast.

For **random writes to a large dataset** with many concurrent extents: frequent AL misses → frequent flushes → severe performance degradation.

**Mitigation:** Use a separate NVMe device or NVDIMM for metadata:
```
resource r0 {
  device minor 0;
  disk /dev/sda;
  meta-disk /dev/nvme0n1p1;    ← fast flash for AL + bitmap
}
```

### AL Size (`al-extents`) Trade-off

| `al-extents` | Flush frequency | Crash recovery time |
|---|---|---|
| 128 (small) | High (covers only 512 MiB) | Fast (few extents to resync) |
| 1024 (default) | Medium (covers 4 GiB) | ~4s per extent × 1024 extents = manageable |
| 6433 (large) | Low | Could be 25 GiB of resync on recovery |

```bash
grep -n "al_extents\|DRBD_AL_EXTENTS\|al_ext_needed" drbd/drbd_nl.c drbd/drbd_int.h | head -15
```

---

## 10. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Read `drbd_actlog.c` end-to-end
It is ~1000 lines. For every function, write its name and what it does. Pay special attention to:
- `drbd_al_begin_io_fastpath/_nonblock/_commit()`
- `drbd_al_write_transaction()`
- `drbd_al_apply_to_bm()`
- `drbd_al_read_log()`

### Exercise 2 (40 min): Trace the AL disk write
```bash
grep -n "REQ_PREFLUSH\|REQ_FUA\|submit_bio\|drbd_md_sync_page_io" drbd/drbd_actlog.c drbd/drbd_main.c | head -20
```
Find `drbd_md_sync_page_io()`. What does it do? How does it differ from a normal bio submission? Why is `REQ_FUA` necessary (vs just `REQ_PREFLUSH`)?

### Exercise 3 (40 min): Read `lru_cache.c` completely
~600 lines. Understand:
- `lc_create()` / `lc_destroy()`
- `lc_get()` — what does it return on hit vs miss?
- `lc_put()` — when does it make an element evictable?
- `lc_find()` — hash lookup, no side effects
- `lc_try_lock_for_transaction()` — what is the AL lock for?

### Exercise 4 (30 min): Compute AL miss rate for a workload
Assume:
- DRBD device: 1 TiB
- `al-extents`: 1024 (covers 4 GiB of active write area)
- Workload: 100% random 4K writes across the entire 1 TiB
- Write rate: 10,000 IOPS

What fraction of writes cause an AL miss? How many metadata flushes per second does that cause? What does this mean for latency?

### Exercise 5 (40 min): Find where AL is initialised on device creation
```bash
grep -n "lc_create\b\|act_log\s*=" drbd/drbd_main.c drbd/drbd_nl.c | head -20
```
What parameters are passed to `lc_create()`? How does `al-extents` from `drbd.conf` reach this call? Trace the path from Netlink attribute to `lc_create()`.

---

## Summary

The Activity Log is a circular, CRC-validated, per-transaction on-disk log at the metadata device. In memory it is managed as an LRU cache of `al-extents` slots, each mapping a slot index to an extent number. Every write must reserve a slot (`lc_get`) before touching local disk; if the extent is new, a synchronous `REQ_FUA` write to metadata is required — this is the primary AL performance bottleneck. On crash recovery, `drbd_al_apply_to_bm()` marks all AL-active extents as OOS in the bitmap, ensuring only affected extents are resynced.

**Next:** Day 9 — The OOS bitmap: in-memory layout, page management, bit operations, and async disk I/O.
