# Day 28 — Error Handling & Failure Scenarios: Disk Failure, Network Partition & Recovery

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_req.c`, `drbd/drbd_receiver.c`, `drbd/drbd_state.c`, `drbd/drbd_main.c`

---

## 1. Failure Taxonomy

```
Failure types DRBD handles:

1. Local disk write error (primary)
   → depends on on-io-error policy

2. Local disk read error (primary, read request from application)
   → try peer read (P_DATA_REQUEST)

3. Local disk write error (secondary)
   → send P_NEG_ACK to primary

4. Network partition (connection lost)
   → tl_walk() to fail/requeue in-flight writes
   → reconnect + resync

5. Peer node crash (no clean disconnect)
   → timeout detected via P_PING mechanism
   → fencing if configured

6. Primary crash (unclean shutdown)
   → on reconnect: AL-based resync (Day 8)
   → UUID-based decision (Day 17)

7. Both nodes crash simultaneously
   → requires manual intervention or quorum configuration
```

---

## 2. Local Disk Write Error on Primary

```bash
grep -n "WRITE_COMPLETED_WITH_ERROR\|drbd_handle_failed_mirror\b\|EP_DETACH\|on_io_error" \
    drbd/drbd_req.c drbd/drbd_main.c drbd/drbd_int.h | head -20
```

### Scenario: Primary writes 4 KiB, local disk returns `-EIO`

```
[Primary, softirq] drbd_request_endio(private_bio)
  bio->bi_status = BLK_STS_IOERR

LOCK ACQUIRED: spin_lock_irqsave(&resource->req_lock)

__req_mod(req, WRITE_COMPLETED_WITH_ERROR, NULL, &m)
  → rq_state[0] |= RQ_LOCAL_ABORTED
  → rq_state[0] &= ~RQ_LOCAL_PENDING

  → drbd_req_complete(req, &m):
       // local failed, but net may still be pending
       // bio_endio is NOT called yet if net still pending

LOCK RELEASED

// Then: check on_io_error policy:
drbd_handle_failed_mirror(device, req)
```

```bash
grep -n -A 60 "^static void drbd_handle_failed_mirror\b\|drbd_handle_failed_mirror\b" \
    drbd/drbd_req.c drbd/drbd_main.c | head -70
```

```c
static void drbd_handle_failed_mirror(struct drbd_device *device,
                                       struct drbd_request *req)
{
    struct disk_conf *dc = rcu_dereference(device->ldev->disk_conf);

    switch (dc->on_io_error) {
    case EP_PASSTHROUGH:
        // Return EIO to application, don't detach
        // (app must handle the error)
        break;

    case EP_CALL_HELPER:
        // Schedule call to /lib/drbd/notify-io-error.sh
        // Script can decide to detach or take other action
        drbd_kobject_uevent_env(device, IO_ERROR, envp);
        // Also detach (fall through):

    case EP_DETACH:
        // Mark disk as Failed
        // (causes disk_state → D_FAILED → D_DISKLESS)
        drbd_handle_io_error(device, DRBD_FORCE_DETACH);
        break;
    }
}
```

```bash
grep -n "drbd_handle_io_error\b\|DRBD_FORCE_DETACH\b" drbd/drbd_req.c drbd/drbd_main.c | head -10
```

### Result of Detach with Connected Peer (UpToDate)

After disk fails and detach runs:
```
disk_state: D_FAILED → D_DISKLESS
repl_state: L_ESTABLISHED (still connected)

__after_state_change():
  → if role == R_PRIMARY && disk == D_DISKLESS:
      → if fencing == FP_RESOURCE or FP_STONITH:
           change_role(R_SECONDARY) ← forced demotion
      → else: continue as diskless Primary (reads from peer)
```

---

## 3. Local Disk Read Error on Primary — Peer Read Fallback

```bash
grep -n "READ_COMPLETED_WITH_ERROR\|drbd_send_drequest\b\|P_DATA_REQUEST\b" \
    drbd/drbd_req.c drbd/drbd_sender.c | head -20
```

```
[Primary, softirq] drbd_request_endio(bio)
  bio->bi_status = BLK_STS_IOERR (for a READ bio)

__req_mod(req, READ_COMPLETED_WITH_ERROR, NULL, &m)
  → rq_state[0] |= RQ_LOCAL_ABORTED

  // If peer is connected and UpToDate:
  if (can_do_remote) {
      // Ask peer to read this block
      drbd_send_drequest(peer_device, P_DATA_REQUEST,
                          req->i.sector, req->i.size,
                          (u64)(uintptr_t)req)
      // req waits for P_DATA_REPLY
      rq_state[peer_idx] |= RQ_NET_PENDING
  } else {
      // No peer available → fail the read
      m.bio   = req->master_bio
      m.error = -EIO
      // bio_endio(m.bio, -EIO) → application gets EIO
  }
```

On the secondary (`receive_DataRequest`):
```bash
grep -n -A 40 "^static int receive_DataRequest\b" drbd/drbd_receiver.c
```

```c
static int receive_DataRequest(struct drbd_connection *connection,
                                struct packet_info *pi)
{
    // 1. Allocate peer_req
    // 2. Submit local READ
    // 3. On completion: drbd_send_block(P_DATA_REPLY, peer_req)
    //    → primary receives P_DATA_REPLY
    //    → __req_mod(req, DATA_RECEIVED, ...) → bio_endio() with data
}
```

---

## 4. Local Disk Write Error on Secondary — `P_NEG_ACK`

```bash
grep -n "P_NEG_ACK\b\|got_NegAck\b\|EE_WAS_ERROR\b" drbd/drbd_receiver.c drbd/drbd_req.c | head -20
```

```
[Secondary, softirq] drbd_peer_request_endio(bio)
  bio->bi_status = BLK_STS_IOERR

  set_bit(EE_WAS_ERROR, &peer_req->flags)
  → move to done_ee
  → wake sender thread

[Secondary, sender thread] e_end_block():
  if (test_bit(EE_WAS_ERROR, &peer_req->flags)):
      drbd_send_ack(peer_device, P_NEG_ACK, peer_req)
      drbd_set_out_of_sync(peer_device, sector, size)
      // → bitmap bit stays SET (block remains OOS on secondary)

      // If secondary disk_conf->on_io_error == EP_DETACH:
      change_disk_state(device, D_FAILED, CS_HARD)
```

On the Primary receiving `P_NEG_ACK`:

```bash
grep -n -A 40 "^static int got_NegAck\b" drbd/drbd_receiver.c
```

```c
static int got_NegAck(struct drbd_connection *connection,
                       struct packet_info *pi)
{
    struct drbd_request *req = (struct drbd_request *)(uintptr_t)p->block_id;

    spin_lock_irq(&resource->req_lock);
    __req_mod(req, NEG_ACKED, peer_device, &m);
    // → rq_state[peer_idx] &= ~RQ_NET_PENDING
    //   rq_state[peer_idx] |=  RQ_NET_DONE
    //   (but NOT RQ_NET_OK — write failed on peer)
    spin_unlock_irq(&resource->req_lock);

    // The primary still completed its local write (if OK)
    // bio_endio() may fire here with success (local disk was OK)
    // Application write succeeds even though secondary failed!
    // → This is Protocol C — it guarantees the data is ON DISK somewhere

    // But: peer's disk state may be set to Failed/Outdated
    // → peer disk needs resync after recovery
}
```

---

## 5. Network Partition — Connection Loss

### Detection

```bash
grep -n "ping.*timeout\|last_received\|DRBD_PING_TIMEOUT\|C_NETWORK_FAILURE" \
    drbd/drbd_receiver.c | head -15
```

```c
// drbd_receiver.c: main loop
if (time_after(jiffies,
               connection->last_received + net_conf->ping_timeo * HZ / 10)) {
    // Timeout: no data received for ping_timeo × 100ms
    if (drbd_send_ping(connection) != 0) {
        // Can't even send ping → definitely disconnected
        goto out_disconnect;
    }
    connection->ko_count++;
    if (connection->ko_count > net_conf->ko_count)
        goto out_disconnect;
}
```

### Disconnect Sequence

```bash
grep -n -A 100 "^static void drbd_disconnect\b\|out_disconnect:" drbd/drbd_receiver.c | head -110
```

```
drbd_disconnect(connection):
│
├─ 1. change_cstate(connection, C_DISCONNECTING, CS_HARD)
│   → __after_state_change(): doesn't start resync yet
│
├─ 2. drbd_thread_stop(connection->sender)
│   → sender thread exits, pending work queue drained with cancel=1
│
├─ 3. drbd_flush_workqueue()
│
├─ 4. drbd_finish_peer_reqs(connection)
│   → for each active_ee / sync_ee / done_ee / read_ee:
│       set EE_WAS_ERROR if write was in flight
│       call e_end_block() or equivalent with cancel=1
│       → P_NEG_ACK NOT sent (peer is gone)
│       → OOS bits SET in bitmap (block may not have reached peer)
│
├─ 5. tl_walk(connection, CONNECTION_LOST_WHILE_PENDING)
│   → for each drbd_request in transfer_log:
│       __req_mod(req, SEND_CANCELED or similar, peer_device, &m)
│       → clears RQ_NET_PENDING for this peer
│       → if all other conditions met: bio_endio(m.bio)
│
├─ 6. close_transport(connection)
│   → sock_release(data_socket)
│   → sock_release(meta_socket)
│
├─ 7. change_cstate(connection, C_UNCONNECTED, CS_VERBOSE)
│   → __after_state_change():
│       peer disk_state → D_UNKNOWN
│       repl_state      → L_OFF
│
└─ 8. Schedule reconnect:
    mod_timer(&connection->connect_timer, jiffies + RECONNECT_BACKOFF)
```

---

## 6. In-Flight Writes During Disconnect — What Happens to Applications?

This is the most important failure scenario. With Protocol C:

**Case A: Local write already completed, peer not yet ACKed:**
```
rq_state[0] = RQ_LOCAL_OK (local done)
rq_state[peer] = RQ_NET_PENDING (waiting for ACK)

On disconnect: __req_mod(req, CONNECTION_LOST_WHILE_PENDING)
  → rq_state[peer] &= ~RQ_NET_PENDING
  → rq_state[peer] |= RQ_NET_DONE (without RQ_NET_OK)
  → drbd_req_complete(): local done + net done → bio_endio(bio, 0) ← SUCCESS!
```

The write is reported **successful to the application** even though we don't know if it reached the peer! The bitmap bit remains set for the peer → it will be resynced on reconnect.

**Case B: Local write not yet done, peer not yet sent:**
```
rq_state[0] = RQ_LOCAL_PENDING
rq_state[peer] = RQ_NET_PENDING

On disconnect: peer portion cleared
On local endio: local portion cleared → bio_endio(bio, 0)
```

Again, **application sees success**, but OOS bitmap ensures resync.

---

## 7. Primary Crash Recovery — AL-Based Resync

```bash
grep -n "MDF_CRASHED_PRIMARY\|drbd_al_apply_to_bm\b\|CRASHED_PRIMARY" \
    drbd/drbd_nl.c drbd/drbd_main.c | head -10
```

When a crashed Primary reboots and reattaches its disk:

```
1. drbd_md_read() → loads metadata
   MDF_PRIMARY_IND = 1  (was Primary)
   MDF_CONSISTENT  = 0  (didn't write clean shutdown flag)
   → CRASHED_PRIMARY detected

2. drbd_al_apply_to_bm(device)
   → For each active AL slot: mark OOS in bitmap
   → These are the potentially-inconsistent extents

3. On reconnect to secondary:
   UUID comparison:
   → if primary has CRASHED_PRIMARY flag:
       peer_uuid in primary's history → normal resync
       (only AL extents need resync, not full device)
```

---

## 8. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Simulate and trace a primary disk failure
On a test VM with DRBD:
```bash
# Inject a disk error (using dm-flakey or null_blk):
# Or use: echo 1 > /sys/block/sda/sda1/make-it-fail  # if CONFIG_FAIL_MAKE_REQUEST

# Alternatively: trace the code path
grep -n -A 80 "^static void drbd_handle_io_error\b\|EP_DETACH\b\|disk_state.*D_FAILED" \
    drbd/drbd_main.c drbd/drbd_req.c | head -90
```
Trace all state changes that occur: disk_state transitions, any role changes, what the secondary sees.

### Exercise 2 (40 min): Read `tl_walk()` completely
```bash
grep -n "tl_walk\b\|tl_epoch_barrier_ack\b\|CONNECTION_LOST_WHILE_PENDING\|RESEND\b" \
    drbd/drbd_req.c drbd/drbd_main.c | head -20
grep -n -A 80 "^static void tl_walk\b\|^void tl_walk\b" drbd/drbd_req.c drbd/drbd_main.c
```
What are all the possible `drbd_req_event` values passed to `__req_mod()` by `tl_walk()`? When is each used?

### Exercise 3 (40 min): Trace `P_NEG_ACK` end-to-end
From secondary disk write failure to primary deciding the application write succeeded:
1. `drbd_peer_request_endio()` on secondary
2. `e_end_block()` sends `P_NEG_ACK`
3. `got_NegAck()` on primary
4. `__req_mod()` with `NEG_ACKED`
5. `drbd_req_complete()` — does it call `bio_endio()` with success or error?

### Exercise 4 (35 min): Understand "dual primary" split-brain prevention
In a 2-node cluster with `allow-two-primaries no` (default):
```bash
grep -n "allow_two_primaries\|two.primaries\|R_PRIMARY.*R_PRIMARY" \
    drbd/drbd_state.c drbd/drbd_nl.c | head -15
```
What check in `is_valid_transition()` prevents both nodes from being Primary simultaneously? What happens if the peer sends P_STATE claiming to be Primary while we are also Primary?

### Exercise 5 (35 min): Trace clean shutdown vs crash
**Clean shutdown:** `drbdadm down r0`
```bash
grep -n "drbd_adm_down\b\|MDF_CONSISTENT\b" drbd/drbd_nl.c drbd/drbd_main.c | head -10
```
What `MDF_*` flags are set? Is `drbd_md_sync()` called?

**Crash (power loss):** 
On reboot, what `MDF_*` combination signals "crashed as primary"? What AL-based recovery runs?

---

## Summary

DRBD handles three classes of disk errors: primary local write errors (per `on-io-error` policy — passthrough, call-helper, or detach), primary local read errors (peer-read fallback via `P_DATA_REQUEST`), and secondary write errors (negative ACK via `P_NEG_ACK`, block stays OOS). Network disconnection triggers `drbd_disconnect()` which drains all in-flight state, calls `tl_walk()` to resolve pending application writes (reporting success since local write completed), and sets OOS bitmap bits for the peer. On primary crash recovery, the AL-based resync mechanism (`drbd_al_apply_to_bm()`) limits the resync scope to only the extents that were active at crash time.

**Next:** Day 29 — Advanced topics: RDMA transport overview, `drbd_transport_rdma` architecture, zero-copy data paths, and comparing TCP vs RDMA performance characteristics.
