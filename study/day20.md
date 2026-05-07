# Day 20 — Online Verify: Hash Computation, Mismatch Handling & `ov_left` Tracking

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_worker.c`, `drbd/drbd_receiver.c`, `drbd/drbd_sender.c`, `drbd/drbd_int.h`

---

## 1. What Online Verify Does

`drbdadm verify r0` initiates a background comparison of two nodes' data **without interrupting I/O**. Instead of transferring data, it:

1. SyncSource reads a block → computes a hash
2. SyncTarget reads the same block → computes a hash
3. Both hashes are compared
4. On mismatch: the block is marked OOS in the bitmap (triggering resync later)

The verify runs at the configured `verify-alg` hash algorithm (e.g., MD5, SHA-256, CRC32C).

**Key property:** Verify never writes. It is read-only and safe to run on live systems.

---

## 2. Verify State Machine Entry Points

```bash
grep -n "L_VERIFY_S\|L_VERIFY_T\|drbd_adm_start_ov\b\|start_ov\b" \
    drbd/drbd_state.c drbd/drbd_nl.c drbd/drbd_worker.c | head -20
```

Triggered by `drbdadm verify r0`:
```
drbd_adm_start_ov(skb, info)     [drbd_nl.c]
    │
    ├─ Parse: start_sector, stop_sector (optional range)
    ├─ peer_device->ov_left     = sectors_to_verify
    ├─ peer_device->ov_position = start_sector  // verify cursor
    │
    ├─ change_repl_state(peer_device, L_VERIFY_S, CS_VERBOSE)
    │   (or L_VERIFY_T — which side initiates determines which side is S vs T)
    │
    └─ Trigger first burst:
        drbd_queue_work(connection->sender_work, &peer_device->ov_work)
```

The two sides:

| State | Role | Action |
|---|---|---|
| `L_VERIFY_S` | Initiator / "Source" | Sends `P_OV_REQUEST` asking peer to hash block |
| `L_VERIFY_T` | Target | Receives request, hashes block, sends `P_OV_REPLY` |

Then Verify-S reads its own copy, compares hashes, sends `P_OV_RESULT`.

---

## 3. Verify-S: Sending `P_OV_REQUEST`

```bash
grep -n "w_make_ov_request\b\|P_OV_REQUEST\b" drbd/drbd_worker.c drbd/drbd_sender.c | head -10
grep -n -A 80 "^static int w_make_ov_request\b" drbd/drbd_worker.c
```

```c
static int w_make_ov_request(struct drbd_work *w, int cancel)
{
    struct drbd_peer_device *peer_device = container_of(w, ...);
    sector_t sector = peer_device->ov_position;
    int size;

    while (peer_device->ov_left > 0) {
        size = min_t(sector_t, peer_device->ov_left,
                     OV_REQUEST_SIZE / 512);  // typical: 64 sectors = 32 KiB

        // Rate limiting: check against in-flight count
        if (atomic_read(&peer_device->ov_left) > max_burst)
            break;

        // Allocate peer_req for local read
        peer_req = drbd_alloc_peer_req(peer_device, sector, size, ...);
        peer_req->w.cb = w_e_end_ov_req;  // callback after local read

        // Send P_OV_REQUEST to peer (tell them which block to hash)
        err = drbd_send_drequest(peer_device, P_OV_REQUEST,
                                  sector, size * 512, (u64)(uintptr_t)peer_req);

        // Submit local read
        drbd_submit_peer_request(device, peer_req, REQ_OP_READ, ...);

        peer_device->ov_position += size;
        peer_device->ov_left     -= size;
    }

    if (peer_device->ov_left == 0)
        drbd_resync_finished(peer_device, D_MASK);  // verify complete
    else
        mod_timer(&peer_device->resync_timer, jiffies + SLEEP_TIME);

    return 0;
}
```

---

## 4. Verify-T: Receiving `P_OV_REQUEST` and Computing Hash

```bash
grep -n -A 60 "^static int receive_OVRequest\b" drbd/drbd_receiver.c
```

```c
static int receive_OVRequest(struct drbd_connection *connection,
                               struct packet_info *pi)
{
    struct p_block_req *p = pi->data;
    sector_t sector = be64_to_cpu(p->sector);
    int size         = be32_to_cpu(p->blksize);

    // Allocate peer_req for local read
    peer_req = drbd_alloc_peer_req(peer_device, sector, size, ...);
    peer_req->w.cb = w_e_end_ov_reply;  // callback after read + hash

    // Submit local read
    drbd_submit_peer_request(device, peer_req, REQ_OP_READ, ...);

    return 0;
}
```

After the local read completes, `w_e_end_ov_reply()` runs:

```bash
grep -n -A 80 "^static int w_e_end_ov_reply\b" drbd/drbd_worker.c
```

```c
static int w_e_end_ov_reply(struct drbd_work *w, int cancel)
{
    struct drbd_peer_request *peer_req = container_of(w, ...);
    struct digest_info *di;

    // 1. Compute hash of the data pages
    di = kmalloc(sizeof(*di) + digest_size, GFP_NOIO);
    drbd_csum_pages(peer_device->connection->verify_tfm,
                    peer_req->page_chain.head,
                    peer_req->i.size,
                    di->digest);
    // verify_tfm = crypto_shash handle for verify-alg (MD5/SHA256/etc.)

    // 2. Send P_OV_REPLY carrying the hash
    drbd_send_ack(peer_device, P_OV_REPLY, peer_req);
    // Actually: drbd_send_drequest_csum() sends sector + hash

    drbd_free_peer_req(device, peer_req);
    return 0;
}
```

---

## 5. Hash Computation — `drbd_csum_pages()`

```bash
grep -n "drbd_csum_pages\b\|drbd_csum_bio\b\|verify_tfm\b\|integrity_tfm\b" \
    drbd/drbd_worker.c drbd/drbd_receiver.c drbd/drbd_main.c | head -20
grep -n -A 40 "^void drbd_csum_pages\b\|^static void drbd_csum_pages\b" drbd/drbd_worker.c
```

```c
void drbd_csum_pages(struct crypto_shash *tfm,
                      struct page *chain_head,
                      size_t size,
                      void *digest)
{
    SHASH_DESC_ON_STACK(desc, tfm);
    desc->tfm = tfm;

    crypto_shash_init(desc);

    struct page *page = chain_head;
    size_t remaining = size;
    while (remaining > 0) {
        size_t chunk = min(remaining, (size_t)PAGE_SIZE);
        crypto_shash_update(desc, page_address(page), chunk);
        page = page->private;  // next page in chain
        remaining -= chunk;
    }

    crypto_shash_final(desc, digest);
}
```

The hash algorithm is configured as:
```bash
grep -n "verify.alg\|verify_alg\|verify_tfm\b" drbd/drbd_int.h drbd/drbd_nl.c | head -15
# e.g., net { verify-alg md5; }
# or:   net { verify-alg sha256; }
# or:   net { verify-alg crc32c; }
```

---

## 6. Verify-S: Receiving `P_OV_REPLY` and Comparing Hashes

```bash
grep -n -A 80 "^static int w_e_end_ov_req\b" drbd/drbd_worker.c
```

After the local read on the S-side finishes, `w_e_end_ov_req()` is called. It waits for the corresponding `P_OV_REPLY` from the T-side:

```c
static int w_e_end_ov_req(struct drbd_work *w, int cancel)
{
    struct drbd_peer_request *peer_req = container_of(w, ...);

    // 1. Compute hash of local data
    drbd_csum_pages(peer_device->connection->verify_tfm,
                    peer_req->page_chain.head, peer_req->i.size,
                    local_digest);

    // 2. Wait for peer's P_OV_REPLY (stored in peer_req->digest)
    //    (received by receive_OVReply() and stored on the peer_req)

    // 3. Compare digests
    eq = (memcmp(local_digest, peer_req->digest, digest_size) == 0);

    // 4a. Match: block is in sync
    if (eq) {
        // Nothing to do — ov_left already decremented
    }

    // 4b. Mismatch: block differs!
    else {
        drbd_ov_out_of_sync_found(peer_device, peer_req->i.sector,
                                   peer_req->i.size);
        // → drbd_bm_set_bits(): mark OOS in bitmap
        // → peer_device->ov_out_of_sync += size >> 9
        // → optionally log: drbd_warn(..., "Verify found difference...")
    }

    drbd_free_peer_req(device, peer_req);
    return 0;
}
```

---

## 7. `receive_OVReply()` — Storing the Peer's Hash

```bash
grep -n -A 50 "^static int receive_OVReply\b" drbd/drbd_receiver.c
```

```c
static int receive_OVReply(struct drbd_connection *connection,
                             struct packet_info *pi)
{
    // The P_OV_REPLY carries: sector, block_id (= drbd_peer_request ptr), hash
    struct p_block_ack *p = pi->data;
    u64 block_id  = p->block_id;
    sector_t sector = be64_to_cpu(p->sector);

    // Find the matching peer_req by block_id (same pointer trick as P_WRITE_ACK)
    peer_req = (struct drbd_peer_request *)(uintptr_t)block_id;

    // Store the peer's hash in the peer_req
    memcpy(peer_req->digest, pi->data + sizeof(*p), digest_size);
    set_bit(EE_HAS_DIGEST, &peer_req->flags);

    // Wake w_e_end_ov_req() which is waiting for this
    drbd_queue_work(&connection->sender_work, &peer_req->w);

    return 0;
}
```

---

## 8. `ov_left` Tracking and Progress Reporting

```bash
grep -n "ov_left\b\|ov_out_of_sync\b\|ov_position\b\|ov_last_oos_size\b" \
    drbd/drbd_int.h drbd/drbd_worker.c | head -20
```

```c
// In struct drbd_peer_device:
sector_t ov_left;           // sectors remaining to verify
sector_t ov_position;       // current sector cursor
u64 ov_out_of_sync;         // total out-of-sync sectors found
sector_t ov_last_oos_size;  // size of last OOS block found (for logging)
sector_t ov_last_oos_start; // sector of last OOS block
```

Progress visible in `/proc/drbd` during verify:
```
 0: cs:VerifyS ro:Primary/Secondary ds:UpToDate/UpToDate C r-----
    ...
    ov: used:0 hits:0 misses:0 ...
    ov_left:20971520 oos:512
```

And via `drbdsetup events2`:
```
peer-device role:Primary peer-role:Secondary replication:VerifyS ...
```

---

## 9. Verify Completion: `drbd_ov_out_of_sync_print()` + State Change

```bash
grep -n "drbd_ov_out_of_sync_print\b\|ov_out_of_sync\b" drbd/drbd_worker.c | head -10
grep -n -A 20 "^static void drbd_ov_out_of_sync_print\b" drbd/drbd_worker.c
```

At verify completion, `drbd_resync_finished()` is called (same function as for resync!):

```c
// drbd_worker.c: w_make_ov_request() when ov_left == 0:
if (peer_device->ov_out_of_sync > 0) {
    drbd_warn(peer_device,
              "Online verify found %llu 4k block(s) out of sync!\n",
              peer_device->ov_out_of_sync >> 3);
}
drbd_resync_finished(peer_device, D_MASK);
```

After verify, the OOS blocks found are in the bitmap. The admin can then trigger a targeted resync:
```bash
drbdadm resync-from peer r0
# or simply wait — DRBD will resync automatically on reconnect
```

---

## 10. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Read `w_make_ov_request()`, `w_e_end_ov_req()`, `w_e_end_ov_reply()` consecutively
```bash
grep -n "w_make_ov_request\|w_e_end_ov_req\|w_e_end_ov_reply" drbd/drbd_worker.c
```
For each function, answer: what thread runs it, what lock (if any) is held, what does it send over the network?

### Exercise 2 (40 min): Trace the full verify packet exchange
Draw a sequence diagram for verifying one 32 KiB block showing:
1. `P_OV_REQUEST` (Verify-S → Verify-T)
2. Local reads on both sides (concurrent)
3. `P_OV_REPLY` (Verify-T → Verify-S)
4. Hash comparison on Verify-S side
5. `P_OV_RESULT` if mismatch (optional, check if this packet exists)

```bash
grep -n "P_OV_RESULT\b" drbd/drbd_protocol.h drbd/drbd_receiver.c drbd/drbd_worker.c
```

### Exercise 3 (35 min): Understand `verify-alg` initialization
```bash
grep -n "verify_tfm\b\|crypto_alloc_shash\b\|verify.alg\b" drbd/drbd_nl.c drbd/drbd_main.c | head -15
```
- Where is `crypto_alloc_shash()` called for the verify algorithm?
- When is `verify_tfm` freed?
- What happens if the configured algorithm (e.g., "sha512") is not available in the kernel?

### Exercise 4 (35 min): Find the rate limiting in verify
```bash
grep -n "ov_in_flight\|c_sync_rate\|OV_REQUEST_SIZE\|verify.*rate" drbd/drbd_worker.c | head -15
```
Does verify have its own rate controller, or does it reuse `drbd_rs_controller()`? What limits the verify throughput?

### Exercise 5 (40 min): Simulate a verify on a small device
If you have a DRBD test setup:
```bash
# Artificially corrupt one block on secondary:
sudo dd if=/dev/urandom of=/dev/sdb seek=1000 bs=4096 count=1 conv=notrunc

# Run verify:
drbdadm verify r0

# Watch progress:
watch -n1 "cat /proc/drbd | grep -A1 'VerifyS'"

# After verify:
drbdadm status r0   # should show some oos
```
If no test setup: trace through the code path for `ov_out_of_sync > 0` and identify every function call from mismatch detection to bitmap bit set.

---

## Summary

Online verify uses two phases: the Verify-S side sends `P_OV_REQUEST` packets as it scans the device, then reads local blocks and computes hashes; the Verify-T side receives requests, reads its own blocks, computes hashes, and replies with `P_OV_REPLY`. Verify-S compares local and remote hashes — on mismatch, `drbd_bm_set_bits()` marks the block OOS. `ov_left` tracks remaining sectors; progress is visible in `/proc/drbd`. The crypto hash (`verify-alg`) is initialized with `crypto_alloc_shash()` at connection setup. Verify completion calls `drbd_resync_finished()` which transitions replication state back to `Established`.

**Next:** Day 21 — The Ahead/Behind congestion mechanism: flow control, `L_AHEAD`/`L_BEHIND` states, and `P_OUT_OF_SYNC`.
