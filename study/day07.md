# Day 7 — Connection Setup, Handshake, UUID Exchange & Reconnect

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_receiver.c`, `drbd/drbd_main.c`, `drbd/drbd_state.c`

---

## 1. The Full Connection Lifecycle

```
DRBD node A                              DRBD node B
    │                                        │
    │  drbdadm up r0                         │  drbdadm up r0
    │                                        │
    │  [receiver thread starts]              │  [receiver thread starts]
    │  drbd_receiver()                       │  drbd_receiver()
    │    └─ drbd_connect()                   │    └─ drbd_connect()
    │         ├─ try TCP connect →───────────┤─ accept incoming
    │         └─ or: listen for incoming     │
    │                                        │
    │  [TCP connection established]          │
    │    └─ drbd_do_handshake()  ←──────────→ drbd_do_handshake()
    │         ├─ P_CONNECTION_FEATURES       │
    │         ├─ P_AUTH_CHALLENGE / RESPONSE │  (if cram-hmac-alg configured)
    │         └─ P_INITIAL_META             │
    │                                        │
    │  [UUID exchange: detect sync state]    │
    │    └─ drbd_send_uuids()  ←──────────→ drbd_send_uuids()
    │         └─ P_UUIDS110                  │
    │                                        │
    │  [State: Off → WFBitMapS/T or Established]
    │                                        │
    │  [Bitmap exchange if needed]           │
    │    └─ drbd_send_bitmap() ←───────────→ receive_bitmap()
    │                                        │
    │  [State: → SyncSource/SyncTarget or Established]
```

---

## 2. `drbd_connect()` — Establishing the TCP Connection

```bash
grep -n -A 80 "^static int drbd_connect\b" drbd/drbd_receiver.c
# If not there, check:
grep -n "drbd_connect\b" drbd/drbd_receiver.c drbd/drbd_transport_tcp.c | head -10
```

DRBD uses the transport abstraction. The actual TCP connect is in `drbd_transport_tcp.c`:

```bash
grep -n -A 60 "drbd_tcp_connect\b\|\.connect\s*=" drbd/drbd_transport_tcp.c | head -60
```

The connect logic:
```c
// drbd_transport_tcp.c
static int dtt_connect(struct drbd_transport *transport)
{
    struct drbd_tcp_transport *tcp_transport = ...;

    // Try to connect (client role)
    sock = sock_create_kern(&init_net, AF_INET6, SOCK_STREAM, IPPROTO_TCP, &sock);
    kernel_connect(sock, (struct sockaddr *)&peer_addr, addrlen, 0);

    // Set socket options
    drbd_setbufsize(sock, sndbuf_size, rcvbuf_size);
    tcp_sock_set_nodelay(sock->sk);  // disable Nagle algorithm
    sk->sk_sndtimeo = connection_timeout * HZ;
    sk->sk_rcvtimeo = ping_timeout * HZ;

    // Store in transport
    tcp_transport->stream[DATA_STREAM]    = data_sock;
    tcp_transport->stream[CONTROL_STREAM] = meta_sock;
}
```

If connect fails, the thread sleeps (with exponential backoff) and retries:
```bash
grep -n "retry\|schedule_timeout\|wait_event.*connect" drbd/drbd_receiver.c | head -10
```

---

## 3. `drbd_do_handshake()` — Feature Negotiation

```bash
grep -n "drbd_do_handshake\|receive_first_packet\|drbd_send_handshake" drbd/drbd_receiver.c | head -10
grep -n -A 80 "^static int drbd_do_handshake\b" drbd/drbd_receiver.c
```

### Phase 1: Feature Exchange (`P_CONNECTION_FEATURES`)

```bash
grep -n "struct p_connection_features {" drbd/drbd_protocol.h
```

```c
struct p_connection_features {
    u32 protocol_min;    // minimum protocol version we support
    u32 feature_flags;   // bitmask of optional features
    u32 protocol_max;    // maximum protocol version we support
    // padding fields
} __packed;
```

Both sides send `P_CONNECTION_FEATURES`. They negotiate down to the minimum supported protocol. If incompatible, the connection is dropped.

### Phase 2: Authentication (`P_AUTH_CHALLENGE` / `P_AUTH_RESPONSE`)

```bash
grep -n "drbd_do_auth\|P_AUTH_CHALLENGE\|P_AUTH_RESPONSE\|cram_hmac" drbd/drbd_receiver.c | head -20
grep -n -A 60 "^static int drbd_do_auth\b" drbd/drbd_receiver.c
```

```c
static int drbd_do_auth(struct drbd_connection *connection)
{
    // 1. Send challenge: random nonce
    get_random_bytes(my_challenge, CHALLENGE_LEN);
    drbd_send_command(connection, CONTROL_STREAM,
                       P_AUTH_CHALLENGE, my_challenge, CHALLENGE_LEN);

    // 2. Receive peer's challenge
    drbd_recv_all(connection, CONTROL_STREAM, peer_challenge, CHALLENGE_LEN);

    // 3. Compute HMAC(shared_secret, peer_challenge)
    crypto_shash_init(desc);
    crypto_shash_update(desc, shared_secret, strlen(shared_secret));
    crypto_shash_update(desc, peer_challenge, CHALLENGE_LEN);
    crypto_shash_final(desc, response);

    // 4. Send our response
    drbd_send_command(connection, CONTROL_STREAM, P_AUTH_RESPONSE, response, ...);

    // 5. Receive peer's response, verify it matches
    // HMAC(shared_secret, my_challenge) ?= peer_response
}
```

---

## 4. UUID Exchange — Detecting Sync State

UUIDs are the mechanism by which DRBD determines whether two nodes have compatible data history.

```bash
grep -n "drbd_send_uuids\|receive_uuids\|P_UUIDS110" drbd/drbd_receiver.c drbd/drbd_sender.c | head -20
grep -n "struct p_uuids110 {" drbd/drbd_protocol.h
```

### The UUID Set

Each device maintains:
```c
// drbd_int.h
struct drbd_md {
    u64 current_uuid;               // changes on every primary promotion
    u64 bitmap_uuid;                // UUID of the last resync partner
    u64 history_uuids[HISTORY_UUIDS]; // ring buffer of past current UUIDs
    // ...
};
```

### The UUID Logic

After receiving the peer's UUIDs, DRBD runs:

```bash
grep -n -A 200 "^static int drbd_uuid_compare\b\|^static enum drbd_repl_state drbd_attach_handshake\b" \
    drbd/drbd_receiver.c | head -100
```

Decision tree (simplified):
```
peer.current_uuid == local.current_uuid ?
    YES → Both UpToDate? → Established (no resync needed)
          Else → negotiate who is newer

peer.current_uuid IN local.history_uuids ?
    YES → peer is OUTDATED → we are SyncSource (send data to peer)

local.current_uuid IN peer.history_uuids ?
    YES → we are OUTDATED → we are SyncTarget (receive data from peer)

No UUID match at all ?
    → Bitmap comparison
    → Or: full sync required (split brain resolution)
```

```bash
# Find the full UUID comparison logic
grep -n "drbd_uuid_compare\|uuid_match\|history_uuid" drbd/drbd_receiver.c | head -20
```

---

## 5. Bitmap Exchange — What Gets Sent

After UUID comparison determines that resync is needed but only a partial resync (not full), DRBD sends the bitmap to find which blocks actually differ:

```bash
grep -n "drbd_send_bitmap\b\|receive_bitmap\b" drbd/drbd_sender.c drbd/drbd_receiver.c | head -10
grep -n -A 80 "^int drbd_send_bitmap\b" drbd/drbd_sender.c
```

```c
int drbd_send_bitmap(struct drbd_device *device,
                     struct drbd_peer_device *peer_device)
{
    // Send the entire bitmap in chunks
    // Each chunk is P_BITMAP packet with a payload of compressed bitmap pages

    while (still_have_bits) {
        // Compress a page of bitmap bits (run-length encoding)
        len = drbd_bm_send_rle_lz4(page, &want);  // RLE + LZ4 compression

        // Send as P_BITMAP packet
        drbd_send_command(peer_device, DATA_STREAM, P_BITMAP, buf, len);
    }
    // Final: send P_BITMAP with length=0 to signal end
    drbd_send_command(peer_device, DATA_STREAM, P_BITMAP, NULL, 0);
}
```

On the receiving end:
```bash
grep -n -A 60 "^static int receive_bitmap\b" drbd/drbd_receiver.c
```

Both sides OR their bitmaps together: `local_bm |= peer_bm`. The result is the union of OOS bits — every block that either side thinks is OOS will be resynced.

---

## 6. Disconnect and Reconnect

### Detecting Disconnect

The receiver thread detects network errors via:
- Return value of `drbd_recv_all()` / `transport->ops->recv()` returning 0 (EOF) or negative
- `sk_rcvtimeo` expiring (no data received within `ping-timeout`)
- Explicit `P_PING` / `P_PING_ACK` timeout

### `drbd_disconnect()`

```bash
grep -n -A 100 "^static void drbd_disconnect\b" drbd/drbd_receiver.c
```

```c
static void drbd_disconnect(struct drbd_connection *connection)
{
    // 1. Change connection state → C_DISCONNECTING
    change_cstate(connection, C_DISCONNECTING, CS_HARD);

    // 2. Close TCP sockets
    connection->transport->ops->close(connection->transport);

    // 3. Drain all in-flight peer requests
    //    (fail them with EIO or mark OOS)
    drbd_finish_peer_reqs(connection);

    // 4. Walk transfer log: for each in-flight drbd_request,
    //    call __req_mod(SEND_CANCELED) for this peer
    tl_walk(connection, CONNECTION_LOST_WHILE_PENDING);

    // 5. Bitmap: mark all previously-OOS blocks still OOS
    //    (the write may not have made it to the peer)

    // 6. Schedule reconnect attempt
    drbd_queue_work(&connection->sender_work, &connection->connect_timer_work);
}
```

### `tl_walk()` — Walking the Transfer Log on Disconnect

```bash
grep -n -A 50 "^static void tl_walk\b\|^void tl_walk\b\|tl_walk(" drbd/drbd_req.c drbd/drbd_main.c | head -60
```

For each `drbd_request` in `resource->transfer_log`:
```c
__req_mod(req, SEND_CANCELED, peer_device, &m);
// → clears RQ_NET_PENDING for this peer
// → if all peers done AND local done: bio_endio()
// → if protocol C and peer was only ack source: suspend I/O (fencing)
```

---

## 7. Reconnect Timer

After disconnect, DRBD does not immediately retry. It uses an exponential backoff:

```bash
grep -n "drbd_connection_retry_timer\|retry_connect\|connect_timer" drbd/drbd_receiver.c drbd/drbd_main.c | head -10
```

```c
// Backoff: 1s, 2s, 4s, 8s, up to connect-int (default 10s)
// Configured by: net { connect-int 10; }
static void drbd_connection_retry_timer(struct timer_list *t)
{
    // Queue a work item that re-calls drbd_receiver()
}
```

---

## 8. Week 1 Integration — Putting It All Together

The full flow for a two-node cluster starting from scratch:

```
Both nodes: disk_state=Diskless, role=Secondary, cstate=StandAlone

Step 1: drbdadm up r0 on both nodes
    → drbd_create_resource(), drbd_create_device(), drbd_create_connection()
    → drbd_thread_start(receiver), drbd_thread_start(sender), drbd_thread_start(worker)

Step 2: TCP connection established
    → drbd_connect() in drbd_transport_tcp.c
    → cstate: StandAlone → WFConnection → Connected (TCP level)

Step 3: Handshake
    → P_CONNECTION_FEATURES exchange
    → P_AUTH_CHALLENGE / P_AUTH_RESPONSE (if configured)
    → cstate: Connected → Negotiating (internal state)

Step 4: Disk attach (drbdadm up also runs attach)
    → drbd_adm_attach() in drbd_nl.c
    → disk_state: Diskless → Attaching → Negotiating
    → drbd_al_apply_to_bm() — crash recovery from activity log
    → disk_state: Negotiating → Inconsistent (first time) or UpToDate (if history matches)

Step 5: UUID exchange
    → drbd_send_uuids110() on both sides
    → drbd_uuid_compare() determines relationship

Step 6a: First time ever (no history match)
    → repl_state: WFBitMapS / WFBitMapT
    → Bitmap exchange (both sides send full bitmap = all 1s)
    → repl_state: SyncSource / SyncTarget
    → Resync runs (Day 10)
    → repl_state: Established; disk_state: UpToDate on both

Step 6b: Reconnect after clean disconnect (UUIDs match)
    → repl_state: Established immediately
    → No resync needed

Step 7: Primary promotion
    → drbdadm primary r0
    → drbd_adm_primary() → change_role(R_PRIMARY)
    → role: Secondary → Primary
    → I/O accepted on /dev/drbd0
```

---

## 9. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Trace `drbd_do_handshake()` completely
```bash
grep -n -A 150 "drbd_do_handshake\b" drbd/drbd_receiver.c
```
Identify: every packet sent, every packet received, what state changes occur, what happens if the peer uses an incompatible version.

### Exercise 2 (45 min): Read the UUID comparison logic
```bash
grep -n "drbd_uuid_compare\|drbd_attach_handshake\|bitmap_uuid\|history_uuid" drbd/drbd_receiver.c | head -30
```
Draw a decision tree for all UUID comparison outcomes and the resulting `repl_state`.

### Exercise 3 (40 min): Trace `drbd_disconnect()` and `tl_walk()`
```bash
grep -n "tl_walk\|SEND_CANCELED\|CONNECTION_LOST" drbd/drbd_req.c drbd/drbd_main.c | head -20
```
What happens to an in-flight write when the only connected peer disconnects during protocol C replication?

### Exercise 4 (35 min): Find `receive_bitmap()` and understand RLE compression
```bash
grep -n -A 80 "^static int receive_bitmap\b" drbd/drbd_receiver.c
grep -n "drbd_bm_recv_rle\|rle_decode\|lz4" drbd/drbd_bitmap.c drbd/drbd_receiver.c
```

### Exercise 5 (30 min): Trace the reconnect timer backoff
```bash
grep -n "connect_timer\|retry_timer\|schedule_timeout.*connect" drbd/drbd_receiver.c | head -15
```
What is the minimum reconnect interval? Maximum? What config option controls it?

### Week 1 Final Challenge (30 min):
Without looking at source, draw the complete state diagram showing:
1. `cstate` transitions from StandAlone → WFConnection → Connected → Established → Disconnecting → StandAlone
2. `disk_state` transitions from Diskless → Attaching → Inconsistent → SyncTarget → UpToDate
3. `repl_state` transitions from Off → WFBitMapT → SyncTarget → Established

Then verify your diagram against the code.

---

## Summary

Connection setup proceeds through three phases: TCP establishment via the transport layer, feature/auth handshake, and UUID exchange. UUIDs determine the resync relationship. The bitmap exchange identifies exactly which blocks differ. Disconnect is handled by `drbd_disconnect()` which drains in-flight requests, walks the transfer log with `tl_walk()`, and schedules reconnect. The entire state progression from `StandAlone` to `Established` involves coordinated transitions across `cstate`, `disk_state`, and `repl_state` on both nodes simultaneously.

**Week 2 begins:** Day 8 — The Activity Log (`drbd_actlog.c`): on-disk structure, transaction format, crash recovery.
