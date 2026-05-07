# Day 30 — Final Review: Architecture Synthesis, Common Pitfalls & The Road Ahead

> **Estimated study time: 3–4 hours**
> **All files reviewed. No new code today — synthesis and consolidation.**

---

## 1. The Complete Architecture Map

Draw this from memory, then verify:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     DRBD 9.2 Architecture                           │
│                                                                     │
│  USERSPACE                                                          │
│  drbdadm ──netlink──────────────────────────────────────────────┐  │
│                                                                  │  │
│  KERNEL                                                          │  │
│  ┌──────────────────────────────────────────────────────────┐   │  │
│  │ Generic Netlink (drbd_nl.c)                              │◄──┘  │
│  │ drbd_adm_attach/connect/primary/...                      │      │
│  └──────────────┬───────────────────────────────────────────┘      │
│                 │                                                   │
│  ┌──────────────▼───────────────────────────────────────────┐      │
│  │ Object Graph (drbd_int.h)                                │      │
│  │ resource → connections → peer_devices                    │      │
│  │         → devices (IDR by volume)                        │      │
│  └──────────────┬───────────────────────────────────────────┘      │
│                 │                                                   │
│  ┌──────────────▼───────────────────────────────────────────┐      │
│  │ State Machine (drbd_state.c)                             │      │
│  │ begin/end_state_change, sanitize, validate, apply        │      │
│  │ P_TWOPC_* for distributed role changes                   │      │
│  └──────────────┬───────────────────────────────────────────┘      │
│                 │                                                   │
│  ┌──────────────▼───────────────────────────────────────────┐      │
│  │ I/O Request Path (drbd_req.c)                            │      │
│  │ drbd_make_request → AL → interval tree → local + net    │      │
│  │ rq_state[] per peer, bio_endio when all bits clear       │      │
│  └────┬─────────────────────────────────┬────────────────────┘      │
│       │ local                           │ network                   │
│  ┌────▼──────────┐              ┌───────▼────────────────┐         │
│  │ Local Disk    │              │ Sender (drbd_sender.c) │         │
│  │ backing_bdev  │              │ drbd_send_dblock()     │         │
│  │ AL (actlog.c) │              │ P_DATA, P_BARRIER      │         │
│  │ Bitmap (bm.c) │              └───────┬────────────────┘         │
│  └────┬──────────┘                      │ TCP / RDMA               │
│       │ endio                           │ (transport_tcp.c)        │
│  ┌────▼──────────┐              ┌───────▼────────────────┐         │
│  │ req_mod()     │              │ Receiver(drbd_recv.c)  │         │
│  │ rq_state bits │              │ receive_Data()         │         │
│  │ drbd_req_     │◄─────────────│ got_WriteAck()         │         │
│  │ complete()    │   P_WRITE_ACK│ e_end_block()          │         │
│  └────┬──────────┘              └────────────────────────┘         │
│       │ bio_endio                                                   │
│  APPLICATION                                                        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. The 30 Key Functions — Can You Find Them Without Grep?

Test yourself: open each file and navigate to each function without using search.

| # | Function | File | What it does |
|---|---|---|---|
| 1 | `drbd_init()` | `drbd_main.c` | Module init: register blkdev, mempools, genl |
| 2 | `drbd_create_device()` | `drbd_main.c` | Allocate gendisk, connect make_request |
| 3 | `drbd_make_request()` | `drbd_req.c` | Entry point for all block I/O |
| 4 | `__req_mod()` | `drbd_req.c` | rq_state machine driver |
| 5 | `drbd_req_complete()` | `drbd_req.c` | bio_endio gating logic |
| 6 | `drbd_al_begin_io()` | `drbd_actlog.c` | Reserve AL slot before write |
| 7 | `drbd_al_write_transaction()` | `drbd_actlog.c` | Synchronous AL flush to disk |
| 8 | `drbd_al_apply_to_bm()` | `drbd_actlog.c` | Crash recovery: mark AL extents OOS |
| 9 | `lc_get()` | `lru_cache.c` | LRU cache lookup + eviction |
| 10 | `drbd_bm_find_next()` | `drbd_bitmap.c` | Resync cursor: next OOS bit |
| 11 | `drbd_insert_interval()` | `drbd_interval.c` | Add I/O range to augmented RB tree |
| 12 | `drbd_find_overlap()` | `drbd_interval.c` | O(log n) overlap detection |
| 13 | `drbd_sender()` | `drbd_sender.c` | Sender thread main loop |
| 14 | `drbd_send_dblock()` | `drbd_sender.c` | Build and send P_DATA packet |
| 15 | `drbd_receiver()` | `drbd_receiver.c` | Receiver thread main loop |
| 16 | `receive_Data()` | `drbd_receiver.c` | Process incoming P_DATA on secondary |
| 17 | `e_end_block()` | `drbd_receiver.c` | Secondary: send P_WRITE_ACK after write |
| 18 | `got_WriteAck()` | `drbd_receiver.c` | Primary: process P_WRITE_ACK → bio_endio |
| 19 | `w_make_resync_request()` | `drbd_worker.c` | SyncSource burst loop |
| 20 | `drbd_resync_finished()` | `drbd_worker.c` | Resync done: UUID update, state change |
| 21 | `begin_state_change()` | `drbd_state.c` | Start two-phase state transition |
| 22 | `end_state_change()` | `drbd_state.c` | Validate + apply + post-change work |
| 23 | `__after_state_change()` | `drbd_state.c` | Post-state callbacks (I/O resume, resync) |
| 24 | `drbd_uuid_compare()` | `drbd_receiver.c` | UUID-based sync/split-brain decision |
| 25 | `drbd_do_handshake()` | `drbd_receiver.c` | Feature + auth negotiation |
| 26 | `dtt_connect()` | `drbd_transport_tcp.c` | Establish two TCP sockets |
| 27 | `drbd_md_sync()` | `drbd_main.c` | Flush metadata superblock to disk |
| 28 | `drbd_adm_attach()` | `drbd_nl.c` | Disk attach: open, read metadata, AL init |
| 29 | `drbd_adm_primary()` | `drbd_nl.c` | Role change via Netlink |
| 30 | `drbd_have_quorum()` | `drbd_state.c` | Quorum vote counting |

---

## 3. Common Pitfalls — Bugs That Trip Up DRBD Developers

### Pitfall 1: Forgetting `put_ldev()` on error paths
```c
// WRONG:
if (!get_ldev_if_state(device, D_UP_TO_DATE))
    return -EIO;
// ... code that returns early on error without put_ldev() ...
return 0;

// RIGHT: use goto pattern
if (!get_ldev_if_state(device, D_UP_TO_DATE))
    return -EIO;
// ...
err = do_something();
put_ldev(device);      // ALWAYS before return
return err;
```

### Pitfall 2: Taking `req_lock` when `al_lock` is needed (lock ordering)
```c
// WRONG: al_lock inside req_lock (can cause deadlock)
spin_lock_irq(&resource->req_lock);
    spin_lock_irqsave(&device->al_lock, flags);  // DEADLOCK RISK

// RIGHT: al_lock is independent; never nest req_lock inside al_lock
spin_lock_irqsave(&device->al_lock, flags);
// do AL work
spin_unlock_irqrestore(&device->al_lock, flags);

spin_lock_irq(&resource->req_lock);
// do req work
spin_unlock_irq(&resource->req_lock);
```

### Pitfall 3: RCU dereference outside `rcu_read_lock()`
```c
// WRONG:
struct net_conf *nc = rcu_dereference(connection->net_conf);
// ... nc may be freed by another thread

// RIGHT:
rcu_read_lock();
struct net_conf *nc = rcu_dereference(connection->net_conf);
// use nc quickly
rcu_read_unlock();
// Or: kref-get the object before using across sleep points
```

### Pitfall 4: Missing `wake_up()` after clearing a wait condition
```c
// WRONG:
peer_req->i.completed = 1;
// forgot: wake_up(&device->misc_wait)
// → receivers waiting for this range sleep forever

// RIGHT:
peer_req->i.completed = 1;
wake_up(&device->misc_wait);
```

### Pitfall 5: Not checking `cancel` flag in work callbacks
```c
// WRONG:
static int my_work_cb(struct drbd_work *w, int cancel)
{
    // forget to check cancel!
    do_network_io();  // CRASH: transport may be torn down
}

// RIGHT:
static int my_work_cb(struct drbd_work *w, int cancel)
{
    if (cancel)
        return 0;
    do_network_io();
}
```

### Pitfall 6: Sending on wrong socket stream
```c
// WRONG: sending P_BARRIER on DATA_STREAM
drbd_send_command(peer_device, DATA_STREAM, P_BARRIER, ...);

// RIGHT: control packets go on CONTROL_STREAM (META socket)
drbd_send_command(peer_device, CONTROL_STREAM, P_BARRIER, ...);
```

---

## 4. Debugging Checklist — When DRBD Behaves Unexpectedly

### Stuck I/O
```bash
# 1. Check oldest request:
cat /sys/kernel/debug/drbd/r0/volumes/0/oldest_requests
# Look at rq_state bits: which bits are still set?
# RQ_NET_PENDING: waiting for ACK from peer
# RQ_LOCAL_PENDING: waiting for local disk

# 2. Check if sender thread is alive:
ps aux | grep "drbd_s"

# 3. Check network:
cat /sys/kernel/debug/drbd/r0/connections/peer1/transport

# 4. Check for I/O errors:
dmesg | grep drbd | tail -30
```

### Unexpected Resync
```bash
# 1. Check UUID state:
drbdsetup show-gi r0 peer_node_id=1

# 2. Check bitmap weight (OOS sectors):
cat /sys/kernel/debug/drbd/r0/connections/peer1/volumes/0/resync_extents | head -20

# 3. Check metadata flags:
drbdmeta 0 v09 /dev/sda internal dump-md | grep flags
```

### Performance Problems
```bash
# 1. AL miss rate:
cat /sys/kernel/debug/drbd/r0/volumes/0/act_log
# High misses → increase al-extents or use faster metadata device

# 2. Protocol / write ordering:
grep "wo:" /proc/drbd
# wo:n = no ordering (fastest), wo:f = flush (safe)

# 3. Network saturation:
cat /sys/kernel/debug/drbd/r0/connections/peer1/transport
# Check snd_cwnd, RTT
```

---

## 5. Contributing to DRBD Upstream

### Where to Submit

DRBD 9 is developed by LINBIT. Contributions go to:
- GitHub: https://github.com/LINBIT/drbd
- Mailing list: drbd-dev@lists.linbit.com
- For kernel tree integration: DRBD is at `drivers/block/drbd/` (DRBD 8.4 only in mainline)

### Contribution Workflow

```bash
# 1. Fork the repo
git clone https://github.com/LINBIT/drbd.git -b drbd-9.2
cd drbd

# 2. Create a feature branch
git checkout -b fix/missing-put-ldev-in-attach

# 3. Make and test change
vim drbd/drbd_nl.c
make -C /lib/modules/$(uname -r)/build M=$(pwd)/drbd modules
# Test on a VM cluster

# 4. Commit with proper message
git add -p
git commit -s   # -s adds Signed-off-by

# 5. Check style
perl scripts/checkpatch.pl --no-tree 0001-*.patch

# 6. Send patch
git format-patch HEAD~1 -o outgoing/
# Email to drbd-dev@lists.linbit.com
```

### Good First Contributions

1. **Documentation:** Missing or outdated inline comments (`/* */`)
2. **Spelling fixes:** `codespell drbd/*.c`
3. **Static analysis:** Run `sparse`, `smatch`, or `coccinelle` and fix findings
4. **Compat additions:** Support a new kernel version in `drbd/compat/`
5. **Test cases:** Add test scenarios to the DRBD test suite (drbdtest)

---

## 6. Your Learning Roadmap Beyond Day 30

### Tier 1: Deepen DRBD Knowledge
- Read DRBD 8.4 in the mainline kernel (`drivers/block/drbd/`) — simpler 2-node code, good for learning
- Study DRBD release notes for each 9.x version — what changed, why
- Run `drbdtest` test suite and understand each test case

### Tier 2: Adjacent Kernel Subsystems
- **Block layer:** `block/blk-core.c`, `block/blk-mq.c` — how bios flow before hitting DRBD
- **LVM thin provisioning:** Many DRBD deployments sit on LVM — understand dm-thin
- **Device mapper:** `dm-cache`, `dm-writecache` — often used alongside DRBD
- **Network:** `net/ipv4/tcp.c`, `net/core/sock.c` — the sockets DRBD writes to

### Tier 3: Distributed Systems Theory
- **CAP theorem** applied to DRBD: Consistency + Partition-tolerance at cost of Availability (quorum)
- **Paxos / Raft:** How DRBD's two-phase commit (P_TWOPC) compares to Raft leader election
- **Write-ahead logging:** How DRBD's Activity Log compares to database WAL

### Tier 4: Storage Stack Integration
- **Pacemaker + DRBD:** How `crm-fence-peer` integrates with STONITH
- **Kubernetes + DRBD:** `linstor-csi` and how PVCs use DRBD volumes
- **Ceph RADOS vs DRBD:** Different distributed storage philosophies

---

## 7. The 10 Most Important Files — Final Rankings

| Rank | File | Why |
|---|---|---|
| 1 | `drbd_int.h` | All structs — the vocabulary of DRBD |
| 2 | `drbd_req.c` | The I/O path — everything flows through here |
| 3 | `drbd_receiver.c` | Largest file, secondary-side brain |
| 4 | `drbd_state.c` | State machine — controls all transitions |
| 5 | `drbd_actlog.c` | AL — the crash safety mechanism |
| 6 | `drbd_sender.c` | Network output — the P_DATA path |
| 7 | `drbd_bitmap.c` | OOS tracking — drives all resync decisions |
| 8 | `drbd_nl.c` | Userspace interface — all admin commands |
| 9 | `drbd_worker.c` | Background work — resync, metadata, OV |
| 10 | `drbd_transport_tcp.c` | Network I/O — bytes on the wire |

---

## 8. Self-Assessment Questions

Answer each without looking at code:

1. What is `dagtag` and why is it needed in multi-node clusters?
2. When does `bio_endio()` fire in Protocol A vs Protocol C?
3. What is the `block_id` trick and why is it O(1)?
4. What does `drbd_al_apply_to_bm()` do and when is it called?
5. What are the three dimensions of DRBD state?
6. What is the `rb_max_end` field in `drbd_interval` and why does it enable O(log n) overlap queries?
7. What is an epoch and what triggers a new one?
8. Why does `drbd_uuid_compare()` return -1000 and what happens next?
9. What is `lc_committed()` and when must it be called?
10. How does TCP backpressure flow from a slow secondary to the primary's application?

---

## 9. Final Hands-On: Build Something

Choose one of these projects to consolidate your learning:

**Project A: Add a new DebugFS entry**
Add a file `/sys/kernel/debug/drbd/r0/volumes/0/write_latency` that shows the average time between `drbd_make_request()` and `bio_endio()` for the last 100 writes. Use `ktime_get()` and a circular buffer.

**Project B: Write a Coccinelle rule**
Write a `.cocci` file that finds all `spin_lock_irq(&resource->req_lock)` calls that are not matched with a `spin_unlock_irq(&resource->req_lock)` on all paths. Run it on `drbd_req.c`.

**Project C: Instrument the resync rate controller**
Add `printk(KERN_DEBUG ...)` statements inside `drbd_rs_controller()` to log the input parameters and output value on each call. Rebuild, run `drbdadm resync-from`, and analyse the rate controller's behaviour under a changing workload.

**Project D: Trace a specific packet type**
Add a DebugFS counter for `P_BARRIER` packets sent and received. Display it in the connection's transport file. This requires touching `drbd_transport_tcp.c`, `drbd_sender.c`, `drbd_receiver.c`, and `drbd_debugfs.c`.

---

## 10. Congratulations — What You Now Know

Over 30 days you traced every major subsystem of DRBD 9.2:

- **Days 1–7:** Foundation — structs, state machine, I/O path, protocol, receiver, connection setup
- **Days 8–14:** Storage layer — AL, bitmap, resync, worker, interval tree, LRU cache, backing device
- **Days 15–21:** Control & management — Netlink, transport, UUIDs, quorum, debugfs, verify, Ahead/Behind
- **Days 22–28:** Advanced — multi-node, build system, end-to-end trace, performance, TLS, epochs, error handling
- **Days 29–30:** Architecture — RDMA, synthesis, pitfalls, contributing

You can now:
1. Read any function in the DRBD codebase and understand its role
2. Trace an I/O request from application through kernel to peer and back
3. Debug stuck I/O using DebugFS and `/proc/drbd`
4. Write and submit a kernel patch to the DRBD project
5. Tune DRBD performance for a specific workload
6. Design a multi-node DRBD cluster with appropriate quorum configuration

**The source is your primary reference. Return to it often.**
