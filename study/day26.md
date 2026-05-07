# Day 26 — TLS Encryption in DRBD 9.2: kTLS, Certificate Config & the Crypto Path

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_transport_tcp.c`, `drbd/drbd_int.h`, kernel TLS docs

---

## 1. Why Encrypt DRBD Traffic?

DRBD replicates raw block data over TCP. Without encryption:
- Any node on the replication network can read/inject all block writes
- Particularly dangerous in multi-tenant or cloud environments

DRBD 9.2 adds **kernel TLS (kTLS)** support — TLS termination happens inside the kernel, so encrypted data never copies through userspace.

```bash
grep -n "tls\|TLS\|SSL\|crypto.*tcp\|ktls\|KTLS" drbd/drbd_transport_tcp.c | head -20
grep -n "drbd_tls\|tls_handshake\|tlshd" drbd/drbd_int.h drbd/drbd_transport_tcp.c | head -20
```

---

## 2. kTLS Architecture

```
Without kTLS:                        With kTLS:
┌────────────────┐                   ┌────────────────┐
│ DRBD kernel    │                   │ DRBD kernel    │
│ drbd_send_page │                   │ drbd_send_page │
└───────┬────────┘                   └───────┬────────┘
        │ plaintext                          │ plaintext
┌───────▼────────┐                   ┌───────▼────────┐
│ TCP socket     │                   │ kTLS layer     │ ← encrypts in-kernel
└───────┬────────┘                   └───────┬────────┘
        │ plaintext TCP                      │ encrypted TCP
    NETWORK                              NETWORK
```

kTLS uses `SOL_TLS` socket option to install encryption keys directly into the kernel TCP stack. After setup, `send()`/`recv()` operations automatically encrypt/decrypt.

```bash
# Kernel config required:
grep -r "CONFIG_TLS\b" /boot/config-$(uname -r) 2>/dev/null || \
    grep "CONFIG_TLS" /proc/config.gz 2>/dev/null | zcat | grep TLS
```

---

## 3. The TLS Handshake — `tlshd` Userspace Daemon

DRBD 9.2 does NOT implement TLS handshake itself — it uses a userspace daemon (`tlshd`) from the `ktls-utils` package:

```bash
grep -n "tlshd\|tls_handshake\|handshake_req\|AF_KTLS\|net-namespace" drbd/drbd_transport_tcp.c | head -20
```

The flow:

```
1. DRBD establishes a plain TCP connection (as always)

2. DRBD requests TLS upgrade via kernel handshake API:
   → kernel sends netlink message to tlshd daemon
   → tlshd does TLS 1.3 handshake (using OpenSSL/GnuTLS in userspace)
   → tlshd installs session keys into kernel via setsockopt(SOL_TLS, ...)
   → tlshd signals completion back to kernel

3. DRBD's send/recv now automatically encrypted
   → drbd_send_page() → TCP → kTLS encrypts transparently
   → drbd_recv() → kTLS decrypts transparently
```

```bash
grep -n "tls_client_hello_x509\|tls_server_hello_x509\|handshake_req_alloc\b" \
    drbd/drbd_transport_tcp.c | head -10
```

---

## 4. TLS Configuration in `drbd.conf`

```bash
grep -n "tls\|tlshd\|net.conf.*tls\|cert\|key\b\|ca.cert" drbd/drbd_nl.c drbd/linux/drbd_genl.h | head -20
```

```
resource r0 {
  net {
    tls yes;
    # tlshd reads certificates from system keyring or files:
    # /etc/drbd.d/certs/drbd-client.crt
    # /etc/drbd.d/certs/drbd-client.key
    # /etc/drbd.d/certs/ca.crt
  }
}
```

The `net_conf` struct gains TLS fields:

```bash
grep -n "tls\b\|use_tls\b" drbd/drbd_int.h drbd/linux/drbd_genl.h | head -10
```

```c
// net_conf (from drbd_genl.h):
// __flg_field(5, tls)    ← boolean: enable TLS
// The certificate paths are configured via tlshd.conf, not drbd.conf
```

---

## 5. `dtt_connect()` with TLS — Modified Flow

```bash
grep -n "tls\b\|tls_handshake\|dtt_tls_handshake\b\|upgrade.*tls" drbd/drbd_transport_tcp.c | head -20
```

```c
// drbd_transport_tcp.c: dtt_connect() with TLS
static int dtt_connect(struct drbd_transport *transport)
{
    // ... normal TCP connect (same as Day 16) ...

    // If TLS enabled:
    if (net_conf->tls) {
        err = dtt_tls_handshake(tcp_transport, net_conf);
        if (err) {
            // TLS handshake failed — disconnect
            sock_release(tcp_transport->stream[DATA_STREAM]);
            return err;
        }
        // After this point: all send/recv are encrypted
    }

    return 0;
}
```

```bash
grep -n -A 60 "^static int dtt_tls_handshake\b\|dtt_start_handshake\b" drbd/drbd_transport_tcp.c
```

```c
static int dtt_tls_handshake(struct drbd_tcp_transport *tcp_transport,
                               struct net_conf *net_conf)
{
    struct socket *sock = tcp_transport->stream[DATA_STREAM];
    struct tls_handshake_args args = {
        .ta_sock    = sock,
        .ta_done    = dtt_tls_handshake_done,   // callback
        .ta_data    = tcp_transport,
        .ta_timeout_ms = TLS_HANDSHAKE_TIMEOUT,
    };

    // Request handshake from tlshd via kernel API
    if (tcp_transport->is_server)
        err = tls_server_hello_x509(&args, GFP_KERNEL);
    else
        err = tls_client_hello_x509(&args, GFP_KERNEL);

    if (err)
        return err;

    // Wait for completion (tlshd signals via ta_done callback)
    wait_for_completion_timeout(&tcp_transport->tls_handshake_done,
                                 msecs_to_jiffies(TLS_HANDSHAKE_TIMEOUT));

    return tcp_transport->tls_err;
}
```

---

## 6. Key Installation — `setsockopt(SOL_TLS)`

After `tlshd` completes the handshake, it installs session keys:

```bash
# This happens in tlshd userspace, but the kernel-side result is:
# The socket now has kTLS TX and RX contexts installed
# Verified by:
grep -n "TLS_TX\|TLS_RX\|SOL_TLS\|tls_set_device_offload" \
    /usr/src/linux-headers-$(uname -r)/include/linux/tls.h 2>/dev/null | head -10
```

After key installation, the kernel's TLS layer:
1. **TX path:** `send()` → TLS record encryption → TCP send buffer
2. **RX path:** TCP receive buffer → TLS record decryption → `recv()` data

DRBD's `drbd_send_page()` and `drbd_recv()` code does NOT change — kTLS is fully transparent.

---

## 7. Authentication: DRBD's Existing CRAM-HMAC vs TLS

DRBD has two authentication layers:

| Layer | Mechanism | Config | Purpose |
|---|---|---|---|
| **CRAM-HMAC** | Challenge-response (Day 7) | `cram-hmac-alg sha256; shared-secret "x";` | Mutual authentication at connect time |
| **TLS x.509** | Certificate-based | `tls yes;` + cert files | Encryption + certificate-based auth |

They can be used together (defense in depth) or separately:

```bash
grep -n "cram.hmac\|shared.secret\|drbd_do_auth\b" drbd/drbd_receiver.c drbd/drbd_nl.c | head -10
```

CRAM-HMAC runs AFTER TLS setup (so the shared secret itself is encrypted in transit when TLS is enabled).

---

## 8. Hardware TLS Offload — `TLS_HW`

```bash
grep -n "TLS_HW\|NIC.*tls\|tls_offload\b" drbd/drbd_transport_tcp.c | head -10
```

Modern NICs (Mellanox ConnectX-5+, Intel E810) support TLS encryption in hardware. kTLS automatically uses hardware offload when available:

```
Without HW offload: CPU encrypts → NIC sends
With HW offload:    CPU sends plaintext record metadata → NIC encrypts+sends
```

DRBD benefits automatically — no code changes needed. The kTLS layer handles offload selection.

---

## 9. Diagnosing TLS Issues

```bash
# Check if TLS is active on a connection:
cat /sys/kernel/debug/drbd/r0/connections/peer1/transport
# Should show: tls: enabled, cipher: AES-256-GCM-SHA384 (or similar)

# Check tlshd daemon:
systemctl status tlshd
journalctl -u tlshd -n 50

# Check kernel TLS module:
lsmod | grep tls
# Should show: tls  <size>  2 (used by drbd_transport_tcp and tls offload)

# If TLS handshake fails, DRBD logs:
dmesg | grep -i "drbd.*tls\|tls.*drbd"
```

```bash
grep -n "drbd_err.*tls\|drbd_warn.*tls\|drbd_info.*tls" drbd/drbd_transport_tcp.c | head -10
```

---

## 10. The `dtt_tls_handshake_done` Callback

```bash
grep -n -A 20 "^static void dtt_tls_handshake_done\b" drbd/drbd_transport_tcp.c
```

```c
static void dtt_tls_handshake_done(void *data, int status,
                                    key_serial_t peerid)
{
    struct drbd_tcp_transport *tcp_transport = data;

    tcp_transport->tls_err = status;  // 0 = success, negative = error

    if (status == 0) {
        // Handshake succeeded
        // peerid: key serial of peer's certificate (for peer identity verification)
        tcp_transport->tls_peerid = peerid;
    }

    complete(&tcp_transport->tls_handshake_done);
    // → wakes dtt_tls_handshake() which was waiting
}
```

---

## 11. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Build DRBD with TLS support
```bash
# Check kernel TLS support:
grep CONFIG_TLS /boot/config-$(uname -r)
# If =m or =y: kTLS is available

# Install tlshd:
apt install ktls-utils   # Debian/Ubuntu
# or build from: https://github.com/oracle/ktls-utils

# Check tlshd configuration:
cat /etc/tlshd.conf
```

### Exercise 2 (40 min): Read `tls_client_hello_x509()` kernel API
```bash
# Find the kernel header:
find /usr/src/linux-headers-$(uname -r) -name "handshake.h" 2>/dev/null
grep -n "tls_client_hello_x509\|tls_server_hello_x509\|tls_handshake_args" \
    /usr/src/linux-headers-$(uname -r)/include/net/handshake.h 2>/dev/null | head -20
```
What is the `ta_done` callback signature? What does `peerid` represent?

### Exercise 3 (40 min): Trace the full TLS-enabled connect sequence
Starting from `drbd_adm_connect()` in `drbd_nl.c`:
1. `net_conf->tls = true` (parsed from drbd.conf)
2. `drbd_thread_start(receiver)` → `drbd_receiver()` → `drbd_connect()`
3. `dtt_connect()` → TCP establish → `dtt_tls_handshake()`
4. `tls_client_hello_x509()` → kernel → tlshd handshake
5. `dtt_tls_handshake_done()` callback
6. `drbd_do_handshake()` (DRBD-level, now over encrypted channel)

### Exercise 4 (35 min): Understand certificate validation in DRBD
```bash
grep -n "peerid\|peer_certificate\|peer_identity\|tls_peerid" drbd/drbd_transport_tcp.c | head -10
```
After TLS handshake, `peerid` is returned. How could DRBD use this to verify the peer's identity beyond just "valid certificate"? Is this currently implemented?

### Exercise 5 (35 min): Performance impact of TLS
Theoretical analysis:
- AES-256-GCM throughput on modern x86: ~10 GB/s per core (with AES-NI)
- DRBD replication bandwidth: typically 1–10 GbE = 125 MB/s – 1.25 GB/s
- At what bandwidth does TLS become a CPU bottleneck?
- How does hardware TLS offload change this?

```bash
# Measure AES throughput:
openssl speed -evp AES-256-GCM
```

---

## Summary

DRBD 9.2 integrates with kernel TLS (kTLS) to provide transparent block-level encryption of replication traffic. The TLS handshake is delegated to the `tlshd` userspace daemon, which uses x.509 certificates and the kernel's handshake API (`tls_client/server_hello_x509()`). After handshake completion, session keys are installed into the kernel's TLS layer, and all subsequent `send()`/`recv()` operations are encrypted transparently — DRBD's data-path code (`drbd_send_page()`, `dtt_recv()`) requires no changes. CRAM-HMAC authentication and TLS can coexist for defense in depth.

**Next:** Day 27 — Congestion & flow control deep dive: epoch management, `max-epoch-size`, write ordering under pressure, and how DRBD handles a slow secondary.
