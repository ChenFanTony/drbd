# Review & Corrections — DRBD 9.2 Study Plan

> This document lists corrections discovered after fact-checking the 30-day plan against the actual `https://github.com/ChenFanTony/drbd` (`drbd-9.2` branch) source code on **2026-05-07**.
>
> Read this BEFORE starting Day 1. The original 30 days are still useful as a structured reading guide, but several function names, file names, and architectural details below are wrong.

---

## A. File Inventory — What Actually Exists

The actual `drbd/` directory in this repo contains these `.c` files:

```
drbd_actlog.c          (894 lines — small!)
drbd_bitmap.c          (1817 lines)
drbd_dax_pmem.c        ★ NEW — persistent memory metadata support
drbd_debugfs.c         (~2400 lines)
drbd_interval.c        (194 lines)
drbd_kref_debug.c      ★ NEW — refcount leak debugging
drbd_legacy_84.c       ★ NEW — DRBD 8.4 backward compatibility
drbd_main.c            (6357 lines)
drbd_nl.c              (8084 lines — largest, not 5000)
drbd_nla.c             (very small, ~50 lines)
drbd_proc.c            (only ~30 lines! — just a stub for /proc/drbd)
drbd_receiver.c        (11554 lines — really the largest)
drbd_req.c             (2970 lines)
drbd_sender.c          (3796 lines — ALSO contains drbd_worker())
drbd_state.c           (6336 lines)
drbd_transport.c       (~270 lines)
drbd_transport_lb-tcp.c   ★ NEW — Load Balancing TCP transport
drbd_transport_rdma.c     ★ NEW — RDMA transport (in-tree, NOT external)
drbd_transport_tcp.c   (~1500 lines)
drbd_transport_template.c
kref_debug.c
```

### Files I Wrongly Claimed Exist

The following files I cited **do NOT exist** in `drbd/`:

| Claimed file | Actual location |
|---|---|
| `drbd/drbd_worker.c` | **Does not exist.** All `drbd_worker()` and `w_*` callbacks live in `drbd_sender.c` |
| `drbd/lru_cache.c` | **Does not exist** in `drbd/`. It's in `drbd/drbd-kernel-compat/lru_cache.c` (compat copy of upstream `lib/lru_cache.c`) |
| `drbd/drbd_strings.c` | **Submodule.** Lives in `drbd/drbd-headers/drbd_strings.c` |
| `drbd/drbd_buildtag.c` | Auto-generated at build time, lives in build tree, not source |
| `drbd/drbd_protocol.h` | **Submodule.** Lives in `drbd/drbd-headers/drbd_protocol.h` |
| `drbd/drbd_wrappers.o` (in Makefile) | Removed in 9.2 |

### Files I Missed Entirely

These deserve their own treatment but I never mentioned them:

- **`drbd_dax_pmem.c`** — DAX/pmem support: metadata on persistent memory (NVDIMM). Replaces synchronous `REQ_FUA` writes with byte-addressable cache-line flushes. Eliminates the AL flush bottleneck (Day 8) when used.
- **`drbd_kref_debug.c` + `kref_debug.c`** — Optional refcount leak detection infrastructure. Compile-time enabled via `CONFIG_DRBD_KREF_DEBUG`.
- **`drbd_legacy_84.c`** — Compatibility layer to interoperate with DRBD 8.4 nodes. Allows mixed-version clusters during upgrades.
- **`drbd_transport_lb-tcp.c`** — Load-balancing TCP transport: spreads traffic across multiple TCP connections to one peer. Useful for 100+ Gbps Ethernet where a single TCP flow can't saturate the link.
- **`drbd_transport_rdma.c`** — **Day 29 said this was external. It is not — it ships in this repo.**

### Submodule: `drbd/drbd-headers/`

The `drbd-headers` directory is a git submodule. To populate it:
```bash
git submodule update --init
```

It contains:
- `drbd_protocol.h` — packet types, on-wire structs
- `drbd_strings.c` + `drbd_strings.h` — enum-to-string conversions
- `drbd_meta_data.h` — on-disk metadata layout
- `drbd_transport.h` — transport vtable definitions
- `linux/drbd.h` — UAPI header
- `linux/drbd_genl.h` + `linux/drbd_genl_api.h` — Netlink schema
- `linux/drbd_limits.h` — tunable bounds
- `linux/drbd_config.h` — kernel config

---

## B. Function Name Corrections

### Day 1 / Day 4 — Block Device Entry Point

| What I wrote | What it actually is |
|---|---|
| `drbd_make_request()` is the entry point | **`drbd_submit_bio()`** is the kernel-facing entry point (modern kernels). It calls `__drbd_make_request()` internally |
| `drbd_ops.make_request = drbd_make_request` | `drbd_ops.submit_bio = drbd_submit_bio` |
| `drbd_ops` has `.ioctl, .getgeo, .media_changed` | Actually only has `.owner, .submit_bio, .open, .release` |
| `drbd_pp_pool` | Renamed to **`drbd_buffer_page_pool`** (with separate `drbd_md_io_page_pool` for metadata I/O) |

### Day 4 — Request State

| What I wrote | What it actually is |
|---|---|
| `unsigned long rq_state[1 + DRBD_PEERS_MAX]` (single combined array) | Two separate fields: `unsigned int local_rq_state` and `u16 net_rq_state[DRBD_NODE_ID_MAX]` |
| Bit definitions like `RQ_NET_PENDING, RQ_LOCAL_PENDING` are correct | The constants are correct, but they apply to the right field (local bits in `local_rq_state`, net bits in `net_rq_state[]`) |
| `req` only protected by `resource->req_lock` | Each `drbd_request` also has its own `spinlock_t rq_lock` |

### Day 5 / Day 6 — Sender / Receiver / ACK Sender

| What I wrote | What it actually is |
|---|---|
| `e_end_block()` is called from the sender thread | It's called from a separate **`ack_sender` workqueue** (`connection->ack_sender`). This is architecturally important — ACKs don't queue behind data sends |
| Three threads per connection (receiver, sender, worker) | More accurate: receiver thread + sender thread + ack_sender workqueue + the `drbd_worker()` thread (resource-level) |
| `drbd_worker()` lives in `drbd_worker.c` | Lives in `drbd_sender.c` (line ~3724) |

### Day 7 — Connection Lifecycle

| What I wrote | What it actually is |
|---|---|
| `drbd_disconnect()` | The function is **`conn_disconnect()`** (in `drbd_receiver.c` line ~9829) |
| `drbd_do_handshake()` | Function exists but is much more complex than my description; covers feature negotiation, twopc compatibility check, etc. |

### Day 8 / Day 13 — Activity Log & LRU Cache

| What I wrote | What it actually is |
|---|---|
| `drbd_al_begin_io()` is the main entry | Actually three functions: **`drbd_al_begin_io_fastpath()`** (cache hit path), **`drbd_al_begin_io_nonblock()`** (handles miss without sleeping), and **`drbd_al_begin_io_commit()`** (after disk transaction completes) |
| `drbd_al_complete_io()` returns void | Actually returns `bool` (true if extent was active, false otherwise) |
| `lru_cache.c` is in `drbd/` | It's in `drbd/drbd-kernel-compat/lru_cache.c` |
| `lc_get()` returns NULL on `LC_CHANGING` | `lc_get()` actually has flags and several return modes. `lc_get_cumulative()` and `lc_try_get()` are also part of the API. |

### Day 10 / Day 11 — Resync & Worker Items

| What I wrote | What it actually is |
|---|---|
| `w_make_resync_request()` | Actually called **`make_resync_request()`** (no `w_` prefix; in `drbd_sender.c` line 1164) |
| Many `w_*` functions in a fictional `drbd_worker.c` | Most exist but live in `drbd_sender.c`, with different exact names. Always `grep` the actual file |

### Day 17 — UUIDs

| What I wrote | What it actually is |
|---|---|
| `HISTORY_UUIDS` is "Typically 2 in older DRBD, up to 14 in DRBD 9" | Actually `HISTORY_UUIDS = DRBD_PEERS_MAX = 32` in DRBD 9. The DRBD 8 constant is `HISTORY_UUIDS_V08` |

### Day 23 — Build System

| What I wrote | What it actually is |
|---|---|
| `Makefile` lists `obj-$(CONFIG_BLK_DEV_DRBD)` etc. | The actual build uses `Kbuild`, `Kbuild.drbd`, and `Makefile.spatch` for spatch-based compat generation. More complex than I described |
| `drbd/compat/` directory | Actually `drbd/drbd-kernel-compat/` (different name) |
| Compat detection by Makefile shell tests | Uses Coccinelle (`spatch`) to generate compat code from `.spatch` rules in `drbd/drbd-kernel-compat/cocci/` |

### Day 29 — RDMA

| What I wrote | What it actually is |
|---|---|
| `drbd_transport_rdma` is "external / LINBIT proprietary, separate repo" | **It's right here in `drbd/drbd_transport_rdma.c` (100KB)**. It is GPL and shipped with the rest of the module |

---

## C. Missing Topics That Deserve a Day

If extending the course, add these:

### Day 31 — DAX / Persistent Memory Metadata (`drbd_dax_pmem.c`)
- How DRBD detects pmem backing for metadata
- Cache-line flush (`clwb`) instead of `REQ_FUA`
- AL transaction in pmem: byte-addressable atomic update
- Performance impact: AL miss penalty drops from ~50µs to ~100ns

### Day 32 — Load-Balancing TCP Transport (`drbd_transport_lb-tcp.c`)
- Multiple parallel TCP connections to one peer
- Round-robin or hash-based packet distribution
- Aggregating throughput on 100GbE+ links
- Differences from path-bonding

### Day 33 — DRBD 8.4 Compatibility Mode (`drbd_legacy_84.c`)
- Mixed-version clusters during upgrade
- DRBD 8 protocol packet translation
- Negotiating down to legacy capabilities
- Limitations of mixed-version operation

### Day 34 — Refcount Leak Debugging (`drbd_kref_debug.c`)
- The `kref_debug` infrastructure
- How to enable at compile time
- Reading the debug output to find leaked references
- Common leak patterns and fixes

### Day 35 — Peer ACK Forwarding (`struct drbd_peer_ack`, `queued_mask`, `pending_mask`)
- How ACKs propagate in 3+ node clusters
- Why the ACK might need to traverse multiple nodes
- The `peer_ack_req` mechanism

---

## D. How to Use This Errata

1. When following any day, **start by checking the actual file** with `ls drbd/` and `grep -n` to see if functions/files exist as described.
2. The general flow and concepts are correct, but exact function names and file locations need verification.
3. The **best learning approach**: treat each day's "Hands-On Exercises" as the primary content. The narrative gives you a roadmap, but the exercises (which require running `grep` against the real source) are where you'll discover the truth.

---

## E. Verification Commands You Should Run on Day 1

```bash
git clone https://github.com/ChenFanTony/drbd.git -b drbd-9.2
cd drbd
git submodule update --init    # ← critical, gets drbd-headers

# Verify file inventory matches errata document above:
ls drbd/*.c | sort

# Verify drbd-headers populated:
ls drbd/drbd-headers/

# Find where drbd_worker actually is:
grep -rn "^int drbd_worker\b" drbd/

# Find where drbd_make_request actually is:
grep -rn "drbd_submit_bio\b\|__drbd_make_request\b" drbd/*.c | head -5

# Confirm the rq_state field structure:
grep -n "local_rq_state\|net_rq_state" drbd/drbd_int.h drbd/drbd_req.h
```

---

## F. Severity Summary

| Severity | Issue | Day(s) Affected |
|---|---|---|
| 🔴 High | `drbd_worker.c` does not exist | Day 11, others reference it |
| 🔴 High | `lru_cache.c` location wrong | Day 13 |
| 🔴 High | `rq_state[]` array structure wrong | Day 4, Day 22, Day 24 |
| 🔴 High | Entry point is `drbd_submit_bio`, not `drbd_make_request` | Day 1, Day 4, Day 24 |
| 🔴 High | `drbd_transport_rdma.c` is in repo, not external | Day 29 |
| 🟡 Medium | `e_end_block()` runs in workqueue, not sender thread | Days 5, 6, 24 |
| 🟡 Medium | Multiple AL functions instead of single `drbd_al_begin_io()` | Day 8 |
| 🟡 Medium | `drbd_disconnect()` is `conn_disconnect()` | Day 7, Day 28 |
| 🟡 Medium | `drbd_pp_pool` renamed to `drbd_buffer_page_pool` | Day 1 |
| 🟡 Medium | Compat layer in `drbd-kernel-compat/`, uses Coccinelle | Day 23 |
| 🟢 Low | `HISTORY_UUIDS` value details | Day 17 |
| 🟢 Low | Several missed files (`drbd_dax_pmem.c`, etc.) | All |

## G. What Has Been Fixed in the Source Files (as of this version)

The following corrections have been **directly applied** to the day files:

✅ **Day 1** — `drbd_ops` vtable (uses `.submit_bio`); mempool init API; both page pools (`drbd_md_io_page_pool`, `drbd_buffer_page_pool`); file inventory updated; submodule note added
✅ **Day 4** — Entry point `drbd_submit_bio` → `__drbd_make_request`; `local_rq_state` + `net_rq_state[]` struct; completion_ref-based completion model
✅ **Day 5** — `ack_sender` workqueue documented as second packet producer
✅ **Day 6** — `e_end_block()` runs in `ack_sender` workqueue, not sender thread
✅ **Day 7** — `drbd_disconnect()` → `conn_disconnect()`
✅ **Day 8** — Three-function AL API: `_fastpath`, `_nonblock`, `_commit`; `drbd_al_complete_io()` returns `bool`
✅ **Day 11** — Prominent file-name correction note at top; `ack_sender` workqueue added as 4th thread; `drbd_worker.c` references replaced with `drbd_sender.c`
✅ **Day 13** — `lru_cache.c` location corrected to `drbd/drbd-kernel-compat/lru_cache.c`
✅ **Day 22** — `rq_state` struct shape rewritten; fan-out write code corrected
✅ **Day 24** — Thread map updated to show `ack_sender`; phase 1 with `__drbd_make_request`; phase 2 with `completion_ref`; phase 6 in `ack_sender` workqueue context; phase 7 with refcount-based completion; lock summary updated to include per-request `rq_lock` and `bm_lock`
✅ **Day 28** — `drbd_disconnect()` → `conn_disconnect()`
✅ **Day 29** — RDMA correctly identified as in-tree (not external); `drbd_transport_lb-tcp.c` mentioned
✅ **Day 30** — Architecture diagram updated; key-functions table updated to show entry-point pair

✅ **Across all days** — Path corrections applied:
  - `drbd/drbd_worker.c` → `drbd/drbd_sender.c`
  - `drbd/lru_cache.c` → `drbd/drbd-kernel-compat/lru_cache.c`
  - `drbd/drbd_protocol.h` → `drbd/drbd-headers/drbd_protocol.h`
  - `drbd/drbd_strings.c` → `drbd/drbd-headers/drbd_strings.c`
  - `drbd/linux/drbd*.h` → `drbd/drbd-headers/linux/drbd*.h`
  - `drbd_pp_pool` → `drbd_buffer_page_pool`
  - `w_make_resync_request()` → `make_resync_request()`
  - `drbd_make_request()` → `__drbd_make_request()` (in prose contexts)
  - `drbd_disconnect()` → `conn_disconnect()`
  - `drbd_disconnect_or_abort()` → `change_cstate(C_NETWORK_FAILURE, CS_HARD)` (function never existed)

⚠️ **Still documented but not patched** (would require new days to cover properly):
  - `drbd_dax_pmem.c` — persistent memory metadata
  - `drbd_kref_debug.c` + `kref_debug.c` — refcount leak debugging
  - `drbd_legacy_84.c` — DRBD 8.4 mixed-version operation
  - `drbd_transport_lb-tcp.c` — load-balancing TCP transport (mentioned briefly in Day 29)
  - The `peer_ack` forwarding mechanism (`struct drbd_peer_ack`, `queued_mask`, `pending_mask`)
  - `drbd-kernel-compat/cocci/*.spatch` — Coccinelle-driven compat generation

If you want a full Day 31–35 covering the missing topics, request them explicitly.
