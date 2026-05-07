# Day 21 — Ahead/Behind: Congestion Control, `L_AHEAD`/`L_BEHIND` & `P_OUT_OF_SYNC`

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_req.c`, `drbd/drbd_state.c`, `drbd/drbd_worker.c`, `drbd/drbd_receiver.c`, `drbd/drbd_int.h`

---

## 1. The Problem: Slow Secondary Blocking Primary

In Protocol C (synchronous), every application write on the Primary must wait for `P_WRITE_ACK` from the Secondary before `bio_endio()` is called. If the Secondary's disk is slow (e.g., under heavy resync load), Primary I/O latency climbs unboundedly.

**Ahead/Behind** is a congestion-relief mechanism:

```
Normal (Protocol C):
  Primary write → send P_DATA → wait P_WRITE_ACK → bio_endio()
  Latency = local_disk_time + RTT + peer_disk_time

Congested → transition to Ahead/Behind:
  Primary (L_AHEAD): write locally + record OOS, DON'T wait for ACK
  Secondary (L_BEHIND): keep receiving data, write locally at its own pace

Effect: Primary latency returns to local-disk-only latency
Cost:   If Primary crashes while Ahead, secondary must fully resync
```

This is a **deliberate trade-off**: latency vs. potential data loss window.

---

## 2. Configuration

```bash
grep -n "on.congestion\|congestion.fill\|congestion.extents\|cong_fill\|cong_extents" \
    drbd/drbd_int.h drbd/drbd_nl.c | head -15
```

```c
// From drbd.conf:
// net {
//   on-congestion pull-ahead;     // switch to Ahead/Behind when congested
//   congestion-fill 100M;         // enter Ahead when TCP send buffer > 100M in flight
//   congestion-extents 1000;      // enter Ahead when > 1000 AL extents in flight
// }

enum drbd_on_congestion {
    OC_BLOCK,         // default: block until congestion clears
    OC_PULL_AHEAD,    // switch to Ahead/Behind
    OC_DISCONNECT,    // disconnect when congested
};
```

---

## 3. Congestion Detection — `drbd_congested()`

```bash
grep -n "drbd_congested\b\|cong_fill\|cong_extents\|ap_in_flight" \
    drbd/drbd_req.c drbd/drbd_main.c drbd/drbd_int.h | head -20
grep -n -A 40 "^static bool drbd_congested\b\|^bool drbd_congested\b" drbd/drbd_req.c
```

```c
static bool drbd_congested(struct drbd_peer_device *peer_device)
{
    struct net_conf *nc = rcu_dereference(peer_device->connection->net_conf);

    if (nc->on_congestion == OC_BLOCK)
        return false;  // Ahead/Behind disabled

    // Check 1: too many bytes in-flight (not yet ACKed by peer)?
    if (nc->cong_fill > 0) {
        int in_flight_bytes = atomic_read(&peer_device->connection->ap_in_flight)
                              << 9;  // in sectors → bytes
        if (in_flight_bytes >= nc->cong_fill)
            return true;
    }

    // Check 2: too many AL extents active (many concurrent write sectors)?
    if (nc->cong_extents > 0) {
        if (peer_device->device->act_log->used >= nc->cong_extents)
            return true;
    }

    return false;
}
```

```bash
grep -n "ap_in_flight\b" drbd/drbd_int.h drbd/drbd_req.c drbd/drbd_receiver.c | head -15
```

`ap_in_flight` (application writes in flight) is incremented when a write is sent to the peer and decremented when `P_WRITE_ACK` / `P_BARRIER_ACK` is received.

---

## 4. Entering `L_AHEAD` — Transition Logic

```bash
grep -n "L_AHEAD\b\|L_BEHIND\b\|drbd_congested\b" drbd/drbd_req.c drbd/drbd_state.c | head -20
```

In `drbd_make_request()`, after enqueueing the write for the sender:

```bash
grep -n -A 30 "drbd_congested\b\|OC_PULL_AHEAD\|pull_ahead" drbd/drbd_req.c | head -40
```

```c
// drbd_req.c: drbd_process_write_request() or drbd_make_request()
for_each_peer_device(peer_device, device) {
    if (peer_device->repl_state[NOW] == L_ESTABLISHED) {

        if (drbd_congested(peer_device)) {
            // Too congested — switch to Ahead
            if (peer_device->connection->net_conf->on_congestion == OC_PULL_AHEAD) {
                begin_state_change(resource, ...);
                __change_repl_state(peer_device, L_AHEAD);
                end_state_change(resource, ...);
                // __after_state_change() → send P_CONN_ST_CHG to peer
                // peer transitions to L_BEHIND
            } else if (... == OC_DISCONNECT) {
                drbd_disconnect_or_abort(peer_device->connection);
            }
        }
    }
}
```

---

## 5. Operating in `L_AHEAD` — What Changes

When `repl_state == L_AHEAD` for a peer:

```bash
grep -n "L_AHEAD\b" drbd/drbd_req.c | head -15
```

**On each application write:**
1. Write is submitted to local disk (normal)
2. Write is NOT sent to peer as `P_DATA`
3. Instead: `drbd_set_out_of_sync()` marks the bitmap bit OOS for that peer
4. `w_send_out_of_sync` work item sends `P_OUT_OF_SYNC` to the Behind peer

```bash
grep -n "P_OUT_OF_SYNC\b\|w_send_out_of_sync\b\|drbd_set_out_of_sync\b" \
    drbd/drbd_req.c drbd/drbd_worker.c drbd/drbd_receiver.c | head -15
```

```c
// drbd_req.c: for L_AHEAD peer:
set_bit(SEND_OOS_AFTER_BARRIER, &peer_device->flags);
// Or directly:
drbd_queue_work(&connection->sender_work, &peer_device->send_oos_work);
// → w_send_out_of_sync() → drbd_send_out_of_sync()
//   → P_OUT_OF_SYNC packet: {sector, size}
```

`bio_endio()` fires as soon as the **local write** completes — no peer ACK needed in `L_AHEAD`. This is what restores primary latency.

---

## 6. `P_OUT_OF_SYNC` — Notifying the Behind Peer

```bash
grep -n "struct p_block_desc\|P_OUT_OF_SYNC\b" drbd/drbd_protocol.h | head -5
grep -n -A 30 "^static int receive_out_of_sync\b\|receive_OutOfSync\b" drbd/drbd_receiver.c
```

```c
// drbd_receiver.c: receive_out_of_sync()
static int receive_out_of_sync(struct drbd_connection *connection,
                                struct packet_info *pi)
{
    struct p_block_desc *p = pi->data;
    sector_t sector = be64_to_cpu(p->sector);
    int size         = be32_to_cpu(p->blksize);

    // Mark these sectors OOS in our bitmap
    drbd_bm_set_bits(device, peer_device->bitmap_index,
                     BM_SECT_TO_BIT(sector),
                     BM_SECT_TO_BIT(sector + size/512) - 1);

    // These sectors will need resync when we exit L_BEHIND
    return 0;
}
```

The Behind peer keeps accumulating OOS bits. When Ahead/Behind ends, a bitmap-based resync will clear them all.

---

## 7. Operating in `L_BEHIND` — Secondary Behaviour

```bash
grep -n "L_BEHIND\b" drbd/drbd_receiver.c drbd/drbd_worker.c | head -15
```

In `L_BEHIND`, the secondary:
1. Continues to receive normal `P_DATA` writes — these are written to disk normally
2. Receives `P_OUT_OF_SYNC` packets — marks those bits OOS (no write needed)
3. Does NOT send `P_WRITE_ACK` for writes received while in `L_BEHIND`
   (because the primary isn't waiting for ACKs — it's using local-disk completion)

Actually — in some DRBD versions in `L_BEHIND`, the secondary still writes received data but the primary doesn't wait for the ACK:

```bash
grep -n "L_BEHIND\|dp_flags\|DP_SEND_WRITE_ACK" drbd/drbd_req.c drbd/drbd_receiver.c | head -20
```

The `dp_flags` in `P_DATA` do NOT include `DP_SEND_WRITE_ACK` when in `L_AHEAD`, so the secondary correctly doesn't send an ACK.

---

## 8. Exiting `L_AHEAD` — Return to `L_ESTABLISHED`

Congestion eventually clears. How does the Primary detect this and exit `L_AHEAD`?

```bash
grep -n "ahead_to_sync\|exit.*ahead\|L_AHEAD.*ESTABLISHED\|drbd_ahead_to_sync_target" \
    drbd/drbd_worker.c drbd/drbd_state.c drbd/drbd_receiver.c | head -15
```

The mechanism:
1. The Behind peer periodically sends `P_CONN_ST_CHG_SUCC` or similar when its backlog drains
2. OR: The Primary detects that `drbd_congested()` returns false again

```bash
grep -n "drbd_rs_controller\|congestion.*clear\|un_ah\|drbd_ahead\b" drbd/drbd_worker.c | head -10
```

In `w_make_resync_request()` (which also runs during Ahead/Behind to track the catch-up):
```c
// When congestion clears:
if (!drbd_congested(peer_device) &&
    peer_device->repl_state[NOW] == L_AHEAD) {
    begin_state_change(resource, ...);
    __change_repl_state(peer_device, L_STARTING_SYNC_S);
    // → bitmap-based resync to catch up all OOS blocks
    end_state_change(resource, ...);
}
```

---

## 9. After Exiting Ahead/Behind — Resync the Delta

When the Ahead/Behind episode ends, the OOS bitmap contains exactly the blocks written during the congestion window. A normal bitmap-based resync (`L_SYNC_SOURCE` → `L_SYNC_TARGET`) resyncs only those blocks:

```bash
grep -n "L_STARTING_SYNC_S\|L_STARTING_SYNC_T\|__after_state_change.*SYNC" \
    drbd/drbd_state.c | head -15
```

This is much cheaper than a full resync — only the blocks that diverged during Ahead mode need to be resynced.

---

## 10. Week 3 Connections — Putting It All Together

```
Application write arrives at Primary in L_ESTABLISHED:
    drbd_make_request()
    │
    ├─ for each peer: drbd_congested()?
    │   NO: normal path (Day 4)
    │       rq_state |= RQ_NET_PENDING
    │       send P_DATA with DP_SEND_WRITE_ACK
    │       wait for P_WRITE_ACK
    │
    └─ YES: pull-ahead path
        change repl_state L_ESTABLISHED → L_AHEAD
        rq_state[peer_idx] cleared (no ACK needed from this peer)
        local write completes → bio_endio() immediately
        drbd_bm_set_bits() → mark OOS for peer
        send P_OUT_OF_SYNC → peer sets OOS bit in its bitmap

Secondary in L_BEHIND:
    receive_out_of_sync() → drbd_bm_set_bits()
    (writes may still arrive from earlier P_DATA in flight)

Congestion clears:
    Primary: L_AHEAD → L_STARTING_SYNC_S
    Secondary: L_BEHIND → L_STARTING_SYNC_T
    Bitmap resync runs (only OOS blocks)
    After resync: L_ESTABLISHED on both sides
```

---

## 11. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Read all `L_AHEAD` / `L_BEHIND` branches in `drbd_req.c`
```bash
grep -n "L_AHEAD\|L_BEHIND\|drbd_congested" drbd/drbd_req.c
```
For each branch: what is the exact code path, what `rq_state` bits are set, and when does `bio_endio()` fire?

### Exercise 2 (40 min): Trace `w_send_out_of_sync()`
```bash
grep -n -A 40 "^static int w_send_out_of_sync\b" drbd/drbd_worker.c
grep -n "drbd_send_out_of_sync\b" drbd/drbd_sender.c | head -5
grep -n -A 30 "^int drbd_send_out_of_sync\b" drbd/drbd_sender.c
```
What packet is built? What fields does it contain? How does the receiver handle it?

### Exercise 3 (35 min): Understand `ap_in_flight` accounting
```bash
grep -n "ap_in_flight\b" drbd/drbd_req.c drbd/drbd_receiver.c | head -20
```
- Where is `ap_in_flight` incremented?
- Where is it decremented?
- What happens to its count during `L_AHEAD` operation?

### Exercise 4 (35 min): Read `drbd_congested()` and compute thresholds
For a production system with:
- `congestion-fill 4M` (4 MiB = 8192 sectors)
- `congestion-extents 500`
- Write size: 4 KiB (8 sectors) at 10,000 IOPS

At what point does `ap_in_flight` trigger congestion? At what RTT does the fill threshold trigger? (Assume protocol C.)

### Exercise 5 (40 min): Trace Ahead/Behind exit and resync
```bash
grep -n "L_STARTING_SYNC_S\|drbd_ahead_to_sync_target\|exit_ahead\b" \
    drbd/drbd_state.c drbd/drbd_worker.c | head -15
grep -n -A 40 "__after_state_change\b" drbd/drbd_state.c | grep -A 5 "STARTING_SYNC"
```
After Ahead/Behind exits, what triggers the bitmap resync? How does DRBD know which blocks to resync (vs. a full resync)?

---

## Week 3 Complete — Summary Table

| Day | Topic | Key Source |
|---|---|---|
| 15 | Netlink interface | `drbd_nl.c`: `drbd_adm_prepare()`, genl dispatch |
| 16 | Transport abstraction | `drbd_transport_tcp.c`: `dtt_connect()`, `dtt_recv_pages()` |
| 17 | UUID system | `drbd_uuid_compare()`, history ring buffer, split-brain |
| 18 | Quorum & fencing | `drbd_have_quorum()`, P_TWOPC_* two-phase commit |
| 19 | DebugFS / proc | `/proc/drbd` counters, `oldest_requests`, `act_log` |
| 20 | Online verify | `w_make_ov_request()`, hash comparison, `ov_left` |
| 21 | Ahead/Behind | `drbd_congested()`, `L_AHEAD/L_BEHIND`, `P_OUT_OF_SYNC` |

**Next:** Week 4 begins — Day 22: Multi-node topology: 3+ node clusters, `peer_node_id`, `nodes_to_reach`, and how DRBD 9 handles fan-out replication.
