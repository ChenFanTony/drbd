# Day 9 — The OOS Bitmap: Layout, Bit Operations & Async Disk I/O

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_bitmap.c` (~1800 lines), `drbd/drbd_int.h`

---

## 1. What the Bitmap Tracks

The OOS (Out-Of-Sync) bitmap tracks, at **4 KiB block granularity**, which blocks of the DRBD device differ between the local node and each peer.

```
bit = 0  →  block is in-sync with this peer (or peer unknown)
bit = 1  →  block may differ from this peer (must resync)
```

Critical properties:
- **Per-peer:** A node with N peers has N independent bitmap slots
- **Persistent:** Written to metadata area for crash durability
- **In-memory:** Plain array of `unsigned long` words (one bit per 4 KiB)
- **Granularity:** 4 KiB = `BM_BLOCK_SIZE` = `1 << BM_BLOCK_SHIFT` (BM_BLOCK_SHIFT=12)

---

## 2. Bitmap Size Calculations

```bash
grep -n "BM_BLOCK_SHIFT\|BM_BLOCK_SIZE\|BM_SECT_TO_BIT\|BM_BIT_TO_SECT\|BM_BITS_PER_PAGE" \
    drbd/drbd_bitmap.c drbd/drbd_int.h
```

```c
#define BM_BLOCK_SHIFT    12              // 4 KiB per bit
#define BM_BLOCK_SIZE     (1 << 12)       // 4096 bytes
#define BM_SECT_TO_BIT(x) ((x) >> (BM_BLOCK_SHIFT - 9))
//     sector (512B units) >> 3  =  sector / 8
//     e.g., sector 8 → bit 1 (second 4 KiB block)
#define BM_BIT_TO_SECT(x) ((x) << (BM_BLOCK_SHIFT - 9))
//     bit << 3 = first sector of that 4 KiB block

#define BITS_PER_PAGE         (PAGE_SIZE * 8)    // 32768 bits per 4K page
#define BYTES_PER_BIT         BM_BLOCK_SIZE      // each bit covers 4 KiB of device
```

**Bitmap size for a 1 TiB device:**
```
total_bits  = (1 TiB) / (4 KiB) = 2^40 / 2^12 = 2^28 = 268,435,456 bits
total_bytes = 2^28 / 8 = 33,554,432 bytes = 32 MiB per peer slot
total_pages = 32 MiB / 4 KiB = 8,192 pages per peer
```

For 31 peers: 31 × 32 MiB = ~1 GiB of bitmap in memory and on metadata device.

---

## 3. `struct drbd_bitmap` — The In-Memory Representation

```bash
grep -n "struct drbd_bitmap {" drbd/drbd_bitmap.c
# Then read all fields
```

```c
struct drbd_bitmap {
    struct page **bm_pages;        // array of page pointers
                                   // bm_pages[i] = one 4 KiB page of bitmap bits
    spinlock_t bm_lock;            // protects bm_set count and bm_pages[]

    // Per-peer state (one slot per connected peer):
    struct drbd_bm_aio_ctx *bm_aio_ctx[DRBD_PEERS_MAX];

    unsigned long bm_set;          // total count of set bits (across all peers)
                                   // used for OOS sector count in /proc/drbd
    unsigned long bm_bits;         // total bits (= device_size / BM_BLOCK_SIZE)
    size_t bm_words;               // bm_bits / BITS_PER_LONG
    size_t bm_number_of_pages;     // number of pages in bm_pages[]

    wait_queue_head_t bm_io_wait;  // waiters for async bitmap I/O completion
    unsigned long bm_flags;        // BM_LOCK_ALL, BM_LOCK_SET, BM_LOCK_CLEAR
};
```

---

## 4. Per-Peer Bitmap Addressing

In DRBD 9, multiple peers share a single `drbd_bitmap` structure but each peer occupies a **slot** in the per-page layout. The physical layout on disk is:

```
Metadata area:
  For each page P in [0, bm_number_of_pages):
    Peer 0 bits: page P × (bits_per_peer_page)
    Peer 1 bits: page P × (bits_per_peer_page) + peer_1_offset
    ...
    Peer N-1 bits: ...
```

Actually, in the most common layout each peer gets its own set of pages:

```bash
grep -n "peer_device->bitmap_index\|bitmap_index\|bm_slot\|DRBD_PEERS_MAX" \
    drbd/drbd_bitmap.c drbd/drbd_int.h | head -20
```

The `peer_device->bitmap_index` (or equivalent) identifies which slot in the bitmap belongs to this peer.

---

## 5. Core Bit Operations

### Setting bits (marking OOS):

```bash
grep -n "^void drbd_bm_set_bits\b\|^unsigned long drbd_bm_set_bits\b" drbd/drbd_bitmap.c
grep -n -A 40 "drbd_bm_set_bits\b" drbd/drbd_bitmap.c | head -50
```

```c
void drbd_bm_set_bits(struct drbd_device *device,
                       unsigned int peer_device_idx,
                       unsigned long s, unsigned long e)
{
    // s = start bit, e = end bit (inclusive)
    // peer_device_idx = which peer's bitmap slot

    unsigned long *p_addr;
    unsigned long bitnr;

    // Lock the bitmap
    spin_lock_irq(&b->bm_lock);

    for (bitnr = s; bitnr <= e; bitnr++) {
        // Find which page and which word
        unsigned long page_nr = bitnr / BITS_PER_PAGE;
        unsigned long word_in_page = (bitnr % BITS_PER_PAGE) / BITS_PER_LONG;
        unsigned long bit_in_word  = bitnr % BITS_PER_LONG;

        p_addr = page_address(b->bm_pages[page_nr]);

        // Set the bit (atomic not needed — protected by bm_lock)
        if (!test_and_set_bit(bit_in_word,
                               &p_addr[word_in_page + peer_offset]))
            b->bm_set++;  // increment OOS count only if was 0
    }

    spin_unlock_irq(&b->bm_lock);
}
```

**In practice**, DRBD uses optimised versions that set multiple bits at once using `set_bits_in_word()` which directly ORs `unsigned long` words:

```bash
grep -n "bm_set_full_words_outside_last_page\|__bm_op\|drbd_bm_set_many_bits" drbd/drbd_bitmap.c | head -10
```

### Clearing bits (marking in-sync):

```bash
grep -n "^void drbd_bm_clear_bits\b\|drbd_bm_clear_bits\b" drbd/drbd_bitmap.c | head -5
```

Identical logic but `test_and_clear_bit()` and `b->bm_set--`.

### Finding next set bit (resync cursor):

```bash
grep -n "drbd_bm_find_next\b" drbd/drbd_bitmap.c
grep -n -A 40 "^unsigned long drbd_bm_find_next\b" drbd/drbd_bitmap.c
```

```c
unsigned long drbd_bm_find_next(struct drbd_peer_device *peer_device,
                                 unsigned long bm_fo /* find offset */)
{
    // Search from bm_fo onwards for the next set bit
    // Uses find_next_bit() which is architecture-optimised
    // (often compiles to BSF/BSRQ instructions on x86)

    unsigned long *p_addr;
    unsigned long page_nr = bm_fo / BITS_PER_PAGE;
    unsigned long bit_ofs = bm_fo % BITS_PER_PAGE;

    while (page_nr < b->bm_number_of_pages) {
        p_addr = page_address(b->bm_pages[page_nr]);
        unsigned long result = find_next_bit(
            p_addr + peer_page_offset, BITS_PER_PAGE, bit_ofs);
        if (result < BITS_PER_PAGE)
            return page_nr * BITS_PER_PAGE + result;
        page_nr++;
        bit_ofs = 0;
    }
    return DRBD_END_OF_BITMAP;  // no more set bits
}
```

This is the **resync cursor** — the SyncSource calls this in a tight loop to find the next block to send.

---

## 6. Counting Set Bits

```bash
grep -n "drbd_bm_count_bits\|drbd_bm_total_weight\|bm_set\b" drbd/drbd_bitmap.c | head -10
```

DRBD maintains `b->bm_set` as a running count — it does not recount on every read. The count is exposed as:
```
# In /proc/drbd:
ro:Primary/Secondary ds:UpToDate/Inconsistent ...
   resync: used:0/61440K(0%) ...
   rs_left: 20971520    ← b->bm_set (in sectors, not bits)
```

---

## 7. Bitmap I/O: Writing to Disk

The bitmap is too large to write synchronously on every bit change. Instead:

**On bit set/clear:** Update in-memory only (fast, O(1)).

**On demand:** Write changed pages to disk using async I/O via `drbd_bm_aio_ctx`:

```bash
grep -n "struct drbd_bm_aio_ctx {" drbd/drbd_bitmap.c
```

```c
struct drbd_bm_aio_ctx {
    struct drbd_device *device;
    int error;
    unsigned int flags;          // BM_AIO_COPY_PAGES, BM_AIO_WRITE_HINTED, etc.
    atomic_t in_flight;          // number of bios still in flight
    wait_queue_head_t io_wait;   // wake when in_flight == 0
    unsigned long start_jif;
};
```

### `drbd_bm_write()` — Write entire bitmap to disk

```bash
grep -n -A 80 "^int drbd_bm_write\b" drbd/drbd_bitmap.c
```

```c
int drbd_bm_write(struct drbd_device *device,
                   struct drbd_peer_device *peer_device)
{
    struct drbd_bm_aio_ctx ctx = { .device = device, .in_flight = ATOMIC_INIT(1) };

    // Submit one bio per page of bitmap
    for (int i = 0; i < b->bm_number_of_pages; i++) {
        struct bio *bio = bio_alloc(GFP_NOIO, 1);
        bio->bi_bdev   = device->ldev->md_bdev;
        bio->bi_sector = drbd_md_pstruct_offset(device, MD_BM) + i * (PAGE_SIZE/512);
        bio->bi_end_io = drbd_bm_endio;
        bio->bi_private = &ctx;
        bio_add_page(bio, b->bm_pages[i], PAGE_SIZE, 0);
        bio->bi_opf = REQ_OP_WRITE;
        atomic_inc(&ctx.in_flight);
        submit_bio(bio);
    }

    // Decrement for the initial 1 (representing "not done yet")
    if (atomic_dec_and_test(&ctx.in_flight))
        complete(&ctx.io_wait);

    // Wait for all page writes to complete
    wait_for_completion(&ctx.io_wait);
    return ctx.error;
}
```

### `drbd_bm_endio()` — Bitmap write completion

```bash
grep -n -A 20 "^static void drbd_bm_endio\b" drbd/drbd_bitmap.c
```

```c
static void drbd_bm_endio(struct bio *bio)
{
    struct drbd_bm_aio_ctx *ctx = bio->bi_private;
    if (bio->bi_status)
        ctx->error = -EIO;
    if (atomic_dec_and_test(&ctx->in_flight))
        wake_up(&ctx->io_wait);
    bio_put(bio);
}
```

### When is the bitmap written to disk?

```bash
grep -n "drbd_bm_write\b" drbd/drbd_worker.c drbd/drbd_receiver.c drbd/drbd_nl.c drbd/drbd_state.c | head -20
```

| Trigger | Caller | Purpose |
|---|---|---|
| Resync progress | `drbd_worker.c: w_make_resync_request` | Persist cleared bits periodically |
| Disconnect | `drbd_receiver.c: drbd_disconnect` | Persist any last OOS bits |
| Attach | `drbd_nl.c: drbd_adm_attach` | After AL apply, write initial bitmap |
| Resync finish | `drbd_worker.c: drbd_resync_finished` | Final flush: all bits should be clear |
| Detach | `drbd_nl.c: drbd_adm_detach` | Persist current state |

---

## 8. Bitmap Read on Attach

```bash
grep -n "drbd_bm_read\b" drbd/drbd_bitmap.c drbd/drbd_nl.c | head -5
grep -n -A 60 "^int drbd_bm_read\b" drbd/drbd_bitmap.c
```

Symmetric to `drbd_bm_write()` but uses `REQ_OP_READ`. After reading, `bm_set` is recalculated by scanning all bits:

```c
b->bm_set = 0;
for each page:
    for each word in page:
        b->bm_set += hweight_long(word);  // popcount
```

---

## 9. Bitmap Locking

```bash
grep -n "drbd_bm_lock\b\|drbd_bm_unlock\b\|BM_LOCK_ALL\|BM_LOCK_SET\|BM_LOCK_CLEAR" drbd/drbd_bitmap.c | head -20
```

DRBD has two levels of bitmap locking:
1. `bm_lock` spinlock — protects individual bit operations and `bm_set` count
2. `BM_LOCK_*` flags — coarse locks used during disk I/O (prevents concurrent modifications)

During `drbd_bm_write()`, `BM_LOCK_ALL` is set to prevent bit changes while pages are being written. Callers that need to set bits while a bitmap write is in progress use `drbd_bm_write_page()` (write single page) instead.

---

## 10. When `drbd_bm_set_bits` Is Called at Sync Time

`drbd_bm_set_bits()` is never called directly in hot paths — callers go through `drbd_set_out_of_sync()` (a macro in `drbd_actlog.c`) which converts a sector range to bit range and calls `drbd_bm_set_bits()` or `drbd_bm_set_many_bits()`. There are seven distinct call sites, each corresponding to a distinct reason a block becomes OOS:

### 10.1 Resync write failed at SyncTarget

**Location:** `drbd_sender.c` — `e_end_block()`

**Trigger:** SyncTarget receives a resync block from SyncSource and writes it locally. If the local write fails (`EE_WAS_ERROR` flag set on the `drbd_peer_request`), the block cannot be cleared. `e_end_block()` calls `drbd_set_out_of_sync()` to keep the bit set (or re-mark it if it was prematurely cleared).

```bash
grep -n "EE_WAS_ERROR\|drbd_set_out_of_sync" drbd/drbd_sender.c | head -10
# line ~192: drbd_set_out_of_sync() in e_end_block when EE_WAS_ERROR
```

### 10.2 Resync block unreadable at SyncSource

**Location:** `drbd_sender.c` — `drbd_rs_failed_io()`

**Trigger:** SyncSource cannot read the block from its local disk (read error during `w_make_resync_request`). `drbd_rs_failed_io()` marks the block OOS so the resync cursor skips it and records it for retry.

### 10.3 Receive `P_OUT_OF_SYNC` (L_BEHIND)

**Location:** `drbd_receiver.c:8822` — `receive_out_of_sync()`

**Trigger:** SyncSource sends `P_OUT_OF_SYNC` to inform the SyncTarget that a range is out of sync. The SyncTarget calls `drbd_set_out_of_sync()` directly — no data is received, just the notification. Used when SyncSource skips blocks (e.g., verify mismatch) rather than sending data.

```bash
grep -n "receive_out_of_sync\b" drbd/drbd_receiver.c
# line ~8822
```

### 10.4 Online Verify mismatch

**Location:** `drbd_sender.c:2352` — `drbd_ov_out_of_sync_found()`

**Trigger:** During online verify (`drbd-utils: drbdadm verify`), checksums of a block are compared between primary and secondary without stopping I/O. When checksums differ, `drbd_ov_out_of_sync_found()` marks the block OOS. The block will later be resynced.

```bash
grep -n "drbd_ov_out_of_sync_found\b" drbd/drbd_sender.c
# line ~2352
```

### 10.5 `P_DATA` write on secondary with degraded disk

**Location:** `drbd_receiver.c:3377` — `receive_Data()` + `drbd_receiver.c:1727` — `drbd_submit_peer_request()`

**Trigger:** Secondary receives a `P_DATA` packet but its local disk state is `< D_INCONSISTENT`. `receive_Data()` sets the `EE_SET_OUT_OF_SYNC` flag on the `drbd_peer_request`. When the (failed or skipped) write completes, `drbd_submit_peer_request()` processes the flag and marks the block OOS via `drbd_set_out_of_sync()`.

```bash
grep -n "EE_SET_OUT_OF_SYNC" drbd/drbd_receiver.c
# line ~3377 (set), ~1727 (processed)
```

### 10.6 Late `P_NEG_ACK`

**Location:** `drbd_receiver.c:10780` — `got_NegAck()`

**Trigger:** Primary receives a `P_NEG_ACK` (negative acknowledgment) from the secondary — the secondary's local write failed *after* accepting the data. The primary calls `drbd_set_out_of_sync()` to mark those blocks as OOS for that peer, so they will be queued for resync.

```bash
grep -n "got_NegAck\b" drbd/drbd_receiver.c
# line ~10780
```

### 10.7 Crash recovery (AL → bitmap)

**Location:** `drbd_actlog.c` — `drbd_al_apply_to_bm()`

**Trigger:** Called by `drbdadm apply-al` (userspace) before kernel attach. Walks all occupied AL slots (extents that were "in flight" when the node crashed) and calls `drbd_bm_set_many_bits()` for each. Each 4 MB AL extent maps to 1024 bitmap bits (1024 × 4 KiB = 4 MiB). This is the coarsest call site — it sets bits in bulk.

```bash
grep -n "drbd_al_apply_to_bm\b" drbd/drbd_actlog.c
grep -n -A 20 "^void drbd_al_apply_to_bm\b" drbd/drbd_actlog.c
```

### Summary table

| # | Call site | File | Trigger |
|---|---|---|---|
| 1 | `e_end_block()` | `drbd_sender.c:192` | SyncTarget local write failure |
| 2 | `drbd_rs_failed_io()` | `drbd_sender.c` | SyncSource local read failure |
| 3 | `receive_out_of_sync()` | `drbd_receiver.c:8822` | P_OUT_OF_SYNC packet received |
| 4 | `drbd_ov_out_of_sync_found()` | `drbd_sender.c:2352` | Online Verify checksum mismatch |
| 5 | `drbd_submit_peer_request()` | `drbd_receiver.c:1727` | P_DATA on degraded-disk secondary |
| 6 | `got_NegAck()` | `drbd_receiver.c:10780` | Late P_NEG_ACK from secondary |
| 7 | `drbd_al_apply_to_bm()` | `drbd_actlog.c` | Crash recovery (AL → bitmap) |

---

## 11. When `drbd_bm_clear_bits` Is Called

Bits are cleared via `drbd_set_in_sync()` — the symmetric counterpart to `drbd_set_out_of_sync()`. There are three triggers:

### 11.1 Successful resync block (primary path)

**Location:** `drbd_receiver.c` — `got_RSWriteAck()`

The full flow across both sides:

```
SyncSource: drbd_bm_find_next() → finds set bit
SyncSource: reads block from local disk
SyncSource: sends P_RS_DATA_REPLY to SyncTarget

SyncTarget: writes block locally → success
SyncTarget: sends P_RS_WRITE_ACK back to SyncSource

SyncSource: got_RSWriteAck() → drbd_set_in_sync() → drbd_bm_clear_bits()
SyncTarget: also clears its own copy of the bit on successful write
```

Both sides maintain their own per-peer bitmap and clear the bit independently — SyncTarget on write success, SyncSource on ACK reception.

```bash
grep -n "got_RSWriteAck\b\|drbd_set_in_sync\b" drbd/drbd_receiver.c | head -10
```

### 11.2 Online Verify confirms match

**Location:** `drbd_sender.c` — `drbd_ov_in_sync_found()`

During online verify no data is transferred — only checksums are compared. When checksums match, `drbd_ov_in_sync_found()` calls `drbd_set_in_sync()` to clear the bit. This can clear bits that were set by `apply-al` even though the actual data was never different — the verify confirms they never diverged.

```bash
grep -n "drbd_ov_in_sync_found\b" drbd/drbd_sender.c
```

### 11.3 Resync finished

**Location:** `drbd_worker.c` — `drbd_resync_finished()`

When `bm_set` drops to 0 (all resync bits cleared by path 11.1 above), `drbd_resync_finished()` transitions the replication state from `L_SYNC_SOURCE`/`L_SYNC_TARGET` back to `L_ESTABLISHED` and writes the now-zero bitmap to disk via `drbd_bm_write()`.

```bash
grep -n "drbd_resync_finished\b" drbd/drbd_worker.c
```

### 11.4 Full bit lifecycle

```
Normal application I/O (connected, replication working):
  bit stays 0 — never touched during successful writes

Connection drops with writes in-flight:
  tl_clear() → bit = 1  (block is OOS with this peer)

Reconnect, resync runs:
  SyncSource sends block → SyncTarget ACKs → bit = 0  (confirmed in-sync)

Online Verify (no data transfer):
  checksums match → bit = 0  (confirmed never diverged)
  checksums differ → bit = 1  (OOS, queued for resync)
```

Bits are **never** cleared during application I/O. Clearing means "both sides have confirmed this block agrees" — only resync or verify can make that confirmation.

### Summary table

| Trigger | Call site | File |
|---|---|---|
| Resync block ACKed by SyncTarget | `got_RSWriteAck()` | `drbd_receiver.c` |
| Online Verify checksums match | `drbd_ov_in_sync_found()` | `drbd_sender.c` |
| Resync complete, all bits gone | `drbd_resync_finished()` | `drbd_worker.c` |

---

## 12. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Read `drbd_bitmap.c` in full
~1800 lines. For every exported function (no `static`), write a one-sentence description. Identify all functions that do disk I/O.

### Exercise 2 (40 min): Compute bitmap layout for your device
```bash
# Find your metadata structure offset formula
grep -n "drbd_md_pstruct_offset\|MD_AL_OFFSET\|MD_BM_OFFSET\|md_offset" drbd/drbd_main.c drbd/drbd_actlog.c
```
Given: 500 GiB DRBD device, 2 peers, `meta-disk internal`. Calculate:
- Total bitmap bits per peer
- Total pages per peer
- Total metadata bytes for bitmap (both peers)
- Sector offset where bitmap starts on the device

### Exercise 3 (40 min): Trace `drbd_bm_set_bits()` called from `drbd_al_apply_to_bm()`
```bash
grep -n "drbd_bm_set_many_bits\|drbd_bm_set_bits" drbd/drbd_actlog.c
```
How many bits are set per AL extent? What is the range `[start, end)` in bit units?

### Exercise 4 (30 min): Find and analyse `drbd_bm_find_next()`
```bash
grep -n -A 50 "^unsigned long drbd_bm_find_next\b" drbd/drbd_bitmap.c
```
What happens when all bits are clear? What is `DRBD_END_OF_BITMAP` and where is it defined?

### Exercise 5 (40 min): Trace the async bitmap write
Starting from `drbd_resync_finished()` → `drbd_bm_write()` → bio submission → `drbd_bm_endio()`:
```bash
grep -n "drbd_resync_finished\b" drbd/drbd_worker.c
grep -n -A 10 "drbd_bm_write\b" drbd/drbd_worker.c
```
How many bios are submitted for a 100 GiB device? What is the total bytes written?

---

## Summary

The OOS bitmap is a per-peer bit array at 4 KiB granularity, managed in memory as an array of `struct page *` pointers. Bit operations (`set`, `clear`, `find_next`) are O(1) or O(n/BITS_PER_LONG) and protected by a spinlock. Disk I/O is asynchronous, page-granular, and tracked via `drbd_bm_aio_ctx`. The bitmap is written on disconnect, resync progress, and resync completion; read on disk attach. The `bm_set` running count directly drives the resync progress display and determines when resync is complete.

**Next:** Day 10 — The resync engine: `w_make_resync_request()`, burst scheduling, SyncSource/SyncTarget paths.
