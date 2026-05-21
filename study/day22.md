# Day 22 — Multi-Node Topology: 3+ Node Clusters, Fan-Out Replication & `peer_node_id`

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_main.c`, `drbd/drbd_nl.c`, `drbd/drbd_state.c`, `drbd/drbd_int.h`, `drbd/drbd_receiver.c`

---

## 1. DRBD 9 Topology Model

DRBD 9 supports up to 32 nodes per resource (vs DRBD 8's 2-node limit). Each node maintains a **direct TCP connection to every other node** — a full mesh:

```
3-node cluster (A, B, C):
    A ←──────→ B
    A ←──────→ C
    B ←──────→ C

Each arrow = one drbd_connection with sender + receiver + worker threads
Each drbd_connection = 2 TCP sockets (DATA + META)

For N nodes: N*(N-1)/2 total connections in the cluster
For 3 nodes: 3 connections total
For 5 nodes: 10 connections total
```

Each node identifies itself and peers with a **node ID** (0–31):

```bash
grep -n "node_id\|peer_node_id\|DRBD_NODE_ID\|my_node_id" drbd/drbd_int.h drbd/drbd_nl.c | head -20
```

```c
// drbd_int.h
struct drbd_resource {
    ...
    int node_id;   // this node's ID in the cluster (from drbd.conf: node-id)
    ...
};

struct drbd_connection {
    ...
    int peer_node_id;  // the peer's node ID
    ...
};
```

---

## 2. How Node IDs Are Assigned

```bash
grep -n "node.id\|node_id\b" drbd/drbd-headers/linux/drbd_genl.h drbd/drbd_nl.c | head -20
```

From `drbd.conf`:
```
resource r0 {
  node-id 0;              ← this node's ID

  on node-A {
    node-id 0;
    address 10.0.0.1:7789;
  }
  on node-B {
    node-id 1;
    address 10.0.0.2:7789;
  }
  on node-C {
    node-id 2;
    address 10.0.0.3:7789;
  }

  connection {
    host node-A;
    host node-B;
  }
  connection {
    host node-A;
    host node-C;
  }
  connection {
    host node-B;
    host node-C;
  }
}
```

At connection establishment, `P_CONNECTION_FEATURES` exchanges node IDs:

```bash
grep -n "struct p_connection_features {" drbd/drbd-headers/drbd_protocol.h
grep -n "my_node_id\|peer_node_id\|node_id" drbd/drbd-headers/drbd_protocol.h | head -10
```

```c
struct p_connection_features {
    u32 protocol_min;
    u32 protocol_max;
    u32 sender_node_id;   // my node ID → peer learns our ID from this
    u32 receiver_node_id; // who we think the peer is
    u32 feature_flags;
    u32 volume_max;       // max volumes this connection supports
} __packed;
```

---

## 3. Fan-Out Write Replication

On a Primary node with N-1 peers, every application write is sent to ALL connected peers simultaneously:

```bash
grep -n "for_each_peer_device\b" drbd/drbd_req.c | head -15
```

```c
// drbd_req.c: drbd_process_write_request() or __drbd_make_request()
for_each_peer_device(peer_device, device) {
    if (peer_device->repl_state[NOW] < L_ESTABLISHED)
        continue;

    // Set per-peer net_rq_state bits
    int idx = peer_device->node_id;
    req->net_rq_state[idx] |= RQ_NET_PENDING;
    if (nc->wire_protocol == DRBD_PROT_C)
        req->net_rq_state[idx] |= RQ_EXP_WRITE_ACK;
    atomic_inc(&req->completion_ref);

    // Enqueue for the sender thread of THIS connection
    drbd_queue_write(peer_device, req);
}
```

**Per-peer state** is tracked in the `drbd_request` struct as two separate fields (NOT a single combined array):

```bash
grep -n "local_rq_state\|net_rq_state\[" drbd/drbd_int.h drbd/drbd_req.h | head -10
```

```c
// drbd_int.h — actual struct in DRBD 9.2
struct drbd_request {
    ...
    spinlock_t rq_lock;                       // per-request lock
    unsigned int local_rq_state;              // local I/O state bits
    u16 net_rq_state[DRBD_NODE_ID_MAX];       // per-peer (32 slots, indexed by node ID)
    ...
};
```

```bash
grep -n "DRBD_NODE_ID_MAX\b\|DRBD_PEERS_MAX\b" drbd/drbd-headers/linux/drbd.h
# DRBD_PEERS_MAX = 32; DRBD_NODE_ID_MAX = DRBD_PEERS_MAX
```

`bio_endio()` is called only when `req->completion_ref` (an atomic refcount) drops to zero — that ref is incremented per pending peer (and once for local I/O), decremented on each completion.

---

## 4. Per-Peer Bitmap Slots

Each connection to a peer has its own bitmap slot:

```bash
grep -n "bitmap_index\b\|peer_device->bitmap_index" drbd/drbd_int.h drbd/drbd_bitmap.c | head -10
```

```c
struct drbd_peer_device {
    ...
    int bitmap_index;    // which slot in drbd_bitmap.bm_pages[] belongs to this peer
    ...
};
```

Assignment during connection setup:

```bash
grep -n "bitmap_index\s*=" drbd/drbd_nl.c drbd/drbd_main.c | head -10
```

The bitmap index assignment is stable (derived from `peer_node_id`) so that the correct OOS bits survive across reboots:

```c
// bitmap_index == peer_node_id (in most DRBD 9 implementations)
// Or assigned from metadata per-peer UUID slot
peer_device->bitmap_index = peer_device->connection->peer_node_id;
```

---

## 5. The `nodes_to_reach` Bitmask — Cluster Membership

```bash
grep -n "nodes_to_reach\b\|NODE_MASK\b\|node_mask\b" drbd/drbd_int.h drbd/drbd_state.c drbd/drbd-headers/drbd_protocol.h | head -20
```

```c
#define NODE_MASK(id)  (1ULL << (id))

// In P_TWOPC_PREPARE and P_UUIDS110:
u64 nodes_to_reach;    // bitmask: which node IDs must participate
u64 node_mask;         // bitmask: which nodes are currently known/connected
```

**Example:** 3-node cluster (IDs 0, 1, 2), all connected:
```
nodes_to_reach = 0b111 = 7
node_mask      = 0b111 = 7
```

If node 2 disconnects:
```
node_mask      = 0b011 = 3   (only nodes 0 and 1 visible)
```

---

## 6. Forwarding Writes Through an Intermediary

In a 3-node cluster A→B→C, if C is only connected to B (not to A):

```
A (Primary) writes:
  → P_DATA to B  (direct connection A↔B)
  B → P_DATA to C  (B forwards to C on behalf of A)
```

This **write forwarding** is handled by the receiver on B:

```bash
grep -n "forward\|DRBD_PROT_C.*forward\|drbd_forward_write\b" \
    drbd/drbd_receiver.c drbd/drbd_req.c | head -15
```

In `receive_Data()` on B (acting as intermediary):
```c
// If this peer_device's repl_state means B should forward to C:
if (peer_device->repl_state[NOW] == L_ESTABLISHED &&
    connection->agreed_pro_version >= 110) {
    // Forward the write to any peers that don't have a direct connection to A
    for_each_peer_device(fwd_peer_device, device) {
        if (fwd_peer_device == peer_device)
            continue;
        if (fwd_peer_device->repl_state[NOW] != L_ESTABLISHED)
            continue;
        drbd_send_dblock(fwd_peer_device, ...);
    }
}
```

```bash
grep -n "drbd_forward_write\b\|forward.*write\|P_FORWARDED_DATA\b" \
    drbd/drbd_receiver.c drbd/drbd-headers/drbd_protocol.h | head -10
```

---

## 7. `dagtag` Write Ordering in Multi-Node Clusters

**dagtag** = **data generation tag**. It is a 64-bit value in units of **512-byte
sectors**, monotonically increasing per resource.  It is *not* a simple per-request
counter — it advances by the size of each write.

```bash
grep -n "dagtag_sector\b" drbd/drbd_int.h | head -10
```

```c
// drbd_int.h: struct drbd_resource
u64 dagtag_sector;     // protected by tl_update_lock; advances by write size

// drbd_int.h: struct drbd_request
u64 dagtag_sector;     // this request's endpoint in the global sector stream

// drbd_int.h: struct drbd_connection
atomic64_t last_dagtag_sector;  // latest dagtag seen from this peer
```

### Assignment (drbd_req.c:1994)

```c
// For writes: advance the resource counter by the write's size
WRITE_ONCE(resource->dagtag_sector,
           resource->dagtag_sector + (req->i.size >> 9));
// For reads: no advance — just snapshot the current value
req->dagtag_sector = resource->dagtag_sector;
```

Write A (4K) gets `dagtag = 8`, write B (4K) gets `dagtag = 16`, etc.  The
dagtag encodes both **ordering** and **size** in one number.

### Use 1: `P_DAGTAG` — gap filling on the sender side

Each connection's sender tracks `connection->send.current_dagtag_sector`.
Before sending a write, it checks whether the write's *start* dagtag matches
where the connection currently is (`drbd_sender.c:3517`):

```c
u64 current_dagtag_sector = req->dagtag_sector - (req->i.size >> 9);
if (current_dagtag_sector != connection->send.current_dagtag_sector)
    drbd_send_dagtag(connection, current_dagtag_sector);
connection->send.current_dagtag_sector = req->dagtag_sector;
```

A **gap** appears when some requests are not sent on this connection (reads, or
writes going only to a subset of peers).  `P_DAGTAG` fills the gap so the
receiver's `last_dagtag_sector` stays accurate.

The receiver handler simply stores the value (`drbd_receiver.c:8833`):
```c
static int receive_dagtag(struct drbd_connection *connection, ...)
{
    set_connection_dagtag(connection, be64_to_cpu(p->dagtag));
    // → atomic64_set(&connection->last_dagtag_sector, dagtag)
    // → release_dagtag_wait(...)  ← wake any resync reads waiting on this
    return 0;
}
```

### Use 2: `depend_dagtag` — resync safety in 3-node clusters

When a resync source (e.g. NodeB) asks a target (NodeC) to read a block, it
attaches a `depend_dagtag` to the request (`send_resync_request`,
`drbd_sender.c:431`):

```c
dagtag_result = find_current_dagtag(resource);
// → if we are primary: our own resource->dagtag_sector
// → if secondary: last_dagtag_sector from the connected primary
```

The target (NodeC) receives this via `depend_dagtag` in the peer_req.
`drbd_peer_resync_read()` (`drbd_receiver.c:3682`) checks:

```c
if (peer_req->depend_dagtag &&
    need_to_wait_for_dagtag_of_peer_request(peer_req)) {
    // NodeC's last_dagtag_sector from the primary < depend_dagtag
    // → NodeC hasn't received all primary writes yet
    list_add_tail(&peer_req->w.list, &connection->dagtag_wait_ee);
    return;   // parked
}
```

When a subsequent `P_DATA` or `P_DAGTAG` advances `last_dagtag_sector` past
`depend_dagtag`, `set_connection_dagtag()` calls `release_dagtag_wait()` which
unparks the waiting resync read.

**Why this matters:** without `depend_dagtag`, NodeC might read stale data (from
before the primary's write) and send it to NodeB as "current", corrupting the
resync.

### Use 3: `P_PEER_DAGTAG` — sync direction after a primary disappears

When a primary (NodeA) disconnects, surviving nodes need to decide who is ahead.
A node that was connected to NodeA sends `P_PEER_DAGTAG` to other peers
(`drbd_main.c:1802`):

```c
int drbd_send_peer_dagtag(struct drbd_connection *connection,
                           struct drbd_connection *lost_peer)
{
    p->dagtag  = cpu_to_be64(atomic64_read(&lost_peer->last_dagtag_sector));
    p->node_id = cpu_to_be32(lost_peer->peer_node_id);
    return send_command(connection, -1, P_PEER_DAGTAG, DATA_STREAM);
}
```

The receiver (`receive_peer_dagtag`, `drbd_receiver.c:8863`) compares:

```c
dagtag_offset = atomic64_read(&lost_peer->last_dagtag_sector)
              - (s64)be64_to_cpu(p->dagtag);

if      (dagtag_offset > 0) new_repl_state = L_WF_BITMAP_S;  // I am ahead → source
else if (dagtag_offset < 0) new_repl_state = L_WF_BITMAP_T;  // I am behind → target
else                         new_repl_state = L_ESTABLISHED;  // equal → no resync needed
```

### Use 4: Barrier timing safety

The sender checks `seen_dagtag_sector` before sending a barrier
(`drbd_sender.c:3391`):

```c
if (dagtag_newer_eq(connection->send.seen_dagtag_sector,
                    READ_ONCE(resource->dagtag_sector))) {
    // All in-flight writes have been seen → safe to send barrier
    maybe_send_barrier(connection, connection->send.current_epoch_nr + 1);
}
```

This prevents sending a barrier before a write that was assigned a dagtag but
hasn't been picked up by the sender thread yet.

```bash
grep -n "dagtag_sector\b\|depend_dagtag\b\|dagtag_wait_ee\b\|set_connection_dagtag\b" \
    drbd/drbd_req.c drbd/drbd_sender.c drbd/drbd_receiver.c | head -30
```

---

## 8. Dual-Primary with Protocol C — Full Write Path

This is the most important topology to understand deeply: two nodes both promoted
to Primary, `two_primaries = yes`, `protocol C`.

### 8.1 Configuration constraint

```bash
grep -n "two_primaries.*PROT_C\|ERR_NOT_PROTO_C" drbd/drbd_nl.c | head -5
```

`drbd_nl.c:3949`:
```c
if (new_net_conf->two_primaries &&
    (new_net_conf->wire_protocol != DRBD_PROT_C))
    return ERR_NOT_PROTO_C;
```

**`two_primaries = yes` is enforced to require Protocol C.**  This is not a
convention — the kernel rejects the configuration otherwise.  Protocol C means
`bio_endio()` fires only after BOTH local disk completion AND `P_WRITE_ACK` from
the peer.

### 8.2 Symmetric roles

Each node simultaneously plays **two roles** on the same connection:

```
NodeA                              NodeB
  Primary (write initiator)  ←→   Primary (write initiator)
  Secondary (write receiver) ←→   Secondary (write receiver)

NodeA's own writes → P_DATA →→→ NodeB receives, writes locally, sends P_WRITE_ACK
NodeB's own writes → P_DATA →→→ NodeA receives, writes locally, sends P_WRITE_ACK
```

The sender thread on each node drains its own transfer log and sends P_DATA for
its own writes.  The receiver thread on each node processes incoming P_DATA from
the peer and triggers local I/O.

### 8.3 Write path on NodeA (write initiator side)

This is the standard `drbd_make_request()` path (day04) with one key difference:

```c
// drbd_req.c (fan-out loop)
for_each_peer_device(peer_device, device) {
    int idx = peer_device->node_id;
    req->net_rq_state[idx] |= RQ_NET_PENDING | RQ_EXP_WRITE_ACK | RQ_EXP_BARR_ACK;
    atomic_inc(&req->completion_ref);   // +1 for this peer
}
// +1 for local disk → completion_ref starts at 2 in a 2-node cluster
```

NodeA's `completion_ref` starts at 2 (local + 1 peer).  `bio_endio()` fires when
both decrement it to zero:
- Local disk endio → `__req_mod(COMPLETED_OK)` → `completion_ref--`
- `P_WRITE_ACK` from NodeB → `got_BlockAck()` → `__req_mod(WRITE_ACKED_BY_PEER)` → `completion_ref--`

### 8.4 Write path on NodeB (write receiver side)

NodeB's receiver thread calls `receive_Data()`.  The `tp` (two_primaries) flag
drives three additional steps (`drbd_receiver.c:3321`):

```c
tp = nc->two_primaries;

/* Step 1: Protocol C is asserted */
D_ASSERT(device, d.dp_flags & DP_SEND_WRITE_ACK);  // enforced — drbd_receiver.c:3349
peer_req->flags |= EE_SEND_WRITE_ACK;

/* Step 2: peer_seq ordering — serialise NodeA's writes arriving on NodeB */
if (tp) {
    err = wait_for_and_update_peer_seq(peer_device, d.peer_seq);
    // Blocks until peer_seq - 1 has been processed.
    // Guards against packet reordering within NodeA's stream.
}

/* Step 3: hard conflict detection against NodeB's own local writes */
if (tp) {
    err = drbd_peer_write_conflicts(peer_req);  // drbd_receiver.c:3384
    if (err)
        goto out_del_list;  // → disconnect
}
```

After passing all three steps, NodeB inserts the `peer_req` into the interval
tree, submits a local bio, and on bio completion runs `e_end_block()`:

```c
// drbd_receiver.c: e_end_block()
if (peer_req->flags & EE_SEND_WRITE_ACK) {
    pcmd = P_WRITE_ACK;
    drbd_send_ack(peer_device, pcmd, peer_req);
}
```

`P_WRITE_ACK` carries `block_id = (u64)(uintptr_t)req` (the pointer NodeA put in
the P_DATA header), allowing NodeA to look up the request in O(1).

### 8.5 What `drbd_peer_write_conflicts()` actually detects

```bash
grep -n -A 20 "^static int drbd_peer_write_conflicts\b" drbd/drbd_receiver.c
```

```c
// drbd_receiver.c:3012
static int drbd_peer_write_conflicts(struct drbd_peer_request *peer_req)
{
    // Look for a LOCAL_WRITE (NodeB's own application write) overlapping
    // the sector range of the incoming peer write from NodeA.
    i = drbd_find_conflict(device, &peer_req->i, CONFLICT_FLAG_APPLICATION_ONLY);
    if (i) {
        drbd_alert(device,
            "Concurrent writes detected: local=%llus +%u, remote=%llus +%u\n",
            i->sector, i->size, sector, size);
        return -EBUSY;   // → disconnect
    }
    return 0;
}
```

The check fires only when NodeA and NodeB write to the **same sector at the same
time**.  It is NOT the normal path — it is the error path that forces reconnection
and split-brain recovery.

A false positive cannot occur in normal operation: if NodeA and NodeB each write
different sectors, `drbd_find_conflict()` returns NULL and both proceed without
interference.

### 8.6 Epoch/barrier independence in dual-primary

Each node independently manages its own epochs on the connection.  The sender of
each node sends `P_BARRIER` to the peer when its own write stream crosses an epoch
boundary (via `maybe_send_barrier()`, triggered by `start_new_tl_epoch()`).

```
NodeA sender thread:          NodeB sender thread:
  ... sends P_DATA_1 ...          ... sends P_DATA_A ...
  ... sends P_DATA_2 ...          ... sends P_DATA_B ...
  sends P_BARRIER(nr=5)           sends P_BARRIER(nr=3)
  ... sends P_DATA_3 ...          ...

NodeB receiver receives:      NodeA receiver receives:
  P_DATA_1, P_DATA_2              P_DATA_A, P_DATA_B
  P_BARRIER(5) → sends            P_BARRIER(3) → sends
    P_BARRIER_ACK(5)                P_BARRIER_ACK(3)
```

The barrier numbers are independent per direction.  NodeA's epoch numbering is
unrelated to NodeB's.

### 8.7 dagtag in dual-primary

Each node's `resource->dagtag_sector` advances only for its **own** local writes.
When NodeB receives a write from NodeA via `receive_Data()`, the peer_req is
assigned a dagtag from NodeB's tracking of NodeA's stream
(`drbd_receiver.c:3357`):

```c
peer_req->dagtag_sector =
    atomic64_read(&connection->last_dagtag_sector) + (peer_req->i.size >> 9);
// ...
set_connection_dagtag(connection, peer_req->dagtag_sector);
// → atomic64_set(&connection->last_dagtag_sector, peer_req->dagtag_sector)
```

If NodeA sent a gap (reads, or missed writes) before this P_DATA, the sender
already sent a `P_DAGTAG` to NodeB, which updated `last_dagtag_sector` via
`receive_dagtag()`.  This ensures `peer_req->dagtag_sector` is always a correct
cumulative count of NodeA's write stream as seen by NodeB.

### 8.8 Complete symmetric picture

```
NodeA writes sector 100 (4K)         NodeB writes sector 200 (4K)
│                                     │
▼                                     ▼
drbd_make_request()                   drbd_make_request()
  local_bio submitted to disk           local_bio submitted to disk
  completion_ref = 2                    completion_ref = 2
  sender sends P_DATA(100) →→→         sender sends P_DATA(200) →→→
                        ↓                              ↓
             NodeB receive_Data(100)       NodeA receive_Data(200)
               peer_seq check               peer_seq check
               drbd_peer_write_conflicts()  drbd_peer_write_conflicts()
                 → no conflict (200≠100)      → no conflict (100≠200)
               submit local bio              submit local bio
               local disk completes          local disk completes
               e_end_block()                 e_end_block()
               send P_WRITE_ACK(100)→→→     send P_WRITE_ACK(200)→→→
                              ↓                            ↓
                  NodeA got_BlockAck()          NodeB got_BlockAck()
                  WRITE_ACKED_BY_PEER           WRITE_ACKED_BY_PEER
                  completion_ref-- → 0          completion_ref-- → 0
                  bio_endio() ✓                 bio_endio() ✓
```

Both nodes' local disk completions and P_WRITE_ACKs happen in parallel and
independently.  There is no serialization between NodeA's write and NodeB's write
as long as they target different sectors.

---

## 9. Multi-Node Resync Coordination

In a 3-node cluster (A Primary/UpToDate, B Inconsistent, C UpToDate), when B reconnects:

```bash
grep -n "drbd_uuid_compare\b\|nodes_to_reach\|resync.*coordinator\|sync_source_mask" \
    drbd/drbd_receiver.c drbd/drbd_state.c | head -15
```

The resync coordinator is the node with the most recent data. DRBD determines this via UUID comparison across all connected peers. In a 3-node cluster, B might resync from A OR C — DRBD picks the most suitable source:

```c
// Resync source selection considers:
// 1. UUID match (who has the data B needs)
// 2. Load (prefer less-loaded node)
// 3. Proximity (prefer same rack/zone if configured)
// Ultimately: first UpToDate peer in connection order wins in most versions
```

```bash
grep -n "resync_target\|pick_activity_log\|resync_source_selection" drbd/drbd_state.c drbd/drbd_receiver.c | head -10
```

---

## 9. Handling Disk-Full Scenarios in Multi-Node

```bash
grep -n "diskless.*primary\|D_DISKLESS.*R_PRIMARY\|DISKLESS_PRIMARY" \
    drbd/drbd_state.c drbd/drbd_int.h | head -10
```

DRBD 9 supports **diskless Primary** — a node that has no local disk but serves as Primary, reading/writing exclusively through its peer connections:

```
Node A: role=Primary, disk=Diskless
    → all reads: send P_DATA_REQUEST to a peer, receive P_DATA_REPLY
    → all writes: send P_DATA to peers, wait for N peer ACKs
```

This is used for:
- **Compute nodes** that don't need local storage (SDS/hyperconverged)
- **Testing** without a separate data device

```bash
grep -n "D_DISKLESS.*primary\|diskless.*write\|get_ldev_if_state.*D_DISKLESS" \
    drbd/drbd_req.c drbd/drbd_main.c | head -15
```

---

## 10. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Trace `local_rq_state` and `net_rq_state[]` for a 3-peer write
For a Primary with 3 peers (node IDs 0, 1, 2) all in L_ESTABLISHED with Protocol C:
```bash
grep -n "net_rq_state\|local_rq_state\|completion_ref" drbd/drbd_req.c | head -20
```
Trace: which `local_rq_state` bits and which `net_rq_state[i]` bits are set on `__drbd_make_request()`, what does `req->completion_ref` start at, and what events must decrement it before `bio_endio()` fires?

### Exercise 2 (40 min): Understand `P_CONNECTION_FEATURES` node ID exchange
```bash
grep -n -A 40 "^static int drbd_do_handshake\b" drbd/drbd_receiver.c
grep -n "sender_node_id\|receiver_node_id" drbd/drbd_receiver.c drbd/drbd-headers/drbd_protocol.h | head -15
```
What happens if node A thinks peer is node ID 1, but peer says it's node ID 2?

### Exercise 3 (40 min): Trace `P_PEER_DAGTAG` exchange
```bash
grep -n "P_PEER_DAGTAG\|drbd_send_peer_dagtag\|got_PeerDagtag\b" \
    drbd/drbd_sender.c drbd/drbd_receiver.c | head -15
```
When is `P_PEER_DAGTAG` sent? What does the receiver do with it? How does it affect write ordering on a Secondary that receives writes from two different sources?

### Exercise 4 (35 min): Read multi-node state change propagation
When A promotes to Primary in a 3-node cluster:
```bash
grep -n "for_each_connection.*drbd_send_state\|drbd_send_state\b" \
    drbd/drbd_state.c drbd/drbd_sender.c | head -10
```
How many `P_STATE` packets are sent? Do B and C both receive the state change? Does B forward A's state to C?

### Exercise 5 (35 min): Explore bitmap_index assignment for multi-node
```bash
grep -n "bitmap_index\s*=\|alloc_peer_device\|drbd_new_peer_device\b" drbd/drbd_main.c drbd/drbd_nl.c | head -15
```
In a 3-node cluster with node IDs 0, 1, 2: which bitmap slot does each peer get on node 0? What happens if node 3 joins later — does the bitmap need to be resized?

---

## Summary

DRBD 9 uses a full-mesh connection topology: every node connects directly to every other node. Each connection is identified by `peer_node_id` and carries its own sender/receiver/worker threads. Application writes are fanned out to all connected peers simultaneously using the `rq_state[]` array (one slot per peer). Per-peer bitmap slots track OOS independently for each connection. The `dagtag` provides a global monotonic write sequence for ordering in multi-path scenarios. Two-phase commit (`P_TWOPC_*`) and `nodes_to_reach` bitmasks coordinate cluster-wide state changes.

**Next:** Day 23 — Build system, Coccinelle semantic patches, and how to read/write DRBD kernel patches.
