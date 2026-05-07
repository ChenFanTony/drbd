# Day 14 — Backing Device Management: Attach, Detach & Local I/O Error Handling

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_nl.c`, `drbd/drbd_main.c`, `drbd/drbd_actlog.c`, `drbd/drbd_int.h`

---

## 1. `struct drbd_backing_dev` — Everything About the Local Disk

```bash
grep -n "struct drbd_backing_dev {" drbd/drbd_int.h
# Read every field
```

```c
struct drbd_backing_dev {
    struct block_device *backing_bdev;  // the actual data storage device
                                        // e.g., /dev/sda, /dev/lvm/data
    struct block_device *md_bdev;       // metadata device
                                        // == backing_bdev for "internal" metadata
                                        // different device for "external" metadata

    struct file *backing_bdev_file;     // reference to backing_bdev (kernel ≥ 6.5)
    struct file *md_bdev_file;

    struct drbd_md md;                  // in-memory copy of on-disk metadata

    struct disk_conf *disk_conf;        // parsed disk configuration:
                                        // fencing, al-extents, resync-rate,
                                        // on-io-error, no-disk-barrier, etc.

    sector_t known_size;                // cached capacity in sectors
    sector_t d_size;                    // drbd device size (may be less than real disk)
    sector_t u_size;                    // user-configured size override

    int recounting;                     // ref-count guard for size recalculation
};
```

### `struct drbd_md` — The In-Memory Metadata Superblock

```bash
grep -n "struct drbd_md {" drbd/drbd_int.h
```

```c
struct drbd_md {
    u64 md_offset;          // byte offset of metadata superblock on md_bdev
    u64 al_offset;          // byte offset of activity log area
    u64 bm_offset;          // byte offset of bitmap area

    u32 al_stripes;         // number of AL stripes (usually 1)
    u32 al_stripe_size_4k;  // AL stripe size in 4K units

    u32 md_size_sect;       // total metadata area size in 512B sectors
    u32 al_offset_sect;     // AL area offset in sectors from md start
    u32 bm_offset_sect;     // bitmap area offset in sectors from md start

    u64 current_uuid;       // current generation UUID
    u64 flags;              // MDF_* flags (see below)
    u32 magic;              // DRBD_MD_MAGIC_09

    // Per-peer state stored in metadata:
    // (node_id → bitmap_uuid, flags)
};
```

```bash
grep -n "MDF_CONSISTENT\|MDF_PRIMARY_IND\|MDF_CONNECTED_IND\|MDF_FULL_SYNC\|MDF_PEER_OUT_DATED" \
    drbd/drbd_int.h | head -15
```

| `MDF_` flag | Meaning |
|---|---|
| `MDF_CONSISTENT` | Disk was consistent at last clean shutdown |
| `MDF_PRIMARY_IND` | Was Primary at last shutdown (set on promote, cleared on demote) |
| `MDF_CONNECTED_IND` | Was connected to all peers at shutdown |
| `MDF_FULL_SYNC` | Next attach requires full resync (forced by admin) |
| `MDF_PEER_OUT_DATED` | Peer was outdated at last shutdown |
| `MDF_CRASHED_PRIMARY` | Was primary, crashed without clean shutdown |

---

## 2. `drbd_adm_attach()` — The Full Disk Attach Sequence

```bash
grep -n "drbd_adm_attach\b" drbd/drbd_nl.c
grep -n -A 300 "^int drbd_adm_attach\b" drbd/drbd_nl.c
```

Annotated call chain (major steps):

```
drbd_adm_attach(skb, info)   [called from Generic Netlink handler]
│
├─ 1. Parse Netlink attributes → fill disk_conf struct
│   └── drbd_adm_prepare() + nla_parse() + disk_conf_from_attrs()
│       // disk-conf: backing-dev, meta-disk, al-extents, fencing, etc.
│
├─ 2. Allocate drbd_backing_dev
│   └── nbc = kzalloc(sizeof(*nbc), GFP_KERNEL)
│       nbc->disk_conf = new_disk_conf
│
├─ 3. Open the backing block device
│   └── nbc->backing_bdev = blkdev_get_by_path(disk_conf->backing_dev,
│                               FMODE_READ | FMODE_WRITE | FMODE_EXCL, nbc)
│       // FMODE_EXCL: exclusive open — prevents other users
│       // Fails if device is already mounted or in use
│
├─ 4. Open the metadata block device
│   └── if (internal metadata):
│           nbc->md_bdev = nbc->backing_bdev   // same device
│       else:
│           nbc->md_bdev = blkdev_get_by_path(disk_conf->meta_dev, ...)
│
├─ 5. Determine device size
│   └── nbc->known_size = drbd_get_max_capacity(nbc)
│       // = min(backing_bdev_size, md_bdev_size - metadata_overhead)
│
├─ 6. Allocate activity log
│   └── lc = lc_create("act_log", drbd_al_ext_cache,
│                        AL_UPDATES_PER_TRANSACTION,
│                        new_disk_conf->al_extents,
│                        sizeof(struct lc_element), 0)
│       device->act_log = lc
│
├─ 7. Allocate / resize bitmap
│   └── drbd_bm_resize(device, nbc->known_size, 0)
│       // Allocates bm_pages[] array for new device size
│       // If peer already attached: reallocates for correct peer count
│
├─ 8. State change: disk_state → D_ATTACHING
│   └── change_disk_state(device, D_ATTACHING, CS_VERBOSE, ...)
│
├─ 9. Read metadata from disk
│   └── drbd_md_read(device, nbc)
│       │
│       ├── drbd_md_sync_page_io(device, nbc, md_offset, READ)
│       │   // Read the metadata superblock page
│       ├── Validate magic + CRC32C checksum
│       ├── Extract: current_uuid, flags, al_offset, bm_offset
│       └── Returns -EINVAL if metadata is corrupt / unformatted
│
├─ 10. Activity log crash recovery
│    └── if (MDF_PRIMARY_IND set && !MDF_CONSISTENT):
│            // We were Primary and crashed
│            drbd_al_apply_to_bm(device)
│            // Marks all AL-active extents OOS in bitmap
│
├─ 11. Read bitmap from disk
│    └── drbd_bm_read(device, peer_device)
│        // For each peer: reads bitmap pages, counts bm_set
│
├─ 12. Connect device to backing device (the point of no return)
│    └── spin_lock_irq(&resource->req_lock)
│        rcu_assign_pointer(device->ldev, nbc)
│        spin_unlock_irq(...)
│        // From this point: drbd_make_request() can use local disk
│
├─ 13. Determine initial disk state
│    └── Based on MDF_* flags and UUID comparison with peers:
│        ├── No peer ever connected: D_INCONSISTENT (need full sync)
│        ├── Was UpToDate at last shutdown (MDF_CONSISTENT set): → D_CONSISTENT
│        ├── Crashed as Primary: → D_INCONSISTENT (AL-based partial resync)
│        └── (final state negotiated with peer via UUID exchange)
│
└─ 14. State change: disk_state → D_NEGOTIATING or D_UP_TO_DATE etc.
    └── change_disk_state(device, determined_state, CS_VERBOSE, ...)
        // __after_state_change() may trigger bitmap exchange / resync
```

---

## 3. `drbd_md_read()` — Reading the Metadata Superblock

```bash
grep -n -A 80 "^int drbd_md_read\b" drbd/drbd_main.c
```

```c
int drbd_md_read(struct drbd_device *device, struct drbd_backing_dev *bdev)
{
    struct meta_data_on_disk *buffer;

    // Read 4 KiB metadata page
    drbd_md_sync_page_io(device, bdev, bdev->md.md_offset, READ);

    buffer = drbd_md_get_buffer(device, __func__);

    // Validate
    if (buffer->magic != cpu_to_be32(DRBD_MD_MAGIC_09))
        return ERR_MD_INVALID;

    // Verify CRC32C
    u32 stored_crc  = be32_to_cpu(buffer->crc32c_checksum);
    u32 computed_crc = crc32c(0, buffer, sizeof(*buffer) - sizeof(u32));
    if (stored_crc != computed_crc)
        return ERR_MD_INVALID;

    // Copy fields into in-memory drbd_md
    bdev->md.current_uuid   = be64_to_cpu(buffer->uuid[UI_CURRENT]);
    bdev->md.flags          = be32_to_cpu(buffer->flags);
    bdev->md.al_offset      = be64_to_cpu(buffer->al_offset);
    bdev->md.bm_offset      = be64_to_cpu(buffer->bm_offset);
    bdev->md.md_size_sect   = be32_to_cpu(buffer->md_size_sect);
    // ...

    drbd_md_put_buffer(device);
    return 0;
}
```

---

## 4. `drbd_adm_detach()` — The Disk Detach Sequence

```bash
grep -n -A 100 "^int drbd_adm_detach\b" drbd/drbd_nl.c
```

```
drbd_adm_detach(skb, info)
│
├─ 1. Guard: cannot detach if device is Primary (data loss risk)
│   └── if (device->role[NOW] == R_PRIMARY && !force)
│           return ERR_PRIMARY; // use --force to override
│
├─ 2. State change: disk_state → D_DETACHING
│   └── change_disk_state(device, D_DETACHING, CS_VERBOSE, ...)
│       // This suspends further local I/O
│
├─ 3. Wait for in-flight local I/O to drain
│   └── wait_event(device->misc_wait,
│               atomic_read(&device->local_cnt) == 0)
│       // local_cnt is incremented by get_ldev_if_state()
│       // decremented by put_ldev()
│       // All write requests hold a reference; all must complete
│
├─ 4. Flush and free the activity log
│   └── drbd_al_shrink(device)   // wait for pending AL writes
│       lc_destroy(device->act_log)
│       device->act_log = NULL
│
├─ 5. Flush metadata to disk
│   └── drbd_md_sync(device)     // write final metadata
│
├─ 6. Write final bitmap state to disk
│   └── drbd_bm_write(device, NULL) // write all pages
│
├─ 7. Disconnect from backing device
│   └── spin_lock_irq(&resource->req_lock)
│       old_ldev = device->ldev
│       RCU_INIT_POINTER(device->ldev, NULL)
│       spin_unlock_irq(...)
│       // After this: drbd_make_request() will reject local I/O
│
├─ 8. Close block devices
│   └── blkdev_put(old_ldev->backing_bdev, ...)
│       if (external metadata): blkdev_put(old_ldev->md_bdev, ...)
│
└─ 9. Free drbd_backing_dev
    └── kfree(old_ldev->disk_conf)
        kfree(old_ldev)
        // State: disk_state → D_DISKLESS
```

---

## 5. `get_ldev_if_state()` / `put_ldev()` — Reference Counting for the Local Disk

The local disk can be detached at any time. Every code path that accesses `device->ldev` must hold a reference:

```bash
grep -n "get_ldev_if_state\|get_ldev\b\|put_ldev\b" drbd/drbd_int.h drbd/drbd_req.c | head -20
```

```c
// drbd_int.h:
#define get_ldev_if_state(M, MINS) \
    (_get_ldev_if_state((M), (MINS)))

static inline bool _get_ldev_if_state(struct drbd_device *device,
                                       enum drbd_disk_state mins)
{
    // Atomically: check disk_state[NOW] >= mins AND increment local_cnt
    bool rv;
    rcu_read_lock();
    rv = (device->disk_state[NOW] >= mins);
    if (rv)
        atomic_inc(&device->local_cnt);
    rcu_read_unlock();
    return rv;
}

#define put_ldev(M) \
    do { \
        if (atomic_dec_and_test(&(M)->local_cnt)) \
            wake_up(&(M)->misc_wait); \
    } while (0)
```

**Usage pattern in drbd_req.c:**
```c
// Before accessing local disk:
if (!get_ldev_if_state(device, D_UP_TO_DATE)) {
    // Disk not available at required state
    bio_endio(bio, -EIO);
    return;
}

// ... do local disk I/O ...

// When done (in endio callback or error path):
put_ldev(device);
```

The `detach` sequence waits for `local_cnt == 0` — meaning all in-flight requests have called `put_ldev()`.

---

## 6. Local I/O Error Handling

### 6.1 Write Errors

When `drbd_request_endio()` is called with `bio->bi_status != 0` for a write:

```bash
grep -n "WRITE_COMPLETED_WITH_ERROR\|drbd_handle_failed_mirror\|DRBD_FAULT_DT_WR\|on_io_error" \
    drbd/drbd_req.c drbd/drbd_main.c | head -20
```

```c
// drbd_req.c — drbd_request_endio():
if (bio->bi_status) {
    if (bio_op(bio) == REQ_OP_WRITE) {
        // Failed write to local disk
        req->private_bio = ERR_PTR(-EIO);
        __req_mod(req, WRITE_COMPLETED_WITH_ERROR, NULL, &m);
    }
}
```

Inside `__req_mod()` for `WRITE_COMPLETED_WITH_ERROR`:
```bash
grep -n "WRITE_COMPLETED_WITH_ERROR" drbd/drbd_req.c | head -10
grep -n -A 30 "case WRITE_COMPLETED_WITH_ERROR:" drbd/drbd_req.c
```

Depending on `disk_conf->on_io_error` config:
- `EP_DETACH`: trigger disk detach (safest — node becomes diskless)
- `EP_CALL_HELPER`: run `drbdadm` helper script, then maybe detach
- `EP_PASSTHROUGH`: pass the error to the application (no auto-detach)

```bash
grep -n "EP_DETACH\|EP_CALL_HELPER\|EP_PASSTHROUGH\|on_io_error\b" drbd/drbd_int.h drbd/drbd_req.c | head -15
```

### 6.2 Read Errors (for application reads)

```bash
grep -n "READ_COMPLETED_WITH_ERROR\|drbd_send_drequest\|P_DATA_REQUEST" drbd/drbd_req.c drbd/drbd_sender.c | head -15
```

If a local read fails and the primary has a connected peer that is UpToDate:
1. Primary sends `P_DATA_REQUEST` to secondary (asking it to read the block)
2. Secondary reads and sends `P_DATA_REPLY` with the data
3. Primary forwards the data to the application

```bash
grep -n -A 40 "^static int receive_DataReply\b" drbd/drbd_receiver.c
```

### 6.3 Read Errors (for resync)

If the SyncSource's local read fails during resync:
```c
// w_e_send_resync_data():
if (test_bit(EE_WAS_ERROR, &peer_req->flags)) {
    drbd_send_ack(peer_device, P_NEG_RS_DREPLY, peer_req);
    drbd_rs_failed_io(peer_device, sector, size);
    // rs_failed counter incremented
    // Block stays OOS on both sides
}
```

At resync completion, `drbd_resync_finished()` checks:
```c
if (peer_device->rs_failed) {
    // Not all blocks could be resynced
    // Call drbd_resync_finished(peer_device, D_INCONSISTENT)
    // (not UpToDate — some blocks remain OOS)
}
```

---

## 7. Fencing — Protecting Against Split-Brain After Disk Errors

```bash
grep -n "fencing\|DRBD_FENCING_FP\|DRBD_FENCING_RA\|DRBD_FENCING_PP\|drbd_fencing_policy" \
    drbd/drbd_int.h drbd/drbd_nl.c drbd/drbd_state.c | head -20
```

| Fencing policy | Value | Behaviour on I/O error |
|---|---|---|
| `dont-care` | `FP_DONT_CARE` | No fencing — continue as Primary |
| `resource-only` | `FP_RESOURCE` | Demote to Secondary if disk fails |
| `resource-and-stonith` | `FP_STONITH` | STONITH (shoot-the-other-node) + demote |

When fencing is triggered:
```bash
grep -n "drbd_fence_peer\|_drbd_may_suspend\|MDF_CRASHED_PRIMARY\|DRBD_FENCING" \
    drbd/drbd_state.c drbd/drbd_main.c | head -15
```

---

## 8. Metadata Area Layout — How Offsets Are Calculated

```bash
grep -n "drbd_md_set_sector_offsets\b\|drbd_md_first_sector\b\|drbd_md_last_sector\b\|META_DATA_SIZE\b" \
    drbd/drbd_main.c drbd/drbd_int.h | head -20
grep -n -A 50 "^void drbd_md_set_sector_offsets\b" drbd/drbd_main.c
```

For **internal metadata** (metadata at end of backing device):
```
[<─────────── data area ────────────>] [<── metadata area ──>]
0                                    d  d+al_area  d+al+bm   end
                                     │
                                     md_offset (superblock at end)

Layout from end of device:
  Last sector(s): metadata superblock (1 sector = 512B, padded to 4K)
  Before that:    bitmap area (variable, based on device size + peer count)
  Before that:    activity log area (fixed: al_stripes × al_stripe_size sectors)
```

```bash
grep -n "MD_AL_MAX_SECT\|MD_BM_SECTOR\|DRBD_MD_MAGIC_09\|MD_AL_OFFSET" drbd/drbd_int.h drbd/drbd_main.c | head -15
```

---

## 9. Week 2 Integration — Putting the Storage Layer Together

```
Application write → drbd_make_request()
    │
    ├─ get_ldev_if_state(D_UP_TO_DATE)    ← checks ldev != NULL and disk_state
    │      ↳ increments local_cnt
    │
    ├─ drbd_al_begin_io()                 ← reserves AL slot
    │      ↳ lc_get() → possible AL transaction write to md_bdev
    │
    ├─ drbd_insert_interval()             ← overlap tracking
    │
    ├─ submit_bio(private_bio)            ← local write to backing_bdev
    │
    ├─ drbd_send_dblock()                 ← network write to peer
    │
    └─ [on completion]:
         drbd_al_complete_io()            ← releases AL slot (lc_put)
         drbd_remove_interval()           ← removes from interval tree
         put_ldev()                       ← decrements local_cnt
         bio_endio(master_bio)            ← application sees write complete

[disk detach]:
    disk_state → D_DETACHING
    wait(local_cnt == 0)                  ← all writes must release get_ldev
    drbd_al_shrink()                      ← flush AL
    drbd_md_sync()                        ← flush metadata
    drbd_bm_write()                       ← flush bitmap
    device->ldev = NULL                   ← no more local I/O
    blkdev_put(backing_bdev)             ← close the device
    disk_state → D_DISKLESS
```

---

## 10. Hands-On Exercises (3–4 hours)

### Exercise 1 (60 min): Read `drbd_adm_attach()` in full
```bash
grep -n -A 400 "^int drbd_adm_attach\b" drbd/drbd_nl.c
```
For every error path (`goto out*`): what resources are cleaned up? Is there a resource leak if error occurs between step 5 and step 12?

### Exercise 2 (40 min): Read `drbd_md_read()` and `drbd_md_sync()` side by side
```bash
grep -n -A 80 "^int drbd_md_read\b" drbd/drbd_main.c
grep -n -A 80 "^void drbd_md_sync\b" drbd/drbd_main.c
```
For every field written by `drbd_md_sync()`, verify that `drbd_md_read()` reads it back. Are there any fields written but not read, or vice versa?

### Exercise 3 (35 min): Trace local write error path
```bash
grep -n "WRITE_COMPLETED_WITH_ERROR\|drbd_handle_failed_mirror\|on_io_error" drbd/drbd_req.c | head -15
```
Starting from `drbd_request_endio()` called with `bi_status = BLK_STS_IOERR`:
1. What `__req_mod()` event is sent?
2. What does `__req_mod()` do with `rq_state`?
3. What is the `on-io-error = detach` code path?
4. Does `bio_endio()` still get called with an error to the application?

### Exercise 4 (35 min): Trace the metadata area layout for a specific device
Given: 100 GiB backing device, 2 peers, internal metadata, `al-extents = 1024`:
```bash
grep -n "drbd_md_set_sector_offsets\b" drbd/drbd_main.c
grep -n -A 50 "^void drbd_md_set_sector_offsets\b" drbd/drbd_main.c
```
Calculate:
- Bitmap size (bits, bytes, sectors) per peer
- AL size (sectors): `al_stripes * al_stripe_size_4k * 8` sectors
- Total metadata area size in sectors
- Data area capacity = total - metadata

### Exercise 5 (30 min): Understand `MDF_CRASHED_PRIMARY` detection
```bash
grep -n "MDF_CRASHED_PRIMARY\|MDF_PRIMARY_IND\|MDF_CONSISTENT" drbd/drbd_main.c drbd/drbd_nl.c | head -20
```
Trace:
1. `MDF_PRIMARY_IND` is SET when? (where in code)
2. `MDF_PRIMARY_IND` is CLEARED when?
3. `MDF_CONSISTENT` is SET when?
4. `MDF_CONSISTENT` is CLEARED when?
5. On attach: what combination of these flags means "crashed as primary"?

---

## Summary

`drbd_backing_dev` holds references to both the backing data device and the metadata device, plus the parsed disk configuration. `drbd_adm_attach()` opens both devices, reads metadata, applies crash recovery via `drbd_al_apply_to_bm()`, reads the bitmap, and atomically installs `device->ldev`. Every code path accessing the local disk holds a reference via `get_ldev_if_state()` / `put_ldev()`; `drbd_adm_detach()` waits for `local_cnt == 0` before closing the device. Local I/O errors are handled per `on-io-error` policy (detach, call-helper, or passthrough), and fencing policy controls whether the node demotes from Primary on disk failure.

---

## Week 2 Complete — What You Now Know

| Day | Topic | Key takeaway |
|---|---|---|
| 8 | Activity Log | Circular on-disk transaction log; AL miss = synchronous REQ_FUA flush |
| 9 | OOS Bitmap | Per-peer bit array; `bm_find_next()` drives resync cursor |
| 10 | Resync Engine | Timer-driven burst loop; `rs_controller()` rate limiting |
| 11 | Worker Thread | All `w_*` callbacks; `drbd_md_mark_dirty()` coalescing pattern |
| 12 | Interval Tree | Augmented RB-tree for O(log n) overlap detection |
| 13 | LRU Cache | Generic cache with transactional eviction; `lc_get/put/committed` |
| 14 | Backing Device | Attach/detach lifecycle; `get_ldev`/`put_ldev` reference counting |

**Next:** Week 3 begins — Day 15: The Netlink interface (`drbd_nl.c`): how `drbdadm` commands reach the kernel.
