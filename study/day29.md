# Day 29 — RDMA Transport: Architecture, Zero-Copy Paths & TCP vs RDMA Comparison

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd-headers/drbd_transport.h`, `drbd/drbd_transport.c`, `drbd/drbd_transport_rdma.c` (~100 KB, in this repo), `drbd/drbd_int.h`

> **⚠️ Correction from earlier draft:** This day previously claimed `drbd_transport_rdma.c` was "external / LINBIT proprietary, separate repo." That was wrong — **it ships in this repo at `drbd/drbd_transport_rdma.c`** (~100 KB, GPL-licensed, builds as `drbd_transport_rdma.ko` alongside `drbd_transport_tcp.ko`). The same is true for `drbd/drbd_transport_lb-tcp.c` (load-balancing TCP transport).
> ```bash
> ls -lh drbd/drbd_transport_*.c
> ```

---

## 1. Why RDMA for DRBD?

RDMA (Remote Direct Memory Access) bypasses the CPU for data transfer:

```
TCP path:                          RDMA path:
  data in bio pages                  data in bio pages
       ↓                                  ↓
  copy to kernel socket buffer       RDMA engine reads directly
       ↓                             (no copy, no CPU interrupt)
  TCP/IP stack (CPU)                      ↓
       ↓                             InfiniBand / RoCE / iWARP NIC
  NIC DMA                                 ↓
       ↓                             peer RDMA engine writes to
  network                           peer memory directly
       ↓
  peer NIC DMA
       ↓
  peer socket buffer (copy)
       ↓
  peer drbd_recv()
```

**RDMA advantages for DRBD:**
- **Latency:** ~1–3µs vs ~50–200µs for TCP (same datacenter)
- **CPU:** Near-zero CPU for data movement (only control path uses CPU)
- **Bandwidth:** 100–200 Gbps InfiniBand vs 10–25 Gbps typical Ethernet

**When RDMA matters:** Extremely latency-sensitive databases, HPC storage, NVMe-over-Fabrics style workloads.

---

## 2. The Transport Abstraction — How RDMA Fits

The `drbd_transport_ops` vtable (Day 16) abstracts everything. The RDMA transport (`drbd_transport_rdma.ko`, **built from `drbd/drbd_transport_rdma.c` in this repo**) implements the same vtable:

```c
// drbd/drbd_transport_rdma.c — actual implementation in this tree:
static struct drbd_transport_ops dtr_ops = {
    .init         = dtr_init,
    .free         = dtr_free,
    .connect      = dtr_connect,
    .recv         = dtr_recv,
    .recv_pages   = dtr_recv_pages,
    .send_page    = dtr_send_page,
    .send_zc_bio  = dtr_send_zc_bio,    // zero-copy bio send
    .stream_ok    = dtr_stream_ok,
    .hint         = dtr_hint,
    .close        = dtr_close,
    // ...
};
```

```bash
grep -n "struct drbd_transport_ops\b\|drbd_transport_ops\b" drbd/drbd-headers/drbd_transport.h | head -5
# Verify the vtable structure is identical for TCP, RDMA, and LB-TCP
grep -n "static.*drbd_transport_ops\|\.connect\s*=" drbd/drbd_transport_rdma.c drbd/drbd_transport_tcp.c drbd/drbd_transport_lb-tcp.c | head -10
```

From `drbd_receiver.c` and `drbd_sender.c`, the calls are:
```c
connection->transport->ops->recv(...)
connection->transport->ops->send_page(...)
// Identical whether TCP or RDMA transport is loaded
```

---

## 3. RDMA Connection Setup — `dtr_connect()`

Unlike TCP's `kernel_connect()`, RDMA requires establishing **Queue Pairs (QPs)**:

```
RDMA connect sequence:
1. Create IB protection domain (PD)
2. Create completion queue (CQ) — gets notified when ops complete
3. Create Queue Pair (QP): send_queue + recv_queue
   QP = bidirectional channel, like a socket but memory-mapped

4. Exchange QP identifiers with peer via a "CM" (connection manager)
   → rdma_connect() / rdma_accept() using RDMA CM (rdma_cm.h)

5. Post receive buffers (pre-allocate where incoming data will land)
   → rdma_post_recv() — posts memory regions into QP's receive queue

6. Connection established: data can flow
```

The DRBD transport registers a path (IP address pair), and the RDMA CM resolves to InfiniBand GID or RoCE MAC automatically.

---

## 4. RDMA Data Send — `dtr_send_page()` vs `dtt_send_page()`

**TCP:** `kernel_sendpage(sock, page, offset, size, flags)`
- Copies page → socket send buffer (or splice if zero-copy capable)
- CPU involved in every byte movement

**RDMA:** `ib_post_send(qp, send_wr, bad_wr)`
- Registers the page as an RDMA memory region (MR)
- Posts a SEND Work Request (WR) to the QP
- NIC reads the page directly via DMA
- CPU only involved in posting the WR (~100ns overhead)

```c
// Conceptual dtr_send_page() for RDMA:
static int dtr_send_page(struct drbd_transport *transport,
                          enum drbd_stream stream,
                          struct page *page,
                          int offset, size_t size, unsigned flags)
{
    // Register page for DMA:
    mr = ib_reg_mr(pd, page_address(page) + offset, size,
                   IB_ACCESS_LOCAL_WRITE);

    // Post send work request:
    send_wr.opcode     = IB_WR_SEND;
    send_wr.sg_list    = &sge;
    sge.addr           = mr->iova;
    sge.length         = size;
    sge.lkey           = mr->lkey;

    ib_post_send(qp, &send_wr, &bad_wr);

    // Wait for completion (or use async):
    poll_cq(cq, &wc);

    ib_dereg_mr(mr);
    return 0;
}
```

---

## 5. RDMA Zero-Copy Receive — `dtr_recv_pages()`

The key RDMA optimisation for DRBD is **RDMA WRITE from peer directly into local pages**:

```
Without RDMA WRITE (SEND/RECV model):
  Peer: post_send(data)
  Local: post_recv(buffer)  → data arrives in pre-posted buffer
  → one memory copy: buffer → peer_req->pages

With RDMA WRITE:
  Local: expose peer_req->pages as RDMA target (register MR, share rkey)
  Peer: rdma_write(peer_MR, local_data_pages)
  → peer NIC writes DIRECTLY into peer_req->pages
  → ZERO copies on local side
  → local CPU gets completion notification only
```

To enable RDMA WRITE, DRBD adds an MR registration step to the receiver's setup:

```c
// Before receive_Data(), receiver sets up RDMA target:
// 1. Register peer_req->pages as DMA target
mr = ib_reg_mr(pd, peer_req_pages_va, size, IB_ACCESS_REMOTE_WRITE);

// 2. Send the MR's rkey + address to peer (via control message)
drbd_send_drequest_csum(..., mr->rkey, mr->iova, ...);

// 3. Peer does RDMA WRITE using the rkey:
write_wr.opcode      = IB_WR_RDMA_WRITE;
write_wr.wr.rdma.remote_addr = peer_iova;
write_wr.wr.rdma.rkey        = peer_rkey;
ib_post_send(qp, &write_wr, &bad_wr);

// 4. Data arrives directly in local peer_req->pages
// 5. Peer sends completion signal (SEND) → local knows data is ready
```

This eliminates ALL data copies for the replication path — data moves from primary's bio pages directly into secondary's peer_req pages via the NIC's DMA engine.

---

## 6. DRBD Protocol Additions for RDMA

RDMA requires protocol extensions not needed for TCP:

```bash
grep -n "P_RS_THIN_REQ\|P_PEERS_IN_SYNC\|P_DRBD10\|drbd_proto.*rdma\|protocol.*rdma" \
    drbd/drbd-headers/drbd_protocol.h | head -10
```

Additional packets for RDMA operation:
```
P_RDMA_REQ     → "here is my MR rkey + address, write data here"
P_RDMA_WRITE   → "RDMA write complete, data is in your buffer"
```

These are sent over the CONTROL stream (META socket equivalent in RDMA = a separate QP).

---

## 7. TCP vs RDMA Performance Comparison

| Metric | TCP (1GbE) | TCP (25GbE) | RDMA (IB HDR 200Gbps) |
|---|---|---|---|
| Latency | 200–1000µs | 50–200µs | 1–5µs |
| Throughput | 1 Gbps | 25 Gbps | 200 Gbps |
| CPU (data path) | High | High | Near zero |
| Setup complexity | Simple | Simple | Complex (HCA, fabric) |
| Cost | Low (NIC: $50) | Medium ($200) | High ($500–2000/port) |

**For DRBD replication impact:**

Protocol-C write latency with 4 KiB blocks:
```
TCP 1GbE:    local(100µs) + RTT/2(500µs) + peer(100µs) = ~700µs
TCP 25GbE:   local(50µs)  + RTT/2(100µs) + peer(50µs)  = ~200µs
RDMA IB:     local(50µs)  + RTT/2(2µs)   + peer(50µs)  = ~102µs
NVMe+RDMA:   local(10µs)  + RTT/2(2µs)   + peer(10µs)  = ~22µs
```

RDMA's main benefit: RTT drops from hundreds of microseconds to 1–3µs.

---

## 8. MR (Memory Region) Registration Cost

One practical consideration: RDMA MR registration is expensive (~10–100µs per registration). DRBD avoids this overhead by:

1. **Pre-registering page pools:** All pages from `drbd_buffer_page_pool` are pre-registered at module load time
2. **Caching MRs:** Registered MRs are cached and reused
3. **Using `ib_dma_map_page()`** instead of full MR registration for small transfers

```bash
# In drbd_transport_rdma.c (external):
# grep -n "ib_reg_mr\|ib_alloc_mr\|dma_map_page\|mr_cache" drbd_transport_rdma.c
```

---

## 9. The `send_zc_bio` vtable Entry

```bash
grep -n "send_zc_bio\b" drbd/drbd-headers/drbd_transport.h drbd/drbd_sender.c | head -10
```

`send_zc_bio` (zero-copy bio send) is an RDMA-specific optimisation:

```c
// drbd_transport.h:
int (*send_zc_bio)(struct drbd_transport *, struct bio *);
// "zero copy": send bio pages directly without copying to socket buffer

// TCP implementation: falls back to send_page() per page
// RDMA implementation: registers bio->bi_io_vec pages as RDMA MR
//                      posts RDMA WRITE work request
//                      NIC reads bio pages and sends them
```

The TCP transport's `dtt_send_zc_bio()`:
```bash
grep -n "send_zc_bio\b\|dtt_send_zc_bio\b" drbd/drbd_transport_tcp.c | head -5
```

---

## 10. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Study the full transport vtable
```bash
grep -n "struct drbd_transport_ops {" drbd/drbd-headers/drbd_transport.h
# Read every function pointer
```
For each function pointer:
1. What does it abstract (TCP equivalent)?
2. How would RDMA implement it differently?
3. Does TCP use `send_zc_bio`? What does its implementation do?

### Exercise 2 (40 min): Trace how the transport is selected at runtime
When `drbdadm connect` runs:
```bash
grep -n "drbd_get_transport_class\b\|transport_class\b\|tcp_transport_class\b" \
    drbd/drbd_transport.c drbd/drbd_transport_tcp.c | head -15
grep -n -A 40 "^struct drbd_transport \*drbd_get_transport\b\|drbd_alloc_transport\b" \
    drbd/drbd_transport.c
```
How does the kernel know whether to use `drbd_transport_tcp` or `drbd_transport_rdma`? Is it per-connection or per-resource?

### Exercise 3 (40 min): Understand RDMA prerequisites
```bash
# Check if RDMA is available:
ls /dev/infiniband/ 2>/dev/null
lsmod | grep ib_core
cat /sys/class/infiniband/*/fw_ver 2>/dev/null

# Check kernel config:
grep CONFIG_INFINIBAND /boot/config-$(uname -r) | head -10
```
What kernel modules are required for RDMA? How does the kernel RDMA subsystem (`ib_core`) relate to DRBD's transport abstraction?

### Exercise 4 (35 min): Read `drbd_alloc_mempools()` for RDMA considerations
```bash
grep -n "drbd_buffer_page_pool\|drbd_create_mempools\|page_pool" drbd/drbd_main.c | head -15
```
In the TCP transport, pages from `drbd_buffer_page_pool` are used as receive buffers. For RDMA, these same pages need to be DMA-mapped. How would a pre-registration cache work? What struct would track "this page is registered as MR X on device Y"?

### Exercise 5 (35 min): Compare latency contributors
For a synchronous Protocol-C write to a peer 10km away (RTT ~100µs):
- TCP 10GbE: what is the total latency breakdown?
- RDMA IB: what is the total latency breakdown?
- At what network distance does RDMA's advantage diminish? (Hint: when RTT >> RDMA latency advantage)

---

## Summary

DRBD's transport abstraction (`drbd_transport_ops`) cleanly separates the replication protocol from the network implementation. The RDMA transport implements the same vtable as TCP but uses InfiniBand Queue Pairs instead of sockets, RDMA WRITE instead of `kernel_sendpage()`, and pre-registered Memory Regions instead of socket buffers. The primary RDMA advantage is latency: 1–5µs vs 50–1000µs for TCP over LAN. RDMA's CPU elimination is secondary for DRBD since DRBD itself is not CPU-bound. MR registration cost is amortised by pre-registration and caching of the page pool.

**Next:** Day 30 — Final review: end-to-end architecture synthesis, common pitfalls, contributing to DRBD upstream, and your learning roadmap beyond this course.
