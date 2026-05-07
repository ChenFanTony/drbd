# Day 19 — DebugFS, `/proc/drbd` & Runtime Diagnostics

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_debugfs.c`, `drbd/drbd_proc.c`, `drbd/drbd_main.c`

---

## 1. Two Diagnostic Interfaces

| Interface | Path | Purpose | Format |
|---|---|---|---|
| `/proc/drbd` | Procfs | Summary status for all devices | Human-readable text |
| DebugFS | `/sys/kernel/debug/drbd/` | Detailed per-object diagnostics | Per-file, structured text |

Both are read-only from userspace. `drbdsetup status` uses Netlink (Day 15), not these files.

---

## 2. `/proc/drbd` — `drbd_proc.c`

```bash
grep -n -A 100 "^static int drbd_seq_show\b" drbd/drbd_proc.c
```

Registration:
```bash
grep -n "proc_create\|drbd_proc_dir\|drbd_proc\b" drbd/drbd_main.c drbd/drbd_proc.c | head -10
```

```c
// drbd_main.c: drbd_init()
drbd_proc = proc_create("drbd", S_IRUGO, NULL, &drbd_proc_fops);
```

The output of `cat /proc/drbd`:

```
version: 9.2.x (api:1/proto:86-121)

 0: cs:Connected ro:Primary/Secondary ds:UpToDate/UpToDate C r-----
    ns:14523 nr:0 dw:567290 dr:15244 al:1 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0
```

Each field traced to its source:

```bash
grep -n "drbd_seq_printf_stats\|cs:\|ro:\|ds:\|ns:\|dw:\|dr:\|al:\|pe:\|ua:\|oos:" \
    drbd/drbd_proc.c | head -30
```

| Field | Meaning | Source in code |
|---|---|---|
| `cs:Connected` | Connection state | `connection->cstate[NOW]` |
| `ro:Primary/Secondary` | Local/peer role | `resource->role[NOW]` / `connection->peer_role[NOW]` |
| `ds:UpToDate/UpToDate` | Local/peer disk state | `device->disk_state[NOW]` / `peer_device->disk_state[NOW]` |
| `C` | Replication protocol | `net_conf->wire_protocol` (A/B/C) |
| `ns` | Network sent (KiB) | `device->send_cnt` |
| `nr` | Network received (KiB) | `device->recv_cnt` |
| `dw` | Disk writes (KiB) | `device->writ_cnt` |
| `dr` | Disk reads (KiB) | `device->read_cnt` |
| `al` | AL writes | `device->al_writ_cnt` |
| `bm` | Bitmap I/O | `device->bm_io_time` |
| `lo` | Local outstanding (in-flight) | `atomic_read(&device->local_cnt)` |
| `pe` | Peer requests (secondary in-flight) | `atomic_read(&connection->ap_in_flight)` |
| `ua` | Unacked (requests sent but not acked) | Similar to pe |
| `ap` | Application pending | `atomic_read(&device->ap_bio_cnt)` |
| `ep` | Epochs | `connection->epochs` |
| `wo` | Write ordering mode | `b`=barrier `f`=flush `d`=drain `n`=none |
| `oos` | Out-of-sync KiB | `drbd_bm_total_weight(peer_device) << (BM_BLOCK_SHIFT-10)` |

```bash
grep -n "al_writ_cnt\|ap_bio_cnt\|ap_in_flight\|send_cnt\|recv_cnt\|writ_cnt\|read_cnt" \
    drbd/drbd_int.h | head -20
```

---

## 3. The DebugFS Tree

```bash
grep -n "drbd_debugfs_init\b\|debugfs_create_dir\|debugfs_create_file" drbd/drbd_debugfs.c | head -30
```

```c
// drbd_debugfs.c: drbd_debugfs_init()
drbd_debugfs_root = debugfs_create_dir("drbd", NULL);
// Creates: /sys/kernel/debug/drbd/

// Per-resource directory created in drbd_debugfs_resource_add():
debugfs_create_dir(resource->name, drbd_debugfs_root);
// e.g.: /sys/kernel/debug/drbd/r0/

// Per-resource files:
debugfs_create_file("state_twopc", 0444, resource_dir, resource,
                     &drbd_state_twopc_fops);
debugfs_create_file("members", 0444, resource_dir, resource,
                     &drbd_members_fops);
debugfs_create_file("hook_log", 0444, resource_dir, resource,
                     &drbd_hook_log_fops);

// Per-volume subdirectory:
// /sys/kernel/debug/drbd/r0/volumes/0/
debugfs_create_file("oldest_requests", 0444, vol_dir, device,
                     &drbd_oldest_requests_fops);
debugfs_create_file("act_log",         0444, vol_dir, device,
                     &drbd_act_log_fops);
debugfs_create_file("data_gen_id",     0444, vol_dir, device,
                     &drbd_data_gen_id_fops);

// Per-connection subdirectory:
// /sys/kernel/debug/drbd/r0/connections/peer-hostname/
debugfs_create_file("transport",       0444, conn_dir, connection,
                     &drbd_transport_fops);
debugfs_create_file("send_buf",        0444, conn_dir, connection,
                     &drbd_send_buf_fops);

// Per-peer-device:
// /sys/kernel/debug/drbd/r0/connections/peer-hostname/volumes/0/
debugfs_create_file("resync_extents",  0444, peer_dev_dir, peer_device,
                     &drbd_resync_extents_fops);
debugfs_create_file("proc_drbd",       0444, peer_dev_dir, peer_device,
                     &drbd_proc_drbd_fops);
```

---

## 4. Key DebugFS Files — Source Traces

### `oldest_requests` — In-Flight Request Diagnostics

```bash
grep -n -A 80 "^static int drbd_oldest_requests_show\b" drbd/drbd_debugfs.c
```

Shows the oldest in-flight read and write requests (useful for detecting stuck I/O):

```c
static int drbd_oldest_requests_show(struct seq_file *m, void *ignored)
{
    struct drbd_device *device = m->private;
    struct drbd_request *r1, *r2;
    ktime_t now = ktime_get();

    spin_lock_irq(&device->resource->req_lock);

    // Find oldest pending read
    r1 = find_oldest_request_in_tree(&device->read_requests);
    // Find oldest pending write
    r2 = find_oldest_request_in_tree(&device->write_requests);

    if (r1) {
        seq_printf(m, "oldest read: sector %llu, size %u, "
                   "age %lldms, rq_state: %08lx\n",
                   (u64)r1->i.sector, r1->i.size,
                   ktime_ms_delta(now, r1->start_jif),
                   r1->rq_state[0]);
    }
    // similarly for r2
    spin_unlock_irq(&device->resource->req_lock);
    return 0;
}
```

**Diagnostic use:** If DRBD appears stuck, `cat oldest_requests` shows whether there are I/O requests that have been waiting for seconds/minutes and what `rq_state` bits are still set (which acks are pending).

### `act_log` — Activity Log Dump

```bash
grep -n -A 40 "^static int drbd_act_log_show\b\|drbd_act_log_fops\b" drbd/drbd_debugfs.c
```

```c
// Shows: lru_cache stats + per-slot dump
static int drbd_act_log_show(struct seq_file *m, void *ignored)
{
    struct drbd_device *device = m->private;
    struct lru_cache *lc = device->act_log;

    // Print stats
    lc_seq_printf_stats(m, lc);
    // Output: "act_log: used:23/1024 hits:2341234 misses:12345 starving:0 dirty:0"

    // Print per-slot detail
    lc_seq_dump_details(m, lc, "e", blah_printer);
    // Output: per slot: "index:0 number:1234" (slot 0 maps to AL extent 1234)
}
```

### `resync_extents` — OOS Bitmap Summary

```bash
grep -n -A 60 "^static int drbd_resync_extents_show\b" drbd/drbd_debugfs.c
```

Shows the first N set bits in the OOS bitmap and their sector ranges — useful for understanding what a resync is doing.

### `transport` — Transport-Level Stats

```bash
grep -n "drbd_transport_fops\|transport.*debugfs_show\|dtt_debugfs_show\b" \
    drbd/drbd_debugfs.c drbd/drbd_transport_tcp.c | head -10
grep -n -A 40 "^static void dtt_debugfs_show\b" drbd/drbd_transport_tcp.c
```

```c
static void dtt_debugfs_show(struct drbd_transport *transport,
                               struct seq_file *m)
{
    // Show per-socket stats: bytes sent/received, buffer sizes, timeouts
    for (int i = 0; i < 2; i++) {
        struct socket *sock = tcp_transport->stream[i];
        if (sock) {
            struct tcp_sock *tp = tcp_sk(sock->sk);
            seq_printf(m, "stream[%d]: sndbuf=%d rcvbuf=%d "
                       "snd_cwnd=%u rtt=%ums\n",
                       i,
                       sock->sk->sk_sndbuf,
                       sock->sk->sk_rcvbuf,
                       tp->snd_cwnd,
                       tp->srtt_us / (USEC_PER_MSEC * 8));
        }
    }
}
```

---

## 5. Counter Architecture — How Stats Are Updated

```bash
grep -n "device->send_cnt\|device->writ_cnt\|device->read_cnt\|atomic_add\|device->al_writ_cnt" \
    drbd/drbd_req.c drbd/drbd_actlog.c drbd/drbd_sender.c | head -20
```

Most counters are simple `unsigned long` fields incremented without locks (they may be slightly inaccurate under concurrent access, but that's acceptable for diagnostic counters):

```c
// drbd_req.c: after a write completes locally:
device->writ_cnt += req->i.size >> 9;   // in sectors

// drbd_sender.c: after sending P_DATA:
device->send_cnt += req->i.size >> 9;

// drbd_actlog.c: after each AL transaction:
device->al_writ_cnt++;
```

---

## 6. `drbd_debugfs_resource_add()` / `drbd_debugfs_resource_cleanup()` Lifecycle

```bash
grep -n "drbd_debugfs_resource_add\b\|drbd_debugfs_resource_cleanup\b\|drbd_debugfs_device_add\b" \
    drbd/drbd_debugfs.c drbd/drbd_main.c drbd/drbd_nl.c | head -15
```

DebugFS entries are created/removed in sync with object creation/destruction:

```c
// Object created:
drbd_create_resource() → drbd_debugfs_resource_add(resource)
drbd_create_device()   → drbd_debugfs_device_add(device)
drbd_create_connection() → drbd_debugfs_connection_add(connection)

// Object destroyed:
drbd_destroy_resource() → drbd_debugfs_resource_cleanup(resource)
// etc.
```

---

## 7. Practical Diagnostic Scenarios

### Scenario 1: DRBD I/O stalled — find stuck requests
```bash
cat /sys/kernel/debug/drbd/r0/volumes/0/oldest_requests
# If age > 30s: look at rq_state bits
# RQ_NET_PENDING set → waiting for ACK from peer
# RQ_LOCAL_PENDING set → waiting for local disk
```

### Scenario 2: Resync is slow — check rate controller
```bash
cat /sys/kernel/debug/drbd/r0/connections/peer1/volumes/0/proc_drbd
# Shows resync progress, rate, estimated finish time
grep -n "drbd_proc_drbd_show\b" drbd/drbd_debugfs.c
```

### Scenario 3: Activity log thrashing — check hit rate
```bash
cat /sys/kernel/debug/drbd/r0/volumes/0/act_log
# "misses" / (hits + misses) = miss rate
# High miss rate → al-extents too small, or workload too random
```

### Scenario 4: Network congestion — check TCP buffers
```bash
cat /sys/kernel/debug/drbd/r0/connections/peer1/transport
# snd_cwnd small → TCP congestion window constrained
# rtt high → network latency is adding to protocol latency
```

---

## 8. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Read `drbd_proc.c` completely
~300 lines. For every `seq_printf()` call, identify: what variable it reads, where that variable is updated in the codebase.

### Exercise 2 (45 min): Read `drbd_debugfs.c` completely
~600 lines. List every `debugfs_create_file()` call and its associated `show` function.

### Exercise 3 (40 min): Trace the `oldest_requests` data path
```bash
grep -n "start_jif\b" drbd/drbd_req.h drbd/drbd_req.c | head -10
```
- When is `req->start_jif` set?
- What is the unit (jiffies vs ktime)?
- How is the age computed in the debugfs show function?

### Exercise 4 (35 min): Add a new counter (mental exercise)
If you wanted to count the number of `P_DATA` packets sent per connection, where would you:
1. Declare the counter (which struct, which field name)?
2. Increment it (which function, which file)?
3. Expose it via DebugFS (which show function to modify)?

### Exercise 5 (30 min): Explore `data_gen_id`
```bash
grep -n "data_gen_id\|dagtag\b\|drbd_data_gen_id_show\b" drbd/drbd_debugfs.c drbd/drbd_int.h | head -15
```
What is the `dagtag` (data generation tag)? How does it differ from a UUID? When is it useful for diagnostics?

---

## Summary

`/proc/drbd` provides a one-line-per-device human-readable summary by reading counters from `drbd_device` and `drbd_connection` structs. The DebugFS tree under `/sys/kernel/debug/drbd/` provides deeper per-object diagnostics: `oldest_requests` shows stuck I/O with age and `rq_state`; `act_log` shows AL hit/miss statistics; `resync_extents` shows the bitmap cursor; `transport` shows TCP socket internals. All diagnostic counters are updated inline in the data path (no separate sampling). DebugFS entries are created and removed in lockstep with the kernel objects they describe.

**Next:** Day 20 — Online verify (`drbdadm verify`): the verify state machine, hash computation, mismatch handling, and `ov_left` tracking.
