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

In a 2-node cluster, write ordering is trivial: the single connection's sequence numbers enforce order. In multi-node clusters, the **dagtag** (data generation tag) provides a global order:

```bash
grep -n "dagtag\b\|dagtag_sector\b" drbd/drbd_int.h drbd/drbd_req.c drbd/drbd_receiver.c | head -20
```

```c
// drbd_int.h
struct drbd_resource {
    atomic64_t dagtag_sector;   // global monotonic write sequence number
};

struct drbd_request {
    u64 dagtag;    // this request's position in the global order
};
```

On the Primary:
```c
req->dagtag = atomic64_inc_return(&resource->dagtag_sector);
```

Dagtag is sent in `P_DATA` and `P_BARRIER` packets. Secondaries use it to enforce write order when writes from the same Primary arrive via different paths (direct and forwarded).

```bash
grep -n "P_PEER_DAGTAG\b\|drbd_send_peer_dagtag\b" drbd/drbd-headers/drbd_protocol.h drbd/drbd_sender.c | head -10
```

`P_PEER_DAGTAG` is sent by a Secondary to peers after processing a write, so other nodes in the cluster can track the global write order.

---

## 8. Multi-Node Resync Coordination

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
