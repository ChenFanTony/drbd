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

// Phase 2: Commit — validate, apply [NEW]→[NOW], run callbacks
enum drbd_state_rv end_state_change(struct drbd_resource *resource,
                                     unsigned long *irq_flags,
                                     const char *tag)
{
    // 1. Sanitize [NEW] values (may auto-adjust)
    sanitize_state(resource);

    // 2. Validate the proposed transition
    rv = is_valid_transition(resource);
    if (rv < SS_SUCCESS) {
        abort_state_change(resource, irq_flags, tag);
        return rv;
    }

    // 3. Apply: copy [NEW] → [NOW] for all objects
    apply_state_change(resource);

    // 4. Unlock
    spin_unlock_irqrestore(&resource->req_lock, *irq_flags);

    // 5. Post-change callbacks (outside lock)
    __after_state_change(resource, ...);
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
          → begin_state_change(resource, &irq_flags, flags)
          → __change_role(resource, R_PRIMARY)
              → resource->role[NEW] = R_PRIMARY
          → end_state_change(resource, &irq_flags, "primary")
              → sanitize_state()
              → is_valid_transition() — checks quorum, peer states
              → apply_state_change() — role[NOW] = R_PRIMARY
              → __after_state_change()
                  → send P_STATE to all peers
                  → notify_role_change() → udev event
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

## 6. `apply_state_change()` — Writing [NEW] → [NOW]

```bash
grep -n -A 50 "^static void apply_state_change\b" drbd/drbd_state.c
```

This function iterates all objects and copies `[NEW]` to `[NOW]`:

```c
static void apply_state_change(struct drbd_resource *resource)
{
    struct drbd_connection *connection;
    struct drbd_device *device;
    struct drbd_peer_device *peer_device;

    // Apply resource-level states
    resource->role[NOW] = resource->role[NEW];
    resource->susp[NOW] = resource->susp[NEW];

    // Apply per-device states
    idr_for_each_entry(&resource->devices, device, vnr) {
        device->disk_state[NOW] = device->disk_state[NEW];
    }

    // Apply per-peer-device states
    for_each_connection(connection, resource) {
        connection->cstate[NOW] = connection->cstate[NEW];
        idr_for_each_entry(&connection->peer_devices, peer_device, vnr) {
            peer_device->repl_state[NOW] = peer_device->repl_state[NEW];
            peer_device->disk_state[NOW] = peer_device->disk_state[NEW];
        }
    }
}
```

---

## 7. `__after_state_change()` — Post-Change Actions

This is called **outside** the spinlock. It is where the real work happens after a state change:

```bash
grep -n -A 200 "^static void __after_state_change\b" drbd/drbd_state.c
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

## 10. Hands-On Exercises (3–4 hours)

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

---

## Summary

DRBD's state machine uses a two-phase begin/end commit pattern under `req_lock`. The `[NOW]`/`[NEW]` field pair allows multi-object atomic transitions. `sanitize_state()` enforces invariants automatically; `is_valid_transition()` rejects illegal requests with typed error codes. Post-change work (I/O resumption, peer notification, resync start) happens in `__after_state_change()` outside the lock. State changes propagate to peers via `P_STATE` packets and are processed by `receive_state()` on the remote side.

**Next:** Day 4 — The write request path: from `drbd_make_request()` to `bio_endio()`.
