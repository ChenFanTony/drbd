# Day 18 — Quorum, Fencing & Two-Phase Commit for Role Changes

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_state.c`, `drbd/drbd_nl.c`, `drbd/drbd_main.c`, `drbd/drbd_int.h`

---

## 1. The Split-Brain Problem at Scale

With 2 nodes and a network partition, both nodes think the other is dead and both might promote to Primary — causing divergent writes. Solutions:

| Approach | Mechanism |
|---|---|
| **Fencing** | Each node tries to STONITH (kill) the other; loser stops |
| **Quorum** | Node only operates if it can see a majority of nodes |
| **Tiebreaker** | A third lightweight node provides tie-breaking vote |

DRBD 9 implements all three, configurable per resource.

---

## 2. Quorum Configuration

```bash
grep -n "quorum\b\|tiebreaker\|on-no-quorum\|quorum_nodes\|DRBD_QUORUM" \
    drbd/drbd_int.h drbd/drbd_nl.c drbd/drbd_state.c | head -20
```

```c
// net-conf options (from drbd.conf):
// quorum majority;           ← need N/2+1 nodes
// quorum all;                ← need ALL nodes (strict)
// on-no-quorum io-error;     ← suspend I/O when quorum lost
// on-no-quorum suspend-io;   ← (default) suspend, wait for quorum return
// tiebreaker node-id 3;      ← node 3 is a diskless voter
```

```bash
grep -n "enum drbd_quorum\|DRBD_QUORUM_MAJORITY\|DRBD_QUORUM_ALL\|quorum_allowed\b" \
    drbd/drbd_int.h drbd/drbd_state.c | head -15
```

---

## 3. Quorum Calculation — `drbd_have_quorum()`

```bash
grep -n "drbd_have_quorum\b\|have_quorum\b" drbd/drbd_state.c drbd/drbd_main.c | head -10
grep -n -A 60 "^static bool drbd_have_quorum\b\|^bool drbd_have_quorum\b" drbd/drbd_state.c
```

```c
static bool drbd_have_quorum(struct drbd_resource *resource,
                              enum drbd_role role)
{
    struct drbd_connection *connection;
    int voters = 1;        // count ourselves
    int votes_for_me = 1;  // our own vote

    // Count connected peers
    for_each_connection(connection, resource) {
        if (connection->cstate[NOW] < C_CONNECTED)
            continue;

        voters++;

        // A connected UpToDate peer votes for us
        // A diskless tiebreaker also votes (it just provides a vote, no data)
        struct drbd_peer_device *peer_device;
        idr_for_each_entry(&connection->peer_devices, peer_device, vnr) {
            if (peer_device->disk_state[NOW] >= D_UP_TO_DATE ||
                peer_device->disk_state[NOW] == D_DISKLESS) {
                votes_for_me++;
            }
        }
    }

    // Also count tiebreaker node if connected
    // tiebreaker: a node with no disk that just provides vote
    if (resource->tiebreaker && is_tiebreaker_connected(resource))
        voters++;
        // tiebreaker votes for whoever connects to it first

    switch (resource->res_opts.quorum) {
    case QOU_MAJORITY:
        return votes_for_me > voters / 2;  // strictly more than half
    case QOU_ALL:
        return votes_for_me == voters;     // all nodes must be visible
    default:
        return true;                        // quorum disabled
    }
}
```

---

## 4. Where Quorum is Checked

```bash
grep -n "drbd_have_quorum\|quorum_lost\|suspended.*quorum\|DRBD_QUORUM" \
    drbd/drbd_state.c drbd/drbd_main.c | head -20
```

**On promotion attempt (`drbdadm primary`):**
```bash
grep -n "quorum\b" drbd/drbd_state.c | grep "is_valid\|check\|have_quorum" | head -10
```

Inside `is_valid_transition()`:
```c
// Cannot become Primary without quorum
if (role[NEW] == R_PRIMARY && !drbd_have_quorum(resource, R_PRIMARY))
    return SS_NO_QUORUM;
```

**On peer disconnect (might lose quorum):**

In `__after_state_change()`, when a connection drops:
```c
if (!drbd_have_quorum(resource, resource->role[NOW])) {
    // Lost quorum while Primary!
    switch (res_opts->on_no_quorum) {
    case ONQ_SUSPEND_IO:
        resource->susp_quorum = true;  // suspend I/O
        drbd_suspend_io(resource);
        break;
    case ONQ_IO_ERROR:
        // Return EIO to all in-flight and new I/O
        break;
    }
}
```

---

## 5. Two-Phase Commit for Role Changes — `P_TWOPC_*`

DRBD 9 uses a distributed two-phase commit for Primary promotions in multi-node clusters. This prevents two nodes from simultaneously promoting to Primary even with a partial partition.

```bash
grep -n "P_TWOPC_PREPARE\|P_TWOPC_COMMIT\|P_TWOPC_ABORT\|P_TWOPC_YES\|P_TWOPC_NO\|twopc" \
    drbd/drbd_protocol.h drbd/drbd_state.c | head -20
```

### The Protocol

```
Node A wants to become Primary:

Phase 1 — Prepare:
  A → all peers: P_TWOPC_PREPARE {tid=42, initiator=A, target_state=Primary}
  B → A: P_TWOPC_YES {tid=42}   (or P_TWOPC_NO if B objects)
  C → A: P_TWOPC_YES {tid=42}

Phase 2 — Commit (if all said YES):
  A → all peers: P_TWOPC_COMMIT {tid=42}
  A: applies state change locally

Phase 2 — Abort (if any said NO):
  A → all peers: P_TWOPC_ABORT {tid=42}
  A: does not change state
```

```bash
grep -n "struct p_twopc_request {" drbd/drbd_protocol.h
```

```c
struct p_twopc_request {
    struct p_header100 head;
    u32 tid;              // transaction ID (unique per prepare)
    u32 initiator_node_id;// which node is initiating
    u64 nodes_to_reach;  // bitmask: which nodes must participate
    union {
        struct {
            u64 primary_nodes; // which nodes become Primary
            u64 diskful_primary_nodes;
        };
        // ... other state change parameters
    };
} __packed;
```

---

## 6. `drbd_do_cluster_wide_state_change()` — The Initiator Side

```bash
grep -n "drbd_do_cluster_wide_state_change\b\|cluster_wide_state_change\b" \
    drbd/drbd_state.c | head -5
grep -n -A 150 "^static int drbd_do_cluster_wide_state_change\b" drbd/drbd_state.c
```

```c
static int drbd_do_cluster_wide_state_change(
    struct drbd_resource *resource,
    struct twopc_reply *reply,
    ...)
{
    u32 tid = ...;  // generate unique transaction ID

    // 1. Send P_TWOPC_PREPARE to all connected peers
    for_each_connection(connection, resource) {
        drbd_send_twopc_request(connection, P_TWOPC_PREPARE, tid, ...);
    }

    // 2. Wait for YES/NO from all participants
    // Timeout: twopc-timeout (default 100ms)
    wait_event_timeout(resource->twopc_wait,
                        all_replied || timeout,
                        twopc_timeout * HZ / 10);

    // 3a. All YES → commit
    if (all_replied && !any_no) {
        // Apply state change locally
        begin_state_change(resource, ...);
        // ... change state ...
        end_state_change(resource, ...);

        // Send P_TWOPC_COMMIT to peers
        for_each_connection(connection, resource)
            drbd_send_twopc_request(connection, P_TWOPC_COMMIT, tid, ...);
        return SS_SUCCESS;
    }

    // 3b. Any NO or timeout → abort
    for_each_connection(connection, resource)
        drbd_send_twopc_request(connection, P_TWOPC_ABORT, tid, ...);
    return SS_CONCURRENT_ST_CHG;  // or other error
}
```

---

## 7. Handling Incoming `P_TWOPC_PREPARE` — The Participant Side

```bash
grep -n -A 80 "^static int receive_twopc\b\|receive_twopc_prepare\b\|got_TwopcPrepare\b" \
    drbd/drbd_receiver.c drbd/drbd_state.c | head -90
```

```c
// On receiving P_TWOPC_PREPARE:
static int receive_twopc(struct drbd_connection *connection,
                          struct packet_info *pi)
{
    struct p_twopc_request *p = pi->data;

    // 1. Validate: is this a state change we can support?
    //    Would it violate our local constraints?
    rv = drbd_twopc_check_transition(resource, p);

    // 2a. Send YES
    if (rv == SS_SUCCESS) {
        drbd_send_twopc_reply(connection, P_TWOPC_YES, p->tid);
        // Remember we sent YES: if we don't hear COMMIT within timeout,
        // we must abort ourselves
        set_bit(TWOPC_PREPARED, &connection->flags);
    }

    // 2b. Send NO
    else {
        drbd_send_twopc_reply(connection, P_TWOPC_NO, p->tid);
    }

    return 0;
}
```

---

## 8. Fencing Mode — `fencing` Configuration

```bash
grep -n "fencing\|FP_DONT_CARE\|FP_RESOURCE\|FP_STONITH\|drbd_fence_peer_err_t\|drbd_set_role\b" \
    drbd/drbd_int.h drbd/drbd_state.c drbd/drbd_nl.c | head -20
```

### Resource-only fencing

When `fencing resource-only` is configured and a disk I/O error occurs:
```c
// drbd_req.c: WRITE_COMPLETED_WITH_ERROR
→ drbd_handle_failed_mirror() [or equivalent]
    → if (disk_conf->on_io_error == EP_DETACH)
           change_disk_state(device, D_FAILED, CS_HARD)
           → __after_state_change(): 
               if (resource->role[NOW] == R_PRIMARY)
                   change_role(resource, R_SECONDARY, CS_HARD)
                   // Demote! Primary with failed disk is unsafe.
```

### STONITH fencing

```bash
grep -n "stonith\|DRBD_FENCING_STONITH\|after_sb.*stonith\|drbd_fence_peer" \
    drbd/drbd_state.c drbd/drbd_nl.c | head -15
```

When STONITH is configured, DRBD calls a helper script:
```bash
grep -n "drbd_md_set_flag\|call_helper\|fencing.*helper" drbd/drbd_nl.c | head -10
```

```c
// drbd_nl.c: drbd_set_role() with fencing=stonith
if (test_bit(STONITH_OUTDATE_SELF, &connection->flags)) {
    // Call user-space fence-peer handler
    // e.g., /usr/lib/drbd/crm-fence-peer.9.sh
    drbd_kobject_uevent_env(device, AFTER_STATE_CHANGE, envp);
    // Handler must:
    // 1. STONITH the peer (ensure it's dead)
    // 2. Return 0 (success) or non-zero (failed to fence)
}
```

---

## 9. `susp_quorum` — I/O Suspension on Quorum Loss

```bash
grep -n "susp_quorum\b\|DRBD_IO_SUSP_QUORUM\|drbd_suspend_io\|drbd_resume_io" \
    drbd/drbd_int.h drbd/drbd_state.c drbd/drbd_main.c | head -20
```

When quorum is lost:
```c
// drbd_state.c: __after_state_change()
if (susp_quorum[NEW]) {
    // New I/O waits here
    drbd_suspend_io(resource, write_only);
    // write_only: only suspend writes, reads still work
}
```

When quorum returns:
```c
if (!susp_quorum[NEW] && susp_quorum[NOW]) {
    drbd_resume_io(resource);
    // Wakes all I/O that was suspended
    wake_up(&resource->io_wait);
}
```

Application I/O blocked on quorum loss:
```c
// drbd_req.c: drbd_make_request()
if (drbd_suspended(device)) {
    wait_event(device->resource->io_wait,
               !drbd_suspended(device));
}
```

---

## 10. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Read all TWOPC receive handlers
```bash
grep -n "receive_twopc\|P_TWOPC_\|got_TwopcYes\|got_TwopcNo\|got_TwopcCommit\|got_TwopcAbort" \
    drbd/drbd_receiver.c drbd/drbd_state.c | head -20
```
For each handler: what state change does it trigger, what bit does it set/clear, what does it send back?

### Exercise 2 (40 min): Trace a Primary promotion with 3 nodes (A, B, C)
Node A runs `drbdadm primary r0`. Node B and C are Secondary/Connected/UpToDate. Draw:
1. All P_TWOPC_PREPARE messages (A→B, A→C)
2. All P_TWOPC_YES replies (B→A, C→A)
3. P_TWOPC_COMMIT (A→B, A→C)
4. State changes on each node

### Exercise 3 (40 min): Find `drbd_have_quorum()` and trace for a 2+tiebreaker setup
```bash
grep -n -A 60 "drbd_have_quorum\b" drbd/drbd_state.c
```
With nodes A, B, tiebreaker T:
- Network partition: A can see T but not B. B can see T but not A.
- T connects to A first. Does A have quorum? Does B have quorum?

### Exercise 4 (30 min): Understand `on-no-quorum io-error` vs `suspend-io`
```bash
grep -n "ONQ_IO_ERROR\|ONQ_SUSPEND_IO\|on_no_quorum\b" drbd/drbd_int.h drbd/drbd_state.c | head -15
```
What is the behaviour difference from an application perspective? When would you choose each?

### Exercise 5 (40 min): Read `is_valid_transition()` for quorum checks
```bash
grep -n -A 200 "^static enum drbd_state_rv is_valid_transition\b" drbd/drbd_state.c
```
List every check that involves quorum or fencing. Under what conditions can a node be forced to Primary despite quorum loss (`CS_FORCE`)?

---

## Summary

DRBD's quorum system prevents split-brain in clusters with 3+ nodes. `drbd_have_quorum()` counts votes from connected peers; a majority (or all, for strict quorum) is required to remain Primary. Quorum loss triggers I/O suspension or EIO depending on `on-no-quorum` policy. Primary promotions in multi-node clusters use a distributed two-phase commit (`P_TWOPC_PREPARE` → `P_TWOPC_YES/NO` → `P_TWOPC_COMMIT/ABORT`) to ensure only one node can promote at a time. Fencing (`resource-only` or `stonith`) protects against Primary operation with a failed disk.

**Next:** Day 19 — DebugFS and `/proc/drbd`: what metrics are exposed, how they are generated, and using them for troubleshooting.
