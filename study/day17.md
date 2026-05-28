# Day 17 — UUID System: Generation, Comparison, Split-Brain Detection & Resync Decisions

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_main.c`, `drbd/drbd_receiver.c`, `drbd/drbd_int.h`, `drbd/linux/drbd.h`

---

## 1. What UUIDs Track

DRBD UUIDs are **not** universally unique identifiers in the RFC sense. They are 64-bit **write-generation tokens**: every time a node becomes Primary, it generates a new random UUID. By comparing UUID histories between nodes, DRBD can determine their data relationship without comparing actual data.

The UUID system answers: *"Did these two nodes ever share the same data, and if so, who diverged from whom?"*

---

## 2. The UUID Set Per Device

```bash
grep -n "UI_CURRENT\|UI_BITMAP\|UI_HISTORY_START\|UI_HISTORY_END\|UI_SIZE\|HISTORY_UUIDS" \
    drbd/linux/drbd.h drbd/drbd_int.h | head -20
```

```c
// linux/drbd.h
enum drbd_uuid_index {
    UI_CURRENT    = 0,   // UUID of the current generation
    UI_BITMAP     = 1,   // UUID of the last resync partner (bitmap UUID)
    UI_HISTORY_START = 2,// ring buffer of past current-UUIDs
    UI_HISTORY_END = 3,
    UI_SIZE       = 4,
    UI_FLAGS      = 4,   // flags stored here (overloaded)
};

#define HISTORY_UUIDS  (UI_HISTORY_END - UI_HISTORY_START + 1)
// Typically 2 history slots in older DRBD, up to 14 in DRBD 9 (P_UUIDS110)
```

In DRBD 9 with P_UUIDS110:
```bash
grep -n "DRBD_PEERS_MAX\|P_UUIDS110\|struct p_uuids110 {" drbd/drbd_protocol.h drbd/drbd_int.h | head -10
```

```c
struct p_uuids110 {
    struct p_header100 head;
    u64 current_uuid;
    u64 dirty_bits;       // bitmask: which peer slots have OOS bits
    u64 uuid_flags;       // flag bits (see below)
    u64 node_mask;        // which nodes are known
    u32 history_uuids_len;// number of history UUIDs following
    u64 history_uuids[HISTORY_UUIDS_MAX]; // up to 32 history entries
    // Per-peer bitmap UUIDs follow
} __packed;
```

---

## 3. UUID Generation Events

```bash
grep -n "drbd_uuid_new_current\b\|drbd_uuid_set_bm\b\|drbd_uuid_push_history\b\|drbd_uuid_set\b" \
    drbd/drbd_main.c drbd/drbd_receiver.c | head -20
```

### `drbd_uuid_new_current()` — New Primary Generation

```bash
grep -n -A 40 "^void drbd_uuid_new_current\b\|^static void drbd_uuid_new_current\b" drbd/drbd_main.c
```

Called when a node becomes Primary:

```c
void drbd_uuid_new_current(struct drbd_device *device, bool log_it)
{
    u64 val;

    // 1. Push current UUID into history ring buffer
    drbd_uuid_push_history(device, device->ldev->md.current_uuid);

    // 2. Generate new random UUID
    get_random_bytes(&val, sizeof(val));
    // Ensure non-zero (UUID=0 has special meaning: never been Primary)
    val |= 1;
    drbd_uuid_set_current(device, val);

    // 3. Set MDF_PRIMARY_IND flag in metadata
    device->ldev->md.flags |= MDF_PRIMARY_IND;

    // 4. Mark metadata dirty (will be written within 1 second)
    drbd_md_mark_dirty(device);

    // 5. Advertise new UUID to all peers
    drbd_send_uuids(device, 0, 0);
}
```

```bash
grep -n "drbd_uuid_new_current\b" drbd/drbd_state.c drbd/drbd_nl.c | head -10
# Called from: change_role() when role[NEW] == R_PRIMARY
```

### `drbd_uuid_set_bm()` — Bitmap UUID

```bash
grep -n -A 20 "^void drbd_uuid_set_bm\b\|drbd_uuid_set_bm\b" drbd/drbd_main.c | head -30
```

The bitmap UUID (`UI_BITMAP`) is set to the current UUID of the resync peer at the start of a resync. It serves as a "resync in progress" marker:

```c
// At start of resync (on SyncSource):
peer_device->bitmap_uuid = peer_device->current_uuid;
// peer_device->current_uuid = what the peer advertised
```

At end of resync:
```c
// drbd_resync_finished():
drbd_uuid_set_bm(peer_device, 0);   // clear bitmap UUID (resync done)
drbd_uuid_push_history(device, old_bitmap_uuid);  // save in history
```

### `drbd_uuid_push_history()` — History Ring Buffer

```bash
grep -n -A 25 "^void drbd_uuid_push_history\b\|^static void drbd_uuid_push_history\b" drbd/drbd_main.c
```

```c
void drbd_uuid_push_history(struct drbd_device *device, u64 val)
{
    struct drbd_md *md = &device->ldev->md;

    // Shift history: [0] becomes [1], [1] becomes [2], etc.
    memmove(&md->history_uuids[1], &md->history_uuids[0],
            (HISTORY_UUIDS - 1) * sizeof(u64));
    md->history_uuids[0] = val;

    drbd_md_mark_dirty(device);
}
```

---

## 4. UUID Exchange Protocol

```bash
grep -n "drbd_send_uuids\b\|drbd_send_uuids110\b" drbd/drbd_sender.c drbd/drbd_main.c | head -10
grep -n -A 60 "^int drbd_send_uuids110\b\|^static int drbd_send_uuids110\b" drbd/drbd_sender.c
```

The sender builds a `P_UUIDS110` packet containing:
- `current_uuid` — our current generation
- `dirty_bits` — bitmask of which peer slots have OOS bits
- `uuid_flags` — various flags (see below)
- `history_uuids[]` — our UUID history

```bash
grep -n "uuid_flags\|UUID_FLAG_DIRTY\|UUID_FLAG_CRASHED_PRIMARY\|UUID_FLAG_CONSISTENT\|UUID_FLAG_GOT_STABLE" \
    drbd/drbd_int.h drbd/drbd_main.c | head -20
```

| `UUID_FLAG_*` | Meaning in P_UUIDS110 |
|---|---|
| `UUID_FLAG_DIRTY` | We have OOS bits for this peer |
| `UUID_FLAG_CRASHED_PRIMARY` | We were Primary and crashed (no clean shutdown) |
| `UUID_FLAG_CONSISTENT` | Our data is consistent (no pending OOS) |
| `UUID_FLAG_GOT_STABLE` | Our disk state has stabilised |
| `UUID_FLAG_RESYNC` | We are requesting a resync |

---

## 5. UUID Comparison — `drbd_uuid_compare()`

This is the most important function in the UUID system. It determines the data relationship between two nodes:

```bash
grep -n -A 200 "^static int drbd_uuid_compare\b\|^int drbd_uuid_compare\b" drbd/drbd_receiver.c
```

The function returns an integer indicating the relationship:

```c
// Return values (search for the enum/defines):
// > 0 → we are ahead of peer (peer should become SyncTarget)
// < 0 → peer is ahead of us (we should become SyncTarget)
// == 0 → we are in sync (both UpToDate, no resync needed)
```

```bash
grep -n "drbd_uuid_compare\|AFTER_STATE_CHANGE\|drbd_determine_dev_size" drbd/drbd_receiver.c | head -10
```

The full comparison algorithm:

```
drbd_uuid_compare(peer_device, &hg, &rule_nr)
│
│ my_uuid   = device->ldev->md.current_uuid
│ peer_uuid = peer_device->p_uuid[UI_CURRENT]  (received in P_UUIDS110)
│
├─ CASE A: Both UUIDs are zero
│   └── First start ever, no data. hg = 0 (no sync needed — full sync will happen)
│
├─ CASE B: my_uuid == peer_uuid (exact match)
│   └── Same generation. hg = 0 (in sync)
│       But check bitmap UUIDs for ongoing/interrupted resync...
│
├─ CASE C: peer_uuid appears in my history_uuids[]
│   └── Peer's current UUID is something we used to have.
│       We are NEWER (primary longer). We are SyncSource.
│       hg = 1 (we send data to peer)
│
├─ CASE D: my_uuid appears in peer's history_uuids[]
│   └── Our current UUID is in peer's history.
│       Peer is NEWER. We are SyncTarget.
│       hg = -1 (peer sends data to us)
│
├─ CASE E: Bitmap UUIDs help disambiguate
│   │ my_bitmap_uuid   = md->history_uuids[UI_BITMAP]
│   │ peer_bitmap_uuid = peer_device->p_uuid[UI_BITMAP]
│   │
│   ├─ peer_uuid == my_bitmap_uuid
│   │   └── Peer's current = our bitmap UUID: peer is the node
│   │       we were resyncing WITH. We were SyncTarget mid-resync.
│   │       Resume: we are SyncTarget. hg = -1
│   │
│   └─ my_uuid == peer_bitmap_uuid
│       └── Our current = peer's bitmap UUID.
│           Peer was SyncTarget, we were SyncSource mid-resync.
│           Resume: we are SyncSource. hg = 1
│
└─ CASE F: No UUID match at all
    └── SPLIT BRAIN or completely unrelated data.
        hg = -1000 (extreme negative: split brain detected!)
        → Requires manual intervention (or auto-recovery policy)
```

---

## 6. Split-Brain Detection and Recovery

```bash
grep -n "split.brain\|split_brain\|DRBD_AFTER_SB_0P\|after.split.brain\|after_sb_0p" \
    drbd/drbd_receiver.c drbd/drbd_int.h | head -20
```

When `hg == -1000` (split brain), DRBD applies the configured policy:

```bash
grep -n "after_sb_0pri\|after_sb_1pri\|after_sb_2pri\|Discard\|DiscardRemote\|DiscardLocal\|Violently" \
    drbd/drbd_receiver.c drbd/drbd_int.h | head -20
```

```c
// Configuration options (from drbd.conf: net { after-sb-0pri disconnect; })
enum drbd_after_sb_p {
    ASB_DISCONNECT,          // disconnect; human must resolve
    ASB_DISCARD_YOUNGER_PRI, // discard the most recently promoted node's data
    ASB_DISCARD_OLDER_PRI,   // discard the least recently promoted node's data
    ASB_DISCARD_ZERO_CHG,    // discard the side with no changes (UUID unchanged)
    ASB_DISCARD_LEAST_CHG,   // discard the side with fewer changes
    ASB_DISCARD_LOCAL,       // always discard local data
    ASB_DISCARD_REMOTE,      // always discard peer data
    ASB_CONSENSUS,           // apply 0-primary policy to 1-primary situation
    ASB_VIOLENTLY,           // discard remote without checking (dangerous!)
    ASB_CALL_HELPER,         // call split-brain helper script
};
```

```bash
grep -n "drbd_asb_recover_0p\|drbd_asb_recover_1p\|drbd_asb_recover_2p" drbd/drbd_receiver.c | head -10
grep -n -A 60 "^static int drbd_asb_recover_0p\b" drbd/drbd_receiver.c
```

The three recovery functions correspond to how many nodes were Primary at time of split:
- `drbd_asb_recover_0p()` — neither was Primary: safest case
- `drbd_asb_recover_1p()` — one was Primary: apply 1-primary policy
- `drbd_asb_recover_2p()` — both were Primary: apply 2-primary policy (hardest)

---

## 7. `drbd_attach_handshake()` — Full UUID-Based Resync Decision

```bash
grep -n "drbd_attach_handshake\b\|drbd_handshake\b" drbd/drbd_receiver.c | head -5
grep -n -A 150 "^static int drbd_attach_handshake\b\|^static enum drbd_repl_state drbd_attach_handshake\b" \
    drbd/drbd_receiver.c
```

After UUID comparison, this function translates `hg` into initial `repl_state`:

```c
switch (hg) {
case 0:   // in sync
    if (drbd_bm_total_weight(peer_device) > 0)
        // Some OOS bits remain from a previous interrupted resync
        repl_state = L_WF_BITMAP_S;  // exchange bitmaps
    else
        repl_state = L_ESTABLISHED;  // fully in sync — no work to do
    break;

case 1:   // we are SyncSource
    repl_state = L_WF_BITMAP_S;
    break;

case -1:  // we are SyncTarget
    repl_state = L_WF_BITMAP_T;
    break;

case -1000:  // split brain
    drbd_alert(..., "Split-Brain detected!\n");
    // Apply split-brain resolution policy
    err = drbd_asb_recover_0p(peer_device);
    // Result: either continue with a winner/loser or disconnect
    break;
}
```

---

## 8. UUID Flags: `dirty_bits` and `node_mask`

DRBD 9 introduces per-peer OOS tracking in the UUID packet:

```bash
grep -n "dirty_bits\|node_mask\|UUID_FLAG" drbd/drbd_main.c drbd/drbd_receiver.c | head -20
```

```c
// drbd_send_uuids110():
// dirty_bits: one bit per peer node ID
// bit N set → we have OOS bits for peer node N
u64 dirty_bits = 0;
for_each_peer_device(peer_device, device) {
    if (drbd_bm_total_weight(peer_device) > 0)
        dirty_bits |= NODE_MASK(peer_device->connection->peer_node_id);
}
p->dirty_bits = cpu_to_be64(dirty_bits);
```

This allows a new node connecting to a 3-node cluster to learn exactly which nodes it is OOS with, without having to compare bitmaps for every pair.

---

## 9. Why UUIDs? Alternative Architectures for Data Lineage Tracking

The bitmap and AL track **what** is out of sync. The UUID system answers **whether** to sync, **which direction**, and **whether the situation is safe** (split-brain or not). These are three separate questions the bitmap cannot answer:

- **Who is authoritative?** Both sides may have OOS bits for each other — from an interrupted resync, or from two primaries writing divergent data. The bitmap looks identical in both cases.
- **Were we ever in sync?** Two fresh nodes with empty bitmaps look the same as two fully-synced nodes, yet one needs a full initial sync and the other needs nothing.
- **Split-brain detection?** Only UUID ancestry can distinguish "interrupted resync" from "independently written divergent data."

The following alternatives exist architecturally; each has reasons why it is not suitable for a kernel block device replication layer.

### 9.1 Vector Clocks

*Used by: Riak, Amazon Dynamo, CouchDB.*

Each node maintains one counter per peer. On write: increment own counter. On reconnect: compare vectors.

```
Node A: [A:5, B:3]   Node B: [A:3, B:7]
→ A is ahead on A-writes, B is ahead on B-writes → concurrent writes detected
```

**Advantage over UUID**: Detects *partial* divergence — can tell exactly which writes from which node are missing, not just "someone is ahead."

**Not suitable here**: Vector size grows O(N nodes). Each write must persist the vector — unacceptable overhead in a kernel block I/O path. Also, vectors give partial orders rather than a single "who wins" answer; conflict resolution becomes per-write rather than per-reconnect.

### 9.2 Merkle Trees

*Used by: Cassandra anti-entropy, ZFS, Git.*

Hash the device recursively in chunks. On reconnect, compare tree hashes top-down to find diverging subtrees.

**Advantage**: Would replace **both** UUID and bitmap with one structure — provides direction (who changed what) and content (which blocks differ) simultaneously.

**Not suitable here**: Maintaining the Merkle tree incrementally is O(log N) per write with significant memory overhead. For a 1 TiB device at 4 KiB granularity the tree has 256 M leaf nodes. Too expensive on the hot I/O path in a kernel module.

### 9.3 Sequential Term Numbers (Raft-style)

*Used by: Raft consensus, etcd, Paxos.*

Each Primary promotion increments a persistent integer counter. Higher term = more recent primary.

**Advantage over UUID**: Ordered — term 7 is unambiguously newer than term 5. No history ring buffer needed.

**Not suitable here**: Requires a coordinated, durable counter. In split-brain both nodes increment from the same base (both go 5 → 6), making terms ambiguous — the very scenario that needs to be detected. DRBD uses random UUIDs so that independently generated values are unique without coordination. The history ring also provides richer ancestry: not just "who is newer" but "are these lineages related at all?"

### 9.4 Write-Ahead Log / LSN

*Used by: PostgreSQL streaming replication, MySQL binlog.*

Every write is appended to a WAL with a monotonic log sequence number. On reconnect: find the common LSN, replay forward.

**Advantage**: Precise replay — exactly what changed and in what order.

**Not suitable here**: A block device has no semantic understanding of writes; there are no transaction boundaries to replay. The WAL grows unbounded and needs compaction. This is the database replication model and does not map to a general-purpose block device.

### 9.5 Physical Timestamps + Last-Write-Wins

*Used by: Cassandra LWW, some eventually-consistent stores.*

Each write gets a wall-clock timestamp; on conflict the higher timestamp wins silently.

**Not suitable here**: Clock skew makes timestamps unreliable between nodes. More critically, silent last-write-wins means split-brain data loss is **invisible** — no detection, no alert. DRBD explicitly refuses to proceed on detected split-brain because silently picking a winner risks undetected data corruption.

### 9.6 Why random UUID + history ring is the right fit

| Requirement | How UUID satisfies it |
|---|---|
| No coordination on generation | Random bytes — no counter synchronisation needed |
| Split-brain detection | UUID not in either node's history → definitely diverged |
| Direction determination | UUID in peer's history → O(H) scan, H ≤ 32 entries |
| Crash safety | Written to metadata before role change commits |
| Zero hot-path overhead | UUID changes only on Primary promotion, never per write |
| Kernel suitability | O(1) per write (push to ring), O(H) per reconnect |

The key design insight: UUID generation is tied to **role transitions**, not to individual writes. This means the per-write cost is zero. The bitmap handles per-block content tracking; the UUID handles per-generation lineage tracking. Each mechanism does exactly one job.

---

## 10. Hands-On Exercises (3–4 hours)

### Exercise 1 (60 min): Read `drbd_uuid_compare()` completely
```bash
grep -n -A 250 "^static int drbd_uuid_compare\b\|^int drbd_uuid_compare\b" drbd/drbd_receiver.c
```
For each `if` branch and `rule_nr` assignment:
1. What UUID relationship does this branch detect?
2. What value of `hg` is returned?
3. What real-world scenario does it correspond to?

### Exercise 2 (40 min): Trace UUID state across a full cycle
Given a two-node DRBD device, trace the UUID values through:
1. First ever bring-up (both UUIDs = 0 initially)
2. Node A becomes Primary: `drbd_uuid_new_current()` → UUID_A1 generated
3. Node A demotes, Node B promotes: UUID_B1 generated; UUID_A1 in B's history
4. Node B crashes, A reconnects: what does `drbd_uuid_compare()` return?

### Exercise 3 (35 min): Trace `drbd_asb_recover_1p()` for `after-sb-1pri discard-secondary`
```bash
grep -n -A 80 "^static int drbd_asb_recover_1p\b" drbd/drbd_receiver.c
```
Which node's data gets discarded? What state change is triggered?

### Exercise 4 (35 min): Find where UUIDs are written to disk and read back
```bash
grep -n "current_uuid\|md\.uuid\[UI_CURRENT\]\|UI_BITMAP" drbd/drbd_main.c | head -20
grep -n "drbd_md_sync\b\|drbd_md_read\b" drbd/drbd_main.c | head -10
```
Which fields of `struct meta_data_on_disk` hold the UUID data? How many bytes on disk per UUID set?

### Exercise 5 (30 min): Understand the `UUID_FLAG_CRASHED_PRIMARY` flow
```bash
grep -n "UUID_FLAG_CRASHED_PRIMARY\|MDF_CRASHED_PRIMARY\|crashed_primary" \
    drbd/drbd_main.c drbd/drbd_receiver.c | head -20
```
When is this flag set in `p_uuids110.uuid_flags`? How does the peer use it in its resync decision?

---

## Summary

DRBD UUIDs are 64-bit write-generation tokens. A new UUID is generated on every Primary promotion and pushed into a history ring buffer. The bitmap UUID records the resync partner's UUID to enable interrupted-resync resumption. `drbd_uuid_compare()` classifies the relationship between two nodes into: in-sync, I-am-ahead, I-am-behind, or split-brain. Split-brain triggers policy-based recovery (`after-sb-Npri` config). The `P_UUIDS110` packet carries the full UUID set plus per-peer dirty-bits and node-mask for efficient multi-node OOS determination.

**Next:** Day 18 — Quorum and fencing: `quorum` configuration, `tiebreaker`, and how DRBD prevents split-brain data corruption in 3+ node clusters.
