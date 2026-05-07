# Day 2 — Core Data Structures: `drbd_resource`, `drbd_connection`, `drbd_device`, `drbd_peer_device`

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_int.h`, `drbd/drbd_main.c`, kernel docs: https://www.kernel.org/doc/html/latest/admin-guide/blockdev/drbd/data-structure-v9.html

---

## 1. The Object Matrix — Conceptual Model

DRBD 9 uses four types of objects arranged in a 2-D matrix:

```
                     volume 0            volume 1          volume N
                  ┌──────────────┬──────────────────┬──────────────┐
  drbd_resource   │  drbd_device │    drbd_device   │  drbd_device │
                  ├──────────────┼──────────────────┼──────────────┤
  connection→A    │peer_device   │    peer_device   │  peer_device │
                  ├──────────────┼──────────────────┼──────────────┤
  connection→B    │peer_device   │    peer_device   │  peer_device │
                  └──────────────┴──────────────────┴──────────────┘

Horizontal access: resource->devices (IDR keyed by volume number)
                   connection->peer_devices (IDR keyed by volume number)
Vertical access:   resource->connections (linked list)
                   device->peer_devices (linked list, via peer_device->peer_devices)
```

Every object is **reference-counted** with `struct kref`. Destruction order must always be:
```
peer_device → then connection or device → then resource
```

---

## 2. `struct drbd_resource` — Deep Dive

**Location:** `drbd/drbd_int.h`

```bash
grep -n "struct drbd_resource {" drbd/drbd_int.h
# Then read every field until the closing '};'
```

Key fields to understand deeply:

```c
struct drbd_resource {
    char *name;
    // ── Object graph ──────────────────────────────────
    struct idr devices;           // volume_nr (int) → struct drbd_device*
                                  // IDR = integer-keyed radix tree; O(log N) lookup
    struct list_head connections; // list of drbd_connection objects
    struct list_head resources;   // node in global drbd_resources list
    // ── Locking ───────────────────────────────────────
    struct mutex conf_update;     // serialises config changes (drbdadm up/down)
    struct mutex adm_mutex;       // serialises drbdadm commands
    spinlock_t req_lock;          // protects: transfer_log, req lists, state changes
                                  // MUST be held when touching any rq_state bit
    // ── Write ordering ────────────────────────────────
    struct list_head transfer_log; // ordered list of in-flight drbd_requests
                                   // order = dagtag (data generation tag) sequence
    u64 dagtag_sector;            // monotonically increasing write sequence
    // ── Role & quorum ─────────────────────────────────
    enum drbd_role role[2];       // [NOW] and [NEW] — two-phase state change
    bool susp;                    // I/O suspended (fencing, quorum loss)
    bool susp_nod;                // suspended, no data
    bool susp_fen;                // suspended, fencing
    // ── Worker ────────────────────────────────────────
    struct drbd_thread worker;    // resource-level worker thread
    struct drbd_work_queue work;  // work queue for resource-level tasks
    // ── State change machinery ────────────────────────
    wait_queue_head_t state_wait; // waiters for state change completion
    enum chg_state_flags state_change_flags;
};
```

**Trace exercise:**
```bash
# Find where drbd_resource is allocated
grep -n "kzalloc.*drbd_resource\|drbd_resource.*kzalloc" drbd/drbd_main.c

# Find where it is freed
grep -n "drbd_destroy_resource\|kfree.*resource" drbd/drbd_main.c

# Find the global resource list
grep -n "drbd_resources\b" drbd/drbd_main.c drbd/drbd_nl.c | head -10
```

---

## 3. `struct drbd_connection` — Deep Dive

```bash
grep -n "struct drbd_connection {" drbd/drbd_int.h
```

```c
struct drbd_connection {
    struct drbd_resource *resource;    // back-pointer to owning resource
    struct list_head connections;      // node in resource->connections
    struct kref kref;                  // reference count
    struct idr peer_devices;           // volume_nr → drbd_peer_device*

    // ── Threads ───────────────────────────────────────
    struct drbd_thread receiver;       // reads from socket, dispatches packets
    struct drbd_thread sender;         // sends P_DATA, P_BARRIER, etc.
    struct drbd_thread worker;         // background tasks for this connection

    // ── Transport ─────────────────────────────────────
    struct drbd_transport *transport;  // opaque transport handle
                                       // points to drbd_tcp_transport or RDMA
    // ── Connection state ──────────────────────────────
    enum drbd_conn_state cstate[2];    // [NOW] and [NEW]
    enum drbd_role peer_role[2];       // peer's advertised role

    // ── Flow control ──────────────────────────────────
    atomic_t ap_in_flight;            // application writes pending ACK from this peer
    atomic_t rs_in_flight;            // resync writes pending ACK from this peer
    unsigned int ko_count;            // "knock-out" counter; too many → disconnect
    // ── Epoch / write ordering ────────────────────────
    struct list_head current_epoch;   // current write epoch
    spinlock_t epoch_lock;
    unsigned int epochs;              // number of open epochs
    // ── Bitmap exchange ───────────────────────────────
    wait_queue_head_t ee_wait;        // wait for epoch entries to drain
    struct list_head active_ee;       // epoch entries submitted to local disk
    struct list_head sync_ee;         // resync epoch entries
    struct list_head done_ee;         // completed, waiting for ACK send
    struct list_head read_ee;         // peer read requests
    spinlock_t active_ee_lock;        // protects the ee lists
    // ── Authentication ────────────────────────────────
    struct crypto_shash *cram_hmac_tfm; // HMAC for challenge-response auth
    // ── Timestamps for timeout detection ─────────────
    unsigned long last_received;       // jiffies of last received packet
};
```

**Trace exercise:**
```bash
# Where is drbd_connection allocated?
grep -n "drbd_create_connection\|kzalloc.*drbd_connection" drbd/drbd_main.c

# Where are the three threads started?
grep -n "drbd_thread_start\b" drbd/drbd_receiver.c drbd/drbd_main.c drbd/drbd_nl.c

# What triggers connection teardown?
grep -n "drbd_destroy_connection\b" drbd/drbd_main.c drbd/drbd_receiver.c
```

---

## 4. `struct drbd_device` — Deep Dive

```bash
grep -n "struct drbd_device {" drbd/drbd_int.h
```

```c
struct drbd_device {
    struct drbd_resource *resource;   // owning resource
    int vnr;                          // volume number (0-based)
    struct gendisk *vdisk;            // the kernel block device
    struct request_queue *rq_queue;   // associated request queue
    struct block_device *this_bdev;   // self-reference (for bio cloning)

    // ── Local disk ────────────────────────────────────
    struct drbd_backing_dev *ldev;    // NULL if diskless
    // ldev contains:
    //   struct block_device *backing_bdev  (actual storage)
    //   struct block_device *md_bdev       (metadata device, may == backing_bdev)
    //   struct drbd_md md                  (in-memory metadata superblock)

    // ── Disk state ────────────────────────────────────
    enum drbd_disk_state disk_state[2];   // [NOW] and [NEW]

    // ── I/O tracking ──────────────────────────────────
    struct rb_root read_requests;     // interval tree of pending read requests
    struct rb_root write_requests;    // interval tree of pending write requests
    wait_queue_head_t misc_wait;      // waiters for request completion

    // ── Activity log ──────────────────────────────────
    struct lru_cache *act_log;        // in-memory activity log (LRU cache)
    unsigned int al_tr_number;        // current AL transaction number
    int al_writ_cnt;                  // AL writes in this session

    // ── Bitmap ────────────────────────────────────────
    struct drbd_bitmap *bitmap;       // out-of-sync bitmap (one slot per peer)

    // ── Resync ────────────────────────────────────────
    atomic_t local_cnt;              // local I/O reference count
    // (disk cannot be detached while local_cnt > 0)

    // ── Peer device list ──────────────────────────────
    struct list_head peer_devices;    // list of drbd_peer_device for this volume

    // ── Miscellaneous ─────────────────────────────────
    unsigned long flags;              // UNPLUG_QUEUED, SUSPEND_IO, AL_SUSPENDED…
    struct drbd_work md_sync_work;    // deferred metadata flush
};
```

**Trace exercise:**
```bash
# drbd_backing_dev struct — what metadata does it hold?
grep -n "struct drbd_backing_dev {" drbd/drbd_int.h

# struct drbd_md — the in-memory metadata superblock
grep -n "struct drbd_md {" drbd/drbd_int.h

# When is ldev set / cleared?
grep -n "device->ldev\s*=" drbd/drbd_nl.c drbd/drbd_main.c | head -20
```

---

## 5. `struct drbd_peer_device` — Deep Dive

```bash
grep -n "struct drbd_peer_device {" drbd/drbd_int.h
```

```c
struct drbd_peer_device {
    struct list_head peer_devices;      // node in device->peer_devices
    struct drbd_device *device;         // which volume
    struct drbd_connection *connection; // which peer

    // ── Replication state ─────────────────────────────
    enum drbd_repl_state repl_state[2]; // [NOW] and [NEW]
    enum drbd_disk_state disk_state[2]; // peer's disk state as reported

    // ── Resync ────────────────────────────────────────
    unsigned long rs_total;            // total OOS sectors at resync start
    unsigned long rs_left;             // OOS sectors remaining
    unsigned long rs_failed;           // sectors that failed to resync
    unsigned long rs_paused;           // time spent paused (jiffies)
    unsigned long bm_resync_fo;        // find-offset cursor into bitmap
    struct drbd_work resync_work;      // work item to kick resync bursts
    struct timer_list resync_timer;    // periodic resync trigger

    // ── UUID management ───────────────────────────────
    u64 current_uuid;
    u64 bitmap_uuid;
    u64 history_uuids[HISTORY_UUIDS];
    u64 dirty_bits;

    // ── Congestion / Ahead-Behind ─────────────────────
    atomic_t unacked_cnt;             // unacknowledged resync writes
    unsigned int c_sync_rate;         // current resync rate (sectors/sec)

    // ── Online verify ─────────────────────────────────
    sector_t ov_left;                 // sectors remaining in verify scan
    struct digest_info *ov_l_digest;  // local digest buffer
};
```

**Trace exercise:**
```bash
# Where is peer_device allocated?
grep -n "drbd_new_peer_device\|kzalloc.*peer_device" drbd/drbd_main.c

# How does a peer_device link to both its device and connection?
grep -n "peer_device->device\s*=\|peer_device->connection\s*=" drbd/drbd_main.c

# What happens to peer_device when a connection drops?
grep -n "drbd_peer_device_cleanup\|drbd_destroy_peer_device" drbd/drbd_main.c
```

---

## 6. Reference Counting — Lifetime Rules

All four structs use `struct kref`. The rules are strict:

```c
// Taking a reference — safe to use the object
kref_get(&device->kref);

// Releasing — may trigger destructor if refcount hits 0
kref_put(&device->kref, drbd_destroy_device);

// Helper macro pattern used in DRBD:
#define kref_get_unless_zero(kref) ...
```

**Object lifetimes:**

```
drbd_resource  created in: drbd_create_resource()       [drbd_main.c]
               destroyed:  drbd_destroy_resource()      [drbd_main.c]

drbd_connection created:   drbd_create_connection()     [drbd_main.c]
               destroyed:  drbd_destroy_connection()    [drbd_main.c]
               kref held by: resource (1) + caller (1) + threads (1 each)

drbd_device    created:    drbd_create_device()         [drbd_main.c]
               destroyed:  drbd_destroy_device()        [drbd_main.c]
               kref held by: resource (1) + gendisk (1)

drbd_peer_device created:  drbd_new_peer_device()       [drbd_main.c]
               destroyed:  when connection OR device destroyed
               NOT separately reference-counted in all versions
```

```bash
grep -n "kref_put\|kref_get\|kref_init" drbd/drbd_main.c | head -30
```

---

## 7. Object Traversal Macros

These macros are used hundreds of times — memorise them:

```c
// Iterate all devices in a resource (IDR walk)
idr_for_each_entry(&resource->devices, device, vnr) {
    // device: struct drbd_device*
    // vnr:    int volume number
}

// Iterate all connections of a resource (list walk)
for_each_connection(connection, resource) {
    // expands to: list_for_each_entry(connection,
    //                 &resource->connections, connections)
}

// Iterate all peer_devices of a connection (IDR walk)
idr_for_each_entry(&connection->peer_devices, peer_device, vnr) { }

// Iterate all peer_devices of a device (list walk)
for_each_peer_device(peer_device, device) {
    // expands to: list_for_each_entry(peer_device,
    //                 &device->peer_devices, peer_devices)
}

// RCU-safe variants (used without locks)
for_each_connection_rcu(connection, resource) { }
for_each_peer_device_rcu(peer_device, device) { }
```

```bash
grep -n "#define for_each_connection\|#define for_each_peer_device\|#define idr_for_each" \
    drbd/drbd_int.h drbd/drbd_main.c
```

---

## 8. The Two-Phase State Field Pattern

Every state field in DRBD uses a `[2]` array:

```c
enum drbd_role role[2];          // role[NOW] and role[NEW]
enum drbd_disk_state disk_state[2];
enum drbd_repl_state repl_state[2];
```

`NOW = 0`, `NEW = 1`. During a state change:
1. `NEW` is set to the proposed new value
2. Validation runs against `NEW`
3. If valid, `NOW` is updated to match `NEW`

```bash
grep -n "#define NOW\|#define NEW\b" drbd/drbd_state.h drbd/drbd_int.h
grep -n "role\[NOW\]\|role\[NEW\]" drbd/drbd_state.c | head -20
```

---

## 9. Hands-On Exercises (3–4 hours)

### Exercise 1 (40 min): Draw the full object graph on paper
For a resource with 2 volumes and 2 peer connections, draw every `struct` instance and every pointer between them. Include the IDR trees and linked lists as arrows.

### Exercise 2 (40 min): Trace `drbd_create_device()` completely
```bash
grep -n -A 120 "^static int drbd_create_device\b" drbd/drbd_main.c
```
For every field that gets initialised, write what it does and why.

### Exercise 3 (30 min): Count and categorise all fields in `drbd_device`
Open `drbd_int.h`, find `struct drbd_device {}`. Group its fields into categories:
- Object graph pointers (5–8 fields)
- State fields (3–5)
- I/O tracking (5–8)
- Locking primitives (3–5)
- Statistics / counters (5+)

### Exercise 4 (40 min): Trace kref lifecycle for `drbd_connection`
```bash
grep -n "kref_get\|kref_put\|kref_init" drbd/drbd_main.c | grep -i connection
```
Draw a timeline: when is the refcount incremented, decremented, and what function is called at refcount==0?

### Exercise 5 (50 min): Understand IDR vs linked list choice
DRBD uses IDR for `devices` and `peer_devices` (volume number → object) but linked lists for `connections`. Why?
- Look at how `idr_find(&resource->devices, vnr)` is used vs `for_each_connection()`
- When would you need O(1) lookup by volume number?
- When is iterating all connections more natural?

### Exercise 6 (30 min): Explore `struct drbd_backing_dev`
```bash
grep -n "struct drbd_backing_dev {" drbd/drbd_int.h
grep -n "struct drbd_md {" drbd/drbd_int.h
```
What is the difference between `backing_bdev` and `md_bdev`? When are they the same device?

---

## Summary

The four DRBD objects form a matrix: resources own devices (by volume) and connections (by peer). Peer-devices sit at each (device, connection) intersection. All objects are reference-counted with `kref`. State fields use a `[NOW/NEW]` two-phase pattern for atomic transitions. Traversal macros abstract IDR-walk (volume lookup) and list-walk (connection iteration). Every subsequent day will navigate this graph — internalising it now is the key investment.

**Next:** Day 3 — The state machine: all three state dimensions, transition validation, two-phase commit.
