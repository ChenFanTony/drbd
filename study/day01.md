# Day 1 — DRBD Overview, Repo Layout & I/O Stack Position

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_int.h`, `drbd/Makefile`, `drbd/drbd-headers/drbd_protocol.h`, top-level `Makefile`

---

## 1. What is DRBD?

DRBD (Distributed Replicated Block Device) is a **Linux kernel module** that exposes a virtual block device (`/dev/drbdN`) whose writes are transparently replicated to one or more peer nodes over TCP/IP or RDMA. It is "network RAID-1" at the block layer.

Key architectural decisions that shape every line of code you will read:

| Decision | Implication |
|---|---|
| Lives inside the kernel as a block driver | Must never sleep in IRQ context; all blocking in kernel threads |
| Intercepts `struct bio` objects | Sees raw sector ranges, not files or filesystem structures |
| Per-connection kernel threads | One sender + one receiver thread per peer node |
| Custom binary protocol | Packet dispatch table in `drbd_receiver.c` |
| Metadata on-disk | Activity log + bitmap stored adjacent to or separate from data |

---

## 2. Repo Directory Layout

Clone and explore:
```bash
git clone https://github.com/ChenFanTony/drbd.git -b drbd-9.2
find drbd/ -name '*.c' | sort
```

Expected file list (these are ALL the C translation units that compile into `drbd.ko`):

```
drbd/drbd_actlog.c          ← activity log (crash-safe write tracking)
drbd/drbd_bitmap.c          ← out-of-sync bitmap
drbd/drbd_dax_pmem.c        ← persistent memory metadata support (DAX)
drbd/drbd_debugfs.c         ← /sys/kernel/debug/drbd/ entries
drbd/drbd_interval.c        ← augmented red-black tree for I/O overlap detection
drbd/drbd_kref_debug.c      ← optional refcount-leak debugging
drbd/drbd_legacy_84.c       ← DRBD 8.4 backward-compat (mixed-version clusters)
drbd/drbd_main.c            ← module init/exit, core helpers, memory pools
drbd/drbd_nl.c              ← Netlink/Generic Netlink interface to drbdadm (largest, ~8000 lines)
drbd/drbd_nla.c             ← Netlink attribute helpers
drbd/drbd_proc.c            ← /proc/drbd (now just a stub; details in debugfs)
drbd/drbd_receiver.c        ← network receiver thread + packet dispatch (largest, ~11500 lines)
drbd/drbd_req.c             ← application I/O request processing
drbd/drbd_sender.c          ← sender thread + drbd_worker() + many w_* callbacks
drbd/drbd_state.c           ← state machine (role/disk/replication states)
drbd/drbd_transport.c       ← transport abstraction layer
drbd/drbd_transport_tcp.c   ← TCP socket transport implementation
drbd/drbd_transport_lb-tcp.c← Load-balancing TCP (multiple connections per peer)
drbd/drbd_transport_rdma.c  ← RDMA transport (in-tree)
drbd/drbd_transport_template.c ← skeleton for new transports
drbd/kref_debug.c           ← kref tracing infrastructure
```

> **Important:** `drbd_strings.c`, `drbd_protocol.h`, and `lru_cache.c` are NOT in `drbd/`. They live in submodules / compat dirs:
> - `drbd/drbd-headers/drbd_strings.c` (submodule — run `git submodule update --init`)
> - `drbd/drbd-headers/drbd_protocol.h`
> - `drbd/drbd-kernel-compat/lru_cache.c` (compat copy of upstream `lib/lru_cache.c`)
> - `drbd_buildtag.c` is auto-generated at build time
```

Header files of primary interest (note: most are in the `drbd-headers` submodule):
```
drbd/drbd_int.h                       ← ALL internal structs (drbd_device, drbd_connection, …)
drbd/drbd_req.h                       ← drbd_request struct + rq_state bitmask definitions
drbd/drbd_interval.h                  ← drbd_interval struct
drbd/drbd_state.h                     ← state machine helpers
drbd/drbd-headers/drbd_protocol.h     ← wire protocol: packet types + on-wire structs (submodule)
drbd/drbd-headers/drbd_transport.h    ← transport abstraction API (submodule)
drbd/drbd-headers/linux/drbd.h        ← UAPI header (shared with userspace drbdadm)
drbd/drbd-headers/linux/drbd_genl.h   ← Netlink schema definitions
drbd/drbd-headers/linux/drbd_limits.h ← tunable parameter bounds
```

> **Always run `git submodule update --init` after cloning** or these headers won't exist.

---

## 3. DRBD's Position in the Linux I/O Stack

```
┌──────────────────────────────────────────────┐
│  Application (PostgreSQL, ext4 mount, etc.)  │
└────────────────────┬─────────────────────────┘
                     │  read()/write() syscall
┌────────────────────▼─────────────────────────┐
│         VFS (Virtual File System)            │
└────────────────────┬─────────────────────────┘
                     │  page cache writeback → struct bio
┌────────────────────▼─────────────────────────┐
│      Block Layer  (bio_submit, elevators)    │
└────────────────────┬─────────────────────────┘
                     │  bio handed to make_request_fn
┌────────────────────▼─────────────────────────┐
│   DRBD virtual block device (/dev/drbd0)     │  ← YOU ARE STUDYING THIS
│   __drbd_make_request() in drbd_req.c          │
└──────────┬─────────────────────┬─────────────┘
           │ local path          │ network path
┌──────────▼──────────┐  ┌───────▼──────────────┐
│  Real block device  │  │  TCP/IP or RDMA       │
│  (SSD/HDD/LVM)      │  │  to peer node(s)      │
└─────────────────────┘  └──────────────────────┘
```

The key insight: DRBD is registered as a `gendisk` with its own `make_request` function. The block layer calls this function for every bio. DRBD then fans the bio out to both paths.

---

## 4. Source Code Trace: How DRBD Registers as a Block Device

Open `drbd/drbd_main.c`. The full module init sequence is:

```c
// drbd/drbd_main.c  ~line 3500 (search: "static int __init drbd_init")
static int __init drbd_init(void)
{
    // Step 1: Register block device major number
    err = register_blkdev(DRBD_MAJOR, "drbd");
    // DRBD_MAJOR = 147, defined in linux/drbd.h

    // Step 2: Create slab caches and memory pools
    err = drbd_create_mempools();
    // allocates: drbd_request_mempool, drbd_ee_mempool,
    //            drbd_buffer_page_pool (page pool), several kmem_caches

    // Step 3: Register Generic Netlink family
    err = drbd_genl_register();
    // registers "drbd" genl family so drbdadm can talk to the kernel

    // Step 4: Create /sys/kernel/debug/drbd
    drbd_debugfs_init();

    // Step 5: Register reboot notifier (clean shutdown)
    register_reboot_notifier(&drbd_notifier);
}
module_init(drbd_init);
```

**Trace exercise:** For each of the 5 steps above, use `grep -n` to find the function definition and read its body:
```bash
grep -n "drbd_create_mempools\b" drbd/drbd_main.c
grep -n "drbd_genl_register\b"   drbd/drbd_nl.c
grep -n "drbd_debugfs_init\b"    drbd/drbd_debugfs.c
```

---

## 5. Source Code Trace: Per-Device Block Device Registration

When a user runs `drbdadm up r0`, a new `struct gendisk` is allocated per volume. Find this in `drbd_main.c`:

```bash
grep -n "alloc_disk\|blk_alloc_disk\|add_disk" drbd/drbd_main.c
```

The sequence inside `drbd_create_device()` (search that function name):

```
drbd_create_device(resource, vnr, disk_conf)
    ├── kzalloc(sizeof(struct drbd_device), ...)
    ├── device->vdisk = blk_alloc_disk(NUMA_NO_NODE)
    │       creates gendisk + request_queue together
    ├── device->vdisk->major       = DRBD_MAJOR
    ├── device->vdisk->first_minor = minor
    ├── device->vdisk->fops        = &drbd_ops
    │       drbd_ops.submit_bio    = drbd_submit_bio  ← THE entry point (modern kernels)
    │       drbd_ops.open          = drbd_open
    │       drbd_ops.release       = drbd_release
    ├── sprintf(device->vdisk->disk_name, "drbd%d", minor)
    └── add_disk(device->vdisk)    ← visible in /dev/
```

```bash
grep -n "drbd_create_device\b"  drbd/drbd_main.c
grep -n "drbd_ops\b"            drbd/drbd_main.c
grep -n "submit_bio\s*="        drbd/drbd_main.c
```

---

## 6. Source Code Trace: Memory Pools (`drbd_create_mempools`)

```bash
grep -n -A 80 "^static int drbd_create_mempools" drbd/drbd_main.c
```

What you should find:

```c
static int drbd_create_mempools(void)
{
    // slab cache for struct drbd_request
    drbd_request_cache = kmem_cache_create(
        "drbd_req", sizeof(struct drbd_request), 0, 0, NULL);

    // slab cache for struct drbd_peer_request (epoch entries)
    drbd_ee_cache = kmem_cache_create(
        "drbd_ee", sizeof(struct drbd_peer_request), 0, 0, NULL);

    // mempool backed by drbd_request_cache (NOT a pointer — declared as mempool_t)
    mempool_init_slab_pool(&drbd_request_mempool, number, drbd_request_cache);

    // mempool backed by drbd_ee_cache
    mempool_init_slab_pool(&drbd_ee_mempool, number, drbd_ee_cache);

    // page pools — TWO of them in DRBD 9.2:
    mempool_init_page_pool(&drbd_md_io_page_pool, DRBD_MIN_POOL_PAGES, 0);
    // ↑ pages used for synchronous metadata I/O

    mempool_init_page_pool(&drbd_buffer_page_pool, number, 0);
    // ↑ pages used for receive buffers (formerly drbd_buffer_page_pool)
}
```

**Why mempools?** `mempool_alloc()` never fails under memory pressure — it falls back to the pre-allocated reserve. This is critical in the I/O path where allocation failure would mean data loss.

```bash
# Find DRBD_MIN_POOL_PAGES value
grep -rn "DRBD_MIN_POOL_PAGES\b" drbd/
```

> **Note:** The receive page pool is **`drbd_buffer_page_pool`** in DRBD 9.2 (older docs and earlier sources called it `drbd_buffer_page_pool`). Always check the actual symbol with `grep`.

---

## 7. The `block_device_operations` vtable

```bash
grep -n -A 15 "drbd_ops\s*=" drbd/drbd_main.c
# or
grep -n "struct block_device_operations" drbd/drbd_main.c
```

You should see:
```c
static const struct block_device_operations drbd_ops = {
    .owner       = THIS_MODULE,
    .submit_bio  = drbd_submit_bio,    // ← THE entry point in modern kernels
    .open        = drbd_open,
    .release     = drbd_release,
};
// Note: in modern kernels (5.9+), the block layer dispatches each bio
// directly to the .submit_bio handler. The older .make_request mechanism
// (registered via blk_queue_make_request) is gone.
```

---

## 8. Hands-On Exercises (3–4 hours)

### Exercise 1 (30 min): Build and load the module
```bash
# Install kernel headers
sudo apt install linux-headers-$(uname -r) build-essential

# Build
cd drbd && make

# Check produced files
ls -lh drbd.ko drbd_transport_tcp.ko

# Load (on a test VM only!)
sudo insmod drbd.ko
sudo insmod drbd_transport_tcp.ko
cat /proc/drbd   # should show "v: 9.2.x"
dmesg | tail -20
```

### Exercise 2 (30 min): Explore module parameters
```bash
grep -n "module_param\b" drbd/drbd_main.c
# For each param: what type, what default, what does it control?
```

### Exercise 3 (45 min): Read the full `drbd_init()` function
Open `drbd_main.c`, find `drbd_init`, and for every function call inside it:
1. Find the callee's definition with `grep -n "^static\|^int\|^void" drbd/*.c`
2. Write one sentence describing what it does

### Exercise 4 (45 min): Map the cleanup path
Find `drbd_cleanup()`. Verify that it exactly reverses every step of `drbd_init()`. Draw the init vs cleanup as a paired table.

### Exercise 5 (60 min): Trace a minor number allocation
When `/dev/drbd0` is created, what minor number does it get?
```bash
grep -n "minor\|MINORMASK\|idr_alloc\|idr_find" drbd/drbd_main.c drbd/drbd_nl.c | head -30
```
Find `drbd_new_minor()` or equivalent. How does DRBD manage the minor number namespace?

### Exercise 6 (30 min): Read `drbd/drbd-headers/linux/drbd.h` (the UAPI header)
This is the contract between kernel and userspace. Note every `enum`, `struct`, and constant. You will encounter all of them repeatedly in coming days.

---

## Key Terms Glossary

| Term | Definition |
|---|---|
| `bio` | Block I/O descriptor — the unit of I/O in the Linux block layer |
| `gendisk` | Kernel structure representing a disk visible in `/dev/` |
| `mempool` | Pre-allocated reserve pool; allocation never fails |
| `kmem_cache` | Slab allocator cache for fixed-size objects |
| `DRBD_MAJOR` | Linux major device number 147, assigned to DRBD |
| `make_request_fn` | Function pointer called by block layer for each bio |
| `genl` | Generic Netlink — kernel/userspace communication subsystem |

---

## Summary

DRBD's codebase is ~18 C files that together form `drbd.ko`. The module registers major number 147, creates per-object slab caches and mempools, hooks into Generic Netlink for `drbdadm` communication, and exposes DebugFS entries. Per-device, a `gendisk` is allocated with `__drbd_make_request` as the block layer entry point. Tomorrow you trace that entry point from `drbd_main.c`'s init down to the first per-device structures.

**Next:** Day 2 — The four core data structures in depth.
