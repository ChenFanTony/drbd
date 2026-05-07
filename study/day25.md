# Day 25 — Performance Tuning: AL Sizing, TCP Buffers, Write Ordering & Benchmarking

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_actlog.c`, `drbd/drbd_transport_tcp.c`, `drbd/drbd_req.c`, `drbd/drbd_int.h`

---

## 1. Performance Mental Model

A Protocol-C write's latency has four components:

```
total_latency = AL_miss_penalty  (if new AL extent)
              + local_disk_time  (local SSD/HDD write)
              + RTT/2            (network one-way)
              + peer_disk_time   (secondary SSD/HDD write)
```

For sequential workloads on NVMe: AL misses negligible, local ~100µs, RTT ~200µs, peer ~100µs → ~400µs total.

For random workloads on HDD with frequent AL misses: AL miss = ~10ms → dominates everything.

**Tuning goal:** Minimise each component independently.

---

## 2. Activity Log Tuning

### The AL Miss Penalty

```bash
grep -n "drbd_al_write_transaction\b\|REQ_PREFLUSH\|REQ_FUA" drbd/drbd_actlog.c | head -10
```

Each AL miss triggers a synchronous `REQ_PREFLUSH | REQ_FUA` write to the metadata device:
- **HDD:** ~5–15 ms (seek + rotational latency + flush)
- **SATA SSD:** ~500µs–2ms (flush latency varies widely)
- **NVMe:** ~50–200µs
- **NVDIMM/pmem:** ~1µs

### `al-extents` — Tuning the Cache Size

```bash
grep -n "al.extents\|DRBD_AL_EXTENTS_DEF\|DRBD_AL_EXTENTS_MIN\|DRBD_AL_EXTENTS_MAX" \
    drbd/drbd_int.h drbd/linux/drbd.h | head -10
```

```
Default: al-extents 1024  → covers 1024 × 4MiB = 4 GiB of active write area
Minimum: al-extents 7     → covers 28 MiB (almost always misses)
Maximum: al-extents 65536 → covers 256 GiB
```

**Rule of thumb:**
```
al-extents should cover your working set size / 4MiB

Working set = 10 GiB → al-extents = 10*1024/4 = 2560
Working set = 100 GiB sequential → al-extents = 128 (only ~4 extents active at once)
Working set = 100 GiB random → al-extents = 25600 (all extents hot)
```

Diagnosis: check the AL hit rate:
```bash
cat /sys/kernel/debug/drbd/r0/volumes/0/act_log
# hits:234123 misses:456  → miss rate = 0.19% (good)
# hits:1000 misses:999    → miss rate = 50% (terrible — increase al-extents)
```

### Metadata Device Placement

Moving metadata to a faster device eliminates the AL penalty:

```
resource r0 {
  disk /dev/sda;
  meta-disk /dev/nvme0n1p1;    ← NVMe for metadata
}
```

```bash
grep -n "drbd_md_sync_page_io\b\|md_bdev\b" drbd/drbd_main.c drbd/drbd_actlog.c | head -10
# drbd_md_sync_page_io() always goes to md_bdev
# If md_bdev == backing_bdev: metadata shares the data device (internal)
# If separate: metadata on its own device (external)
```

---

## 3. Write Ordering Modes

```bash
grep -n "write_ordering\b\|WO_NONE\|WO_DRAIN\|WO_FLUSH\|WO_BDEV_FLUSH\|wo:\|drbd_write_ordering" \
    drbd/drbd_req.c drbd/drbd_int.h | head -20
```

```c
enum write_ordering_e {
    WO_NONE      = 0,   // no write ordering (unsafe)
    WO_DRAIN_IO  = 1,   // wait for all pending I/O to drain at epoch boundary
    WO_BDEV_FLUSH= 2,   // send REQ_FLUSH to local block device at barrier
    WO_BIO_BARRIER=3,   // use REQ_BARRIER (legacy, mostly gone)
};
```

The write ordering mode affects how DRBD handles epoch boundaries (barriers from the application):

| Mode | `wo:` in /proc/drbd | Mechanism | Performance |
|---|---|---|---|
| `none` | `n` | No ordering | Fastest, unsafe |
| `drain` | `d` | Drain all writes before next epoch | Safe, medium |
| `flush` | `f` | REQ_FLUSH to disk at barrier | Safe, slower |
| `barrier` | `b` | REQ_BARRIER (legacy) | Kernel-dependent |

```bash
grep -n "drbd_bump_write_ordering\b\|drbd_set_write_ordering\b" drbd/drbd_req.c drbd/drbd_main.c | head -10
grep -n -A 40 "^void drbd_bump_write_ordering\b" drbd/drbd_main.c
```

DRBD automatically downgrades the write ordering mode if the backing device doesn't support barriers:
```
WO_BIO_BARRIER → WO_BDEV_FLUSH → WO_DRAIN_IO → WO_NONE
```

### `no-disk-barrier` and `no-disk-flush`

```bash
grep -n "no.disk.barrier\|no.disk.flush\|no_disk_barrier\|no_disk_flush" \
    drbd/drbd_int.h drbd/drbd_req.c drbd/drbd_nl.c | head -15
```

```
disk {
  no-disk-barrier yes;   # skip REQ_BARRIER entirely
  no-disk-flush yes;     # skip REQ_FLUSH (DANGEROUS on hardware without supercapacitor)
  no-disk-drain yes;     # skip epoch drain
  no-md-flush yes;       # skip flush on metadata writes (very dangerous)
}
```

`no-disk-flush yes` is safe ONLY if:
- The block device has a battery/supercapacitor-backed write cache (BBU RAID card, NVMe with power-loss protection)
- Confirmed with hardware vendor

---

## 4. TCP Socket Buffer Tuning

```bash
grep -n "sndbuf.size\|rcvbuf.size\|sndbuf_size\|rcvbuf_size\|drbd_setbufsize\b" \
    drbd/drbd_transport_tcp.c drbd/drbd_int.h | head -20
grep -n -A 20 "^static void drbd_setbufsize\b" drbd/drbd_transport_tcp.c
```

```c
static void drbd_setbufsize(struct socket *sock, unsigned int snd, unsigned int rcv)
{
    if (snd) {
        sock->sk->sk_sndbuf = snd;
        sock->sk->sk_userlocks |= SOCK_SNDBUF_LOCK;
    }
    if (rcv) {
        sock->sk->sk_rcvbuf = rcv;
        sock->sk->sk_userlocks |= SOCK_RCVBUF_LOCK;
    }
}
```

From `drbd.conf`:
```
net {
  sndbuf-size 1M;     # TCP send buffer (default: OS auto-tuned)
  rcvbuf-size 1M;     # TCP receive buffer
  # For 10GbE with 200µs RTT (bandwidth-delay product):
  # BDP = 10Gbps × 200µs = 250 KB → set sndbuf to at least 250K
}
```

**Bandwidth-delay product formula:**
```
optimal_buffer = link_bandwidth_bytes × RTT_seconds
10GbE, 200µs RTT: 10e9/8 × 0.0002 = 250,000 bytes = 244 KiB
1GbE,  1ms RTT:   1e9/8  × 0.001  =  125,000 bytes = 122 KiB
```

If `sndbuf-size` is too small, the sender thread blocks in `kernel_sendpage()` waiting for TCP to drain — increasing protocol latency.

---

## 5. `max-buffers` and `max-epoch-size`

```bash
grep -n "max.buffers\|max.epoch.size\|max_buffers\|max_epoch_size\b" \
    drbd/drbd_int.h drbd/drbd_receiver.c drbd/drbd_nl.c | head -15
```

```
net {
  max-buffers 8000;       # max drbd_peer_request objects allocated at once
                          # each = 1 page (4KiB) + struct overhead
                          # 8000 × 4KiB = 32 MiB receiver buffer

  max-epoch-size 2048;    # max writes per epoch before forced barrier
                          # smaller = more frequent flushing = safer but slower
                          # larger = higher throughput, larger crash recovery window
}
```

```bash
grep -n "ee_wait\b\|drbd_alloc_peer_req\b.*sleep\|max_buffers" drbd/drbd_receiver.c | head -10
```

If `max-buffers` is exhausted, the receiver thread blocks — this applies flow control back to the sender thread on the primary.

---

## 6. `ping-timeout` and `ko-count` — Disconnect Sensitivity

```bash
grep -n "ping.timeout\|ko.count\|ping_timeout\|ko_count" drbd/drbd_int.h drbd/drbd_receiver.c | head -15
```

```
net {
  ping-timeout 5;     # seconds without data before sending P_PING
  ping-int 10;        # interval between pings
  ko-count 7;         # disconnect after 7 consecutive timeouts
  connect-int 10;     # retry interval after disconnect
}
```

In high-latency or lossy networks, increase `ping-timeout` to avoid spurious disconnects.
In low-latency datacenter networks, decrease to detect failures faster.

---

## 7. `resync-rate` and `c-*` Parameters

```bash
grep -n "resync.rate\|c.plan.ahead\|c.fill.target\|c.delay.target\|c.max.rate" \
    drbd/drbd_int.h drbd/linux/drbd.h drbd/drbd_worker.c | head -20
```

```
disk {
  resync-rate 250M;      # target resync throughput (KiB/s, so 250M = 250000)
  c-plan-ahead 20;       # planning horizon in 100ms units = 2 seconds
  c-fill-target 0;       # desired in-flight sectors (0 = use resync-rate only)
  c-delay-target 1;      # target RTT in 100ms units = 100ms
  c-max-rate 400M;       # hard cap on resync rate
  c-min-rate 4M;         # minimum rate (don't let resync stall completely)
}
```

**Resync rate vs. application I/O:**
- Too high `resync-rate`: resync saturates the network/disk, application I/O latency spikes
- Too low: resync takes forever, crash recovery window remains large

**Rule:** Set `resync-rate` to ~30–50% of available bandwidth during business hours via cron:
```bash
drbdsetup peer-device-options r0 peer_node_id=1 --resync-rate=100M  # business hours
drbdsetup peer-device-options r0 peer_node_id=1 --resync-rate=900M  # overnight
```

---

## 8. Benchmarking Methodology

### Step 1: Baseline (no DRBD)
```bash
# Raw device performance:
fio --name=seq-write --rw=write --bs=1M --size=10G --ioengine=libaio \
    --iodepth=32 --direct=1 --filename=/dev/sda
fio --name=rand-write --rw=randwrite --bs=4k --size=10G --ioengine=libaio \
    --iodepth=32 --direct=1 --filename=/dev/sda
```

### Step 2: DRBD overhead (single node, loopback)
```bash
# Replace /dev/sda with /dev/drbd0 in same fio commands
# Difference = DRBD overhead (AL transactions, interval tree, etc.)
```

### Step 3: Protocol A (async) vs Protocol C (sync)
```bash
# Change net { protocol A; } and re-run
# Protocol A latency ≈ Protocol B latency for sequential writes
# Protocol C latency ≈ Protocol A + RTT + peer_write_time
```

### Step 4: AL tuning impact
```bash
# Benchmark random 4K writes with different al-extents:
for extents in 128 512 1024 4096; do
    drbdsetup disk-options r0 --al-extents=$extents
    fio --name=rand4k --rw=randwrite --bs=4k --size=10G ...
    cat /sys/kernel/debug/drbd/r0/volumes/0/act_log | grep "misses"
done
```

### Step 5: Monitor during benchmark
```bash
# In a parallel terminal:
watch -n1 "cat /proc/drbd | grep -A2 'drbd0'"
# Watch: ns (net sent), dw (disk writes), al (AL writes), oos
```

---

## 9. Profiling DRBD with `perf`

```bash
# Record DRBD-related kernel events:
perf record -g -a -e block:block_rq_complete,block:block_bio_queue \
    -- sleep 30

# Report:
perf report --stdio | head -50

# Specific function profiling:
perf top -p $(pgrep -f "drbd_s") --call-graph dwarf
```

Key functions to look for in perf output:
- `drbd_al_write_transaction` — high here = AL misses dominate
- `drbd_bm_find_next` — high here = resync CPU bound
- `drbd_recv_short` — high here = receiver CPU bound (check TCP interrupt affinity)
- `drbd_send_dblock` — high here = sender CPU bound

---

## 10. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Measure AL miss rate vs `al-extents`
On a test system, run random 4K writes and measure:
```bash
for extents in 64 256 1024 4096; do
    echo "=== al-extents=$extents ==="
    drbdadm down r0
    # Edit drbd.conf: disk { al-extents $extents; }
    drbdadm up r0
    fio --name=t --rw=randwrite --bs=4k --size=1G --runtime=30 \
        --filename=/dev/drbd0 --direct=1 --ioengine=libaio --iodepth=8 \
        --output-format=terse 2>/dev/null | cut -d';' -f7,8
    cat /sys/kernel/debug/drbd/r0/volumes/0/act_log
done
```

### Exercise 2 (40 min): Compute optimal TCP buffer for your link
```bash
# Measure RTT between DRBD nodes:
ping -c 100 peer_ip | tail -2

# Measure link bandwidth:
iperf3 -c peer_ip -t 30

# Compute BDP:
# BDP_bytes = bandwidth_bytes_per_sec × RTT_sec
# Set sndbuf-size and rcvbuf-size to at least BDP
```

### Exercise 3 (40 min): Trace `drbd_bump_write_ordering()`
```bash
grep -n -A 60 "^void drbd_bump_write_ordering\b" drbd/drbd_main.c
```
Under what conditions does DRBD downgrade from `WO_BDEV_FLUSH` to `WO_DRAIN_IO`? What kernel error triggers the downgrade? Is it permanent or temporary?

### Exercise 4 (30 min): Read `drbd_congested()` with production values
Given:
- `congestion-fill 4M` (4194304 bytes = 8192 sectors)
- Protocol C, peer disk = HDD at 100 IOPS × 4KiB = 400 KiB/s throughput
- Primary sending at 1 GiB/s

How quickly does `ap_in_flight` grow until congestion triggers? What is the latency impact right before the threshold?

### Exercise 5 (40 min): Find `drbd_md_setbufsize` path
```bash
grep -n "sndbuf_size\|rcvbuf_size\b" drbd/drbd_transport_tcp.c drbd/drbd_int.h | head -10
```
Trace how `sndbuf-size` from `drbd.conf` reaches `sock->sk->sk_sndbuf`:
1. Parsed by `net_conf_from_attrs()` in `drbd_nl.c`
2. Stored in `net_conf->sndbuf_size`
3. Applied by `dtt_connect()` → `drbd_setbufsize()`

---

## Summary

DRBD performance is dominated by four factors: AL miss penalty (fix: fast metadata device or larger `al-extents`), local disk speed, network RTT, and peer disk speed. Write ordering modes control durability vs. throughput tradeoffs. TCP buffer sizing should match the bandwidth-delay product. Resync rate should be tuned to avoid starving application I/O. The DebugFS `act_log` file and `/proc/drbd` counters provide real-time diagnostics. The optimal tuning approach is: baseline without DRBD, then add DRBD overhead, then measure each knob independently.

**Next:** Day 26 — TLS encryption in DRBD 9.2: `drbd_transport_tcp.c` with kernel TLS (`kTLS`), certificate configuration, and the encryption handshake.
