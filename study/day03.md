# Day 3 — The State Machine: Transitions, Validation & Two-Phase Commit

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_state.c`, `drbd/drbd_state.h`, `drbd/drbd_int.h`

---

## 1. Three Orthogonal State Dimensions

Every DRBD volume is characterised at any moment by three independent states:

```
┌─────────────────────────────────────────────────────────────┐
│  resource: role       PRIMARY | SECONDARY | UNKNOWN         │
│  device:   disk_state Diskless→Attaching→UpToDate→Failed…   │
│  peer_dev: repl_state Off→WFBitMapS→SyncSource→Established… │
└─────────────────────────────────────────────────────────────┘
```

Find the enum definitions:
```bash
grep -n "enum drbd_role\b"        drbd/linux/drbd.h
grep -n "enum drbd_disk_state\b"  drbd/linux/drbd.h
grep -n "enum drbd_repl_state\b"  drbd/linux/drbd.h
grep -n "enum drbd_conn_state\b"  drbd/linux/drbd.h
```

### Full `drbd_disk_state` values (study each one):
```
D_DISKLESS       = 0   no local disk attached
D_ATTACHING      = 1   drbd_adm_attach() in progress
D_DETACHING      = 2   disk being removed
D_INCONSISTENT   = 3   data is not yet synchronised
D_OUTDATED       = 4   consistent but not current (peer was primary without us)
D_D_UNKNOWN      = 5   don't know yet (before initial connect)
D_CONSISTENT     = 6   consistent, not certain if current
D_UP_TO_DATE     = 7   fully synchronised, authoritative
D_FAILED         = 8   local I/O error occurred
D_NEGOTIATING    = 9   handshake in progress
```

### Full `drbd_repl_state` values:
```
L_OFF              = 0   not connected
L_ESTABLISHED      = 1   connected, data matches
L_STARTING_SYNC_S  = 2   about to become SyncSource
L_STARTING_SYNC_T  = 3   about to become SyncTarget
L_WF_BITMAP_S      = 4   waiting: will send bitmap to peer
L_WF_BITMAP_T      = 5   waiting: will receive bitmap from peer
L_WF_SYNC_UUID     = 6   exchanging UUIDs
L_SYNC_SOURCE      = 7   actively sending resync data
L_SYNC_TARGET      = 8   actively receiving resync data
L_VERIFY_S         = 9   online verify, local side
L_VERIFY_T         = 10  online verify, peer side
L_PAUSED_SYNC_S    = 11  resync paused (SyncSource side)
L_PAUSED_SYNC_T    = 12  resync paused (SyncTarget side)
L_AHEAD            = 13  congestion: primary stopped replicating
L_BEHIND           = 14  congestion: secondary falling behind
```

---

## 2. State Change Entry Points

All state changes funnel through a single machinery. Find the top-level functions:

```bash
grep -n "^enum drbd_state_rv\|^int change_\|^static.*change_" drbd/drbd_state.c | head -30
```

The two primary entry points:

```c
// For resource-level state changes (role, susp flags)
enum drbd_state_rv change_role(struct drbd_resource *resource,
                                enum drbd_role role,
                                enum chg_state_flags flags,
                                const char *tag,
                                struct sk_buff *reply_skb);

// For disk/replication state changes (called internally)
enum drbd_state_rv change_disk_state(struct drbd_device *device,
                                      enum drbd_disk_state disk_state,
                                      enum chg_state_flags flags,
                                      const char *tag,
                                      struct sk_buff *reply_skb);
```

---

## 3. The Two-Phase Commit Pattern

State changes in DRBD use an explicit begin/end pair. This lets multiple objects change state atomically under one lock acquisition:

```bash
grep -n "begin_state_change\|end_state_change\|abort_state_change" drbd/drbd_state.c | head -20
```

```c
// Phase 1: Begin — lock resource, set [NEW] fields
void begin_state_change(struct drbd_resource *resource,
                         unsigned long *irq_flags,
                         enum chg_state_flags flags)
{
    spin_lock_irqsave(&resource->req_lock, *irq_flags);
    // Copy [NOW] to [NEW] for all objects
    // So callers can read [NEW] and modify it
}

// Phase 2: Commit — validate, apply [NEW]→[NOW], queue post-change work
// end_state_change() (line 981) → __end_state_change() (line 964)
//   → ___end_state_change() (line 787) — does the real work
static enum drbd_state_rv ___end_state_change(struct drbd_resource *resource, ...)
{
    // 1. Validate + sanitize (inside try_state_change())
    rv = try_state_change(resource);   // calls sanitize_state() + is_valid_transition()
    if (rv < SS_SUCCESS) goto out;
    if (flags & CS_PREPARE) goto out;  // TWOPC prepare: don't apply yet

    // 2. Pre-apply housekeeping
    finish_state_change(resource, tag);

    // 3. Snapshot the change for the work item (before overwriting)
    work = alloc_after_state_change_work(resource);

    // 4. Inline [NEW] → [NOW] copy (drbd_state.c:827)
    smp_wmb();
    resource->role[NOW] = resource->role[NEW];
    for_each_connection(connection, resource) {
        connection->cstate[NOW] = connection->cstate[NEW];
        ...
    }
    idr_for_each_entry(&resource->devices, device, vnr) {
        device->disk_state[NOW] = device->disk_state[NEW];
        for_each_peer_device(peer_device, device) {
            peer_device->repl_state[NOW] = peer_device->repl_state[NEW];
            ...
        }
    }
    wake_up_all(&resource->state_wait);

    // 5. Queue post-change work (runs outside the lock in worker thread)
    queue_after_state_change_work(resource, done, work);
}
```

**Full trace of a Primary promotion:**
```bash
grep -n "drbd_adm_primary\|change_role\|begin_state_change\|end_state_change" \
    drbd/drbd_nl.c drbd/drbd_state.c | head -30
```

Call chain:
```
drbdadm primary r0
  → Netlink message → drbd_nl.c: drbd_adm_primary()
      → change_role(resource, R_PRIMARY, CS_VERBOSE, ...)
          → change_cluster_wide_state(do_change_role, ...)   [if peers exist]
              → begin_state_change(resource, &irq_flags, CS_LOCAL_ONLY)
              → __change_role(): resource->role[NEW] = R_PRIMARY
              → [TWOPC prepare/commit with peers — see Section 10]
              → end_state_change(resource, &irq_flags, "primary")
                  → ___end_state_change()
                      → try_state_change()    — sanitize + validate
                      → finish_state_change()
                      → role[NOW] = role[NEW] — inline copy (line 827)
                      → queue_after_state_change_work()
                          → w_after_state_change() [runs in worker thread]
                              → send P_STATE to all peers
                              → drbd_notify_peers() → udev event
                              → resume I/O if was suspended
```

---

## 4. `sanitize_state()` — Automatic State Adjustments

This function enforces invariants. If a caller sets an impossible combination, sanitize adjusts it:

```bash
grep -n -A 80 "^static void sanitize_state\b" drbd/drbd_state.c
```

Examples of invariants enforced:
```c
// If disk goes Failed, replication cannot be Established
if (disk_state[NEW] == D_FAILED && repl_state[NEW] == L_ESTABLISHED)
    repl_state[NEW] = L_OFF;

// If we're Primary and disk becomes Diskless, must fence
if (role[NEW] == R_PRIMARY && disk_state[NEW] == D_DISKLESS)
    set_bit(FORCE_DETACH, &device->flags);

// A diskless node cannot be UpToDate
if (disk_state[NEW] == D_DISKLESS && ...)
    disk_state[NEW] = D_DISKLESS; // already is, just ensure consistent
```

---

## 5. `is_valid_transition()` — Validation Rules

```bash
grep -n -A 100 "^static enum drbd_state_rv is_valid_transition\b" drbd/drbd_state.c
```

This function returns `SS_SUCCESS` or an error code explaining why the transition is illegal. Key checks:

```c
// Cannot become Primary if no UpToDate data
if (role[NEW] == R_PRIMARY && disk_state[NOW] < D_UP_TO_DATE
    && !allow_two_primaries)
    return SS_NO_UP_TO_DATE_DISK;

// Cannot detach while I/O is in flight
if (disk_state[NEW] == D_DISKLESS && atomic_read(&device->local_cnt) > 0)
    return SS_LOWER_THAN_OUTDATED;  // actually: SS_DEVICE_IN_USE

// Resync cannot start without a connected peer
if (repl_state[NEW] == L_SYNC_SOURCE && cstate[NOW] < C_CONNECTED)
    return SS_NO_NET_CONFIG;
```

The error codes:
```bash
grep -n "SS_\|enum drbd_state_rv" drbd/linux/drbd.h drbd/drbd_state.h | head -40
```

---

## 6. `___end_state_change()` — Applying [NEW] → [NOW] (`drbd_state.c:787`)

There is no separate `apply_state_change()` function. The `[NEW]→[NOW]` copy is done **inline** inside `___end_state_change()`. Find it:

```bash
grep -n -A 120 "^static enum drbd_state_rv ___end_state_change\b" drbd/drbd_state.c | head -120
```

The copy sequence (lines 827–878):

```c
smp_wmb();   // ensure state is visible before anything that depends on it

// Resource-level
resource->role[NOW]        = resource->role[NEW];
resource->susp_user[NOW]   = resource->susp_user[NEW];
resource->susp_nod[NOW]    = resource->susp_nod[NEW];
resource->susp_quorum[NOW] = resource->susp_quorum[NEW];
resource->fail_io[NOW]     = resource->fail_io[NEW];

// Per-connection
for_each_connection(connection, resource) {
    connection->cstate[NOW]    = connection->cstate[NEW];
    connection->peer_role[NOW] = connection->peer_role[NEW];
    connection->susp_fen[NOW]  = connection->susp_fen[NEW];
}

// Per-device and per-peer-device
idr_for_each_entry(&resource->devices, device, vnr) {
    device->disk_state[NOW]    = device->disk_state[NEW];
    device->have_quorum[NOW]   = device->have_quorum[NEW];
    for_each_peer_device(peer_device, device) {
        peer_device->disk_state[NOW]  = peer_device->disk_state[NEW];
        peer_device->repl_state[NOW]  = peer_device->repl_state[NEW];
        peer_device->resync_susp_user[NOW] = ...;
        ...
    }
}

wake_up_all(&resource->state_wait);
queue_after_state_change_work(resource, done, work);
```

The three-level call chain: `end_state_change()` (line 981) → `__end_state_change()` (line 964) → `___end_state_change()` (line 787).

---

## 7. `w_after_state_change()` — Post-Change Actions (`drbd_state.c:3822`)

There is no `__after_state_change()` function. Post-change side effects run in the **worker thread** via a queued work item. The work item is allocated in `___end_state_change()` (before the state is applied), then dispatched to the resource's worker thread:

```bash
grep -n -A 200 "^static int w_after_state_change\b" drbd/drbd_state.c | head -200
```

Key actions triggered by specific transitions:

| Transition | Action in `__after_state_change` |
|---|---|
| `role: Secondary → Primary` | Resume I/O, send P_STATE to peers, fire udev event |
| `disk: Inconsistent → UpToDate` | Call `after-resync-target` hook, wake writers |
| `repl: Established → Off` | Resume suspended I/O, complete timed-out requests |
| `repl: Off → WFBitMapT` | Start bitmap exchange |
| `disk: * → Failed` | Schedule detach, set OOS bits for all peers |
| `repl: * → SyncSource` | Arm resync_timer, queue w_make_resync_request |

---

## 8. State Change Flags (`chg_state_flags`)

```bash
grep -n "enum chg_state_flags\|CS_VERBOSE\|CS_HARD\|CS_LOCAL_ONLY\|CS_SERIALIZE" \
    drbd/drbd_state.h drbd/drbd_int.h
```

| Flag | Meaning |
|---|---|
| `CS_VERBOSE` | Log the state change to kernel ring buffer |
| `CS_HARD` | Force transition, skip some safety checks |
| `CS_LOCAL_ONLY` | Don't send P_STATE to peers |
| `CS_SERIALIZE` | Hold `adm_mutex` across the change |
| `CS_ALREADY_SERIALIZED` | Caller already holds adm_mutex |
| `CS_DONT_RETRY` | Don't retry if state is busy |
| `CS_WAIT_COMPLETE` | Wait for all post-change work to finish |

---

## 9. State Propagation to Peers

When local state changes, DRBD sends `P_STATE` or `P_PEER_ACK` packets:

```bash
grep -n "drbd_send_state\b" drbd/drbd_state.c drbd/drbd_sender.c
grep -n "P_STATE\b" drbd/drbd_protocol.h drbd/drbd_receiver.c
```

On the peer, `receive_state()` in `drbd_receiver.c` processes the incoming P_STATE:
```bash
grep -n -A 60 "^static int receive_state\b" drbd/drbd_receiver.c
```

This function:
1. Reads the peer's advertised `role`, `disk_state`, `repl_state`
2. Calls `begin_state_change()` to update the local peer_device's `disk_state[NEW]`
3. May trigger further local transitions (e.g. starting resync if peer is UpToDate and we are not)

---

## 10. Distributed TWOPC: Cluster-Wide State Changes

Section 3 describes the **local** begin/end commit pattern (spinlock, [NEW]/[NOW] fields). That is only half the story. For changes that must be agreed upon by all nodes — becoming Primary, connecting/disconnecting a peer, or resizing a device — DRBD runs a **distributed two-phase commit** over the network.

### 10.1 When Is Distributed TWOPC Used?

`change_cluster_wide_state()` (`drbd_state.c:4929`) decides:
- If the change is `CS_LOCAL_ONLY` → purely local, no TWOPC.
- If `try_state_change()` returns `SS_NOTHING_TO_DO` → nothing to propagate.
- Otherwise → distributed TWOPC.

Examples that trigger distributed TWOPC:
| Operation | Function |
|---|---|
| `drbdadm primary` | `change_role()` → `change_cluster_wide_state()` |
| Node connect/disconnect | `change_cstate_es()` |
| Device resize | `change_cluster_wide_device_size()` |

### 10.2 Packet Types

```
Initiator → Peers:   P_TWOPC_PREPARE   (phase 1: can you do this?)
                     P_TWOPC_PREP_RSZ  (phase 1 variant for resize)

Peers → Initiator:   P_TWOPC_YES       (I consent)
                     P_TWOPC_NO        (I refuse)
                     P_TWOPC_RETRY     (I'm busy, try again)

Initiator → Peers:   P_TWOPC_COMMIT    (phase 2: do it)
                     P_TWOPC_ABORT     (phase 2: don't)
```

### 10.3 Initiator Side: `change_cluster_wide_state()` (`drbd_state.c:4929`)

```
change_cluster_wide_state()
  │
  ├─ begin_state_change(CS_LOCAL_ONLY)   ← lock resource, set [NEW] fields
  ├─ change(context, PH_PREPARE)         ← populate mask/val
  ├─ try_state_change()                  ← validate locally (dry run)
  │
  ├─ Generate random tid
  ├─ Set request.cmd = P_TWOPC_PREPARE
  ├─ Set request.nodes_to_reach = all nodes not directly connected
  │
  ├─ begin_remote_state_change()         ← mark resource->remote_state_change = true
  ├─ __cluster_wide_request()            ← send P_TWOPC_PREPARE to all direct peers
  │     sets TWOPC_PREPARED bit per connection
  │
  ├─ wait_event_interruptible_timeout()  ← wait for cluster_wide_reply_ready()
  │     (ack_receiver thread calls got_twopc_reply() which wakes us)
  │
  ├─ get_cluster_wide_reply()            ← any TWOPC_NO? → SS_CW_FAILED_BY_PEER
  │                                         any TWOPC_RETRY? → SS_CONCURRENT_ST_CHG
  │                                         all TWOPC_YES? → SS_CW_SUCCESS
  │
  ├─ [on success] request.cmd = P_TWOPC_COMMIT
  ├─ [on failure] request.cmd = P_TWOPC_ABORT
  │
  ├─ end_remote_state_change()
  ├─ change(context, PH_COMMIT) + end_state_change()  ← apply [NEW]→[NOW] locally
  └─ twopc_phase2()                      ← send P_TWOPC_COMMIT or P_TWOPC_ABORT to peers
```

Key data: `resource->twopc_reply` accumulates votes; `resource->twopc.state_change.{mask,val}` carry the proposed change.

### 10.4 Participant Side: `process_twopc()` (`drbd_receiver.c:7465`)

When a node receives `P_TWOPC_PREPARE`:

```
receive_twopc()
  └─ process_twopc()
       │
       ├─ check_concurrent_transactions()  ← is another TWOPC in progress?
       │     CSC_CLEAR:       no conflict → proceed
       │     CSC_MATCH:       duplicate → resend last reply
       │     CSC_ABORT_LOCAL: local TWOPC must yield → abort_local_transaction()
       │     CSC_REJECT:      cannot yield → send P_TWOPC_RETRY
       │
       ├─ Set resource->remote_state_change = true
       ├─ Store received mask/val in resource->twopc.state_change
       │
       ├─ [P_TWOPC_PREPARE path]
       │     flags |= CS_PREPARE
       │     Decode state_change->mask, state_change->val from packet
       │     Build reply.primary_nodes (am I Primary?)
       │
       ├─ change_peer_device_state() or change_connection_state() or far_away_change()
       │     ← runs begin_state_change(CS_PREPARE | CS_LOCAL_ONLY)
       │        try_state_change() ← validate (but don't apply)
       │        If OK → reply P_TWOPC_YES; If fail → reply P_TWOPC_NO
       │
       ├─ arm twopc_timer (timeout → auto-abort)
       │
       └─ nested_twopc_request()   ← forward to nodes in nodes_to_reach bitmap
            └─ conn_send_twopc_request() for each indirect neighbor

When P_TWOPC_COMMIT arrives:
       │
       ├─ flags |= CS_PREPARED  (match existing prepared transaction)
       ├─ change_peer_device_state(CS_PREPARED) ← apply [NEW]→[NOW]
       ├─ timer_delete(&resource->twopc_timer)
       └─ nested_twopc_request()   ← forward COMMIT to indirect neighbors
```

### 10.5 Vote Collection: `got_twopc_reply()` (`drbd_receiver.c:10506`)

This runs in the **ack_receiver thread** (separate from the initiator thread):

```c
// Per connection, set exactly one bit:
if (P_TWOPC_YES)   set_bit(TWOPC_YES,   &connection->flags);
if (P_TWOPC_NO)    set_bit(TWOPC_NO,    &connection->flags);
if (P_TWOPC_RETRY) set_bit(TWOPC_RETRY, &connection->flags);

drbd_maybe_cluster_wide_reply(resource);  // wake initiator if all replied
```

`cluster_wide_reply_ready()` (`drbd_state.c:4555`) scans all connections with `TWOPC_PREPARED` set and returns `true` when all have replied YES, or any has replied NO/RETRY.

### 10.6 Indirect Routing (Non-Fully-Connected Clusters)

DRBD clusters need not be fully connected (e.g., A↔B and B↔C but not A↔C). The `nodes_to_reach` field (64-bit bitmask) handles this:

```
Initiator A sends P_TWOPC_PREPARE to B with nodes_to_reach = {C}

B receives it:
  ├─ Votes YES to A directly (P_TWOPC_YES → A)
  └─ Forwards P_TWOPC_PREPARE to C (after removing B from nodes_to_reach)

C votes YES to B (P_TWOPC_YES → B)

B forwards C's reply upstream (nested_twopc_work → twopc_end_nested)
```

`nested_twopc_abort()` (`drbd_receiver.c:7303`) handles forwarding of P_TWOPC_ABORT to nodes that were only reachable indirectly.

### 10.7 Concurrent Transactions and Retry

Two nodes can simultaneously initiate TWOPC (e.g., A tries to become Primary while B tries to connect). `check_concurrent_transactions()` (`drbd_state.c`) uses the `tid` and `initiator_node_id` to detect this:

- **Same initiator, different TID**: `CSC_TID_MISS` → one must wait or send `P_TWOPC_RETRY`
- **Different initiator**: `CSC_REJECT` → refuse with `P_TWOPC_RETRY`
- **Local TWOPC conflicts**: `CSC_ABORT_LOCAL` → abort local, yield to remote

On retry, `change_cluster_wide_state()` uses exponential backoff (`twopc_retry_timeout()`):
```c
if (rv == SS_TIMEOUT || rv == SS_CONCURRENT_ST_CHG) {
    long timeout = twopc_retry_timeout(resource, retries++);
    schedule_timeout_interruptible(timeout);
    goto retry;  // drbd_state.c:5205
}
```

### 10.8 Full Example: `drbdadm primary r0` on Node A

```
NodeA (initiator)               NodeB (participant)
─────────────────               ──────────────────
change_role()
  change_cluster_wide_state()
    tid = random()
    P_TWOPC_PREPARE ──────────→ receive_twopc()
    (mask=role, val=Primary)       process_twopc()
                                     check_concurrent_transactions() → CSC_CLEAR
                                     change_peer_device_state(CS_PREPARE)
                                     try_state_change() → SS_SUCCESS
                                     arm twopc_timer
    ←──────────────── P_TWOPC_YES  (primary_nodes=0, no Primary on B)
    got_twopc_reply()
      set_bit(TWOPC_YES, &connB->flags)
      cluster_wide_reply_ready() → true
    wait returns
    get_cluster_wide_reply() → SS_CW_SUCCESS
    (no TWOPC_NO, no TWOPC_RETRY)
    request.cmd = P_TWOPC_COMMIT
    end_remote_state_change()
    change(PH_COMMIT)
    end_state_change()          (role[NOW] = R_PRIMARY on A)
    twopc_phase2():
    P_TWOPC_COMMIT ───────────→ process_twopc()
                                   flags = CS_PREPARED
                                   change_connection_state(CS_PREPARED)
                                   apply_state_change()  ← peer's role[NOW] = R_PRIMARY
                                   timer_delete()
                                   nested_twopc_request() (no further nodes)
```

### 10.9 Key Differences: Local vs Distributed Commit

| Aspect | Local (`begin/end_state_change`) | Distributed (`change_cluster_wide_state`) |
|---|---|---|
| Scope | Single node | All reachable nodes |
| Lock | `state_rwlock` spinlock | `state_rwlock` + network round-trip |
| Packets | None | `P_TWOPC_PREPARE` → votes → `P_TWOPC_COMMIT` |
| Failure | Immediate `SS_*` error | `SS_CW_FAILED_BY_PEER` or `SS_TIMEOUT` |
| Retry | No | Yes (exponential backoff) |
| Flag | `CS_LOCAL_ONLY` to bypass | None (default path) |

---

## 11. Hands-On Exercises (3–4 hours)

### Exercise 1 (45 min): Full read of `drbd_state.c`
Open the file, read every function. For each function write its name + one sentence purpose in a notebook. Estimated: ~1500 lines.

### Exercise 2 (40 min): Trace a disk attach
```bash
grep -n "drbd_adm_attach\b" drbd/drbd_nl.c
```
Starting from `drbd_adm_attach()`, trace every call to `begin_state_change` / `end_state_change`. What states change? What are the intermediate states `Attaching` → `Negotiating` → `UpToDate` or `Inconsistent`?

### Exercise 3 (30 min): Identify all `SS_` error return codes
```bash
grep -n "SS_[A-Z_]\+" drbd/linux/drbd.h | grep -v "//"
```
For each error code, write when it would be returned and what a user would see in `dmesg`.

### Exercise 4 (45 min): Read `sanitize_state()` completely
For every `if` block in `sanitize_state()`, write: "If condition X is true, force state Y because invariant Z must hold."

### Exercise 5 (30 min): Trace `P_STATE` send and receive
```bash
# Where is P_STATE built and sent?
grep -n "drbd_send_state\|P_STATE" drbd/drbd_sender.c drbd/drbd_state.c

# Where is P_STATE received and processed?
grep -n "receive_state\b" drbd/drbd_receiver.c
```
Draw the message exchange for a node coming back online and triggering resync.

### Exercise 6 (45 min): Trace a full distributed TWOPC
```bash
# Initiator path
grep -n "change_cluster_wide_state\b" drbd/drbd_state.c | head -5
grep -n -A 350 "^change_cluster_wide_state\b" drbd/drbd_state.c | head -350

# Participant path
grep -n -A 380 "^static void process_twopc\b" drbd/drbd_receiver.c | head -380

# Vote collection
grep -n -A 60 "^static int got_twopc_reply\b" drbd/drbd_receiver.c | head -60
```
1. Starting from `change_role()`, trace to `P_TWOPC_PREPARE` being sent.
2. On the receiving node, trace from `receive_twopc()` to `P_TWOPC_YES`.
3. Back on the initiator, trace from `got_twopc_reply()` to `P_TWOPC_COMMIT`.
4. Draw a message-sequence diagram for a 3-node cluster where one node is unreachable via indirect routing.

### Exercise 7 (30 min): Concurrent TWOPC
```bash
grep -n "check_concurrent_transactions\|CSC_ABORT_LOCAL\|CSC_REJECT\|CSC_TID_MISS\|abort_local_transaction" \
    drbd/drbd_state.c drbd/drbd_receiver.c
```
For each of the four `csc_rv` outcomes (`CSC_CLEAR`, `CSC_MATCH`, `CSC_ABORT_LOCAL`, `CSC_REJECT`/`CSC_TID_MISS`), write the scenario that triggers it and the resulting action.

---

## Summary

DRBD's state machine has two layers. The **local layer** uses a begin/end commit pair under `state_rwlock`: `begin_state_change()` locks and copies states to `[NEW]`; `sanitize_state()` enforces invariants; `is_valid_transition()` rejects illegal changes; `apply_state_change()` writes `[NEW]→[NOW]`; `__after_state_change()` triggers side effects outside the lock.

The **distributed layer** runs a network-level two-phase commit for cluster-wide changes. The initiator calls `change_cluster_wide_state()`, sends `P_TWOPC_PREPARE` to all reachable nodes (with `nodes_to_reach` bitmask for indirect forwarding), collects `P_TWOPC_YES`/`P_TWOPC_NO`/`P_TWOPC_RETRY` votes, then sends `P_TWOPC_COMMIT` or `P_TWOPC_ABORT`. Participants receive and validate locally before voting. Concurrent transactions are detected and resolved via retry with exponential backoff.

**Next:** Day 4 — The write request path: from `drbd_make_request()` to `bio_endio()`.
