# Day 16 — Transport Abstraction: `drbd_transport.h`, `drbd_transport.c` & TCP Implementation

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_transport.h`, `drbd/drbd_transport.c`, `drbd/drbd_transport_tcp.c`

---

## 1. Why a Transport Abstraction Layer?

DRBD 9 supports multiple network transports:
- **TCP/IP** (`drbd_transport_tcp.ko`) — default, available everywhere
- **RDMA** (separate `drbd_transport_rdma.ko`, not in this repo) — InfiniBand/RoCE

The abstraction layer lets `drbd_receiver.c` and `drbd_sender.c` call generic send/recv operations without knowing whether TCP sockets or RDMA QPs are underneath.

---

## 2. `struct drbd_transport_ops` — The vtable

```bash
grep -n "struct drbd_transport_ops {" drbd/drbd_transport.h
# Read every function pointer
```

```c
struct drbd_transport_ops {
    // Lifecycle
    int  (*init)(struct drbd_transport *);
    void (*free)(struct drbd_transport *, enum drbd_tr_free_op);
    int  (*connect)(struct drbd_transport *);
    bool (*stream_ok)(struct drbd_transport *, enum drbd_stream);
    void (*close)(struct drbd_transport *);

    // Sending
    int  (*send_page)(struct drbd_transport *, enum drbd_stream,
                      struct page *, int offset, size_t size, unsigned msg_flags);
    int  (*send_zc_bio)(struct drbd_transport *, struct bio *);
    // zc = zero-copy: use splice/sendfile instead of copy

    // Receiving
    int  (*recv)(struct drbd_transport *, enum drbd_stream,
                 void **buf, size_t size, int flags);
    int  (*recv_pages)(struct drbd_transport *, struct drbd_page_chain_head *,
                       size_t size);
    // recv_pages: receive directly into a page chain (zero-copy receive)

    // Flow control / peeking
    void (*flush)(struct drbd_transport *, enum drbd_stream);
    bool (*hint)(struct drbd_transport *, enum drbd_stream,
                 enum drbd_tr_hints);
    // hints: QUICKACK, CORK, UNCORK, NODELAY

    // Statistics / debugging
    void (*debugfs_show)(struct drbd_transport *, struct seq_file *);
    int  (*net_conf_change)(struct drbd_transport *,
                            struct net_conf *);
    void (*set_rcvtimeo)(struct drbd_transport *, enum drbd_stream,
                          long timeout);
    long (*get_rcvtimeo)(struct drbd_transport *, enum drbd_stream);
    int  (*path_added)(struct drbd_transport *, struct drbd_path *);
    void (*path_removed)(struct drbd_transport *, struct drbd_path *);
};
```

```bash
grep -n "drbd_stream\b\|enum drbd_stream" drbd/drbd_transport.h
# DATA_STREAM = 0, CONTROL_STREAM = 1
```

---

## 3. `struct drbd_transport` — The Instance

```bash
grep -n "struct drbd_transport {" drbd/drbd_transport.h
```

```c
struct drbd_transport {
    const struct drbd_transport_ops *ops;  // vtable pointer
    struct drbd_transport_class *class;    // which transport class (tcp / rdma)
    struct list_head paths;                // list of drbd_path (IP address pairs)
    struct net_conf *net_conf;             // current net configuration
    unsigned int flags;                    // RESOLVE_CONFLICTS, etc.
};
```

The transport is embedded in the connection-specific transport struct:
```bash
grep -n "struct drbd_tcp_transport {" drbd/drbd_transport_tcp.c
```

```c
// drbd_transport_tcp.c
struct drbd_tcp_transport {
    struct drbd_transport transport;       // MUST be first — for container_of()
    spinlock_t paths_lock;
    struct socket *stream[2];             // [DATA_STREAM] and [CONTROL_STREAM]
    void *rbuf[2];                        // receive buffers (one per stream)
};
```

---

## 4. `struct drbd_path` — An IP Address Pair

```bash
grep -n "struct drbd_path {" drbd/drbd_transport.h
```

```c
struct drbd_path {
    struct sockaddr_storage my_addr;      // local address (from drbd.conf: address)
    struct sockaddr_storage peer_addr;    // peer address (from drbd.conf: address)
    int my_addr_len;
    int peer_addr_len;
    struct list_head list;                // node in transport->paths
    struct kref kref;
    bool established;                     // is this path currently connected?
};
```

DRBD 9.2 supports **multiple paths** per connection (multipath TCP or bonding):
```bash
grep -n "path_added\|path_removed\|list_for_each_entry.*path" drbd/drbd_transport.c drbd/drbd_transport_tcp.c | head -10
```

---

## 5. Transport Registration — `drbd_transport_class`

```bash
grep -n "struct drbd_transport_class {" drbd/drbd_transport.h
grep -n "drbd_register_transport_class\b\|drbd_unregister_transport_class\b" drbd/drbd_transport.c
```

```c
struct drbd_transport_class {
    const char *name;               // "tcp" or "rdma"
    const int instance_size;        // sizeof(drbd_tcp_transport) or similar
    const int path_instance_size;
    struct list_head list;          // node in global drbd_transport_classes
    struct module *module;

    // Factory function: allocate and init a transport instance
    int (*init)(struct drbd_transport *);
};
```

Module init of `drbd_transport_tcp.ko`:
```bash
grep -n "__init dtt_init\b\|drbd_register_transport_class\b" drbd/drbd_transport_tcp.c
```

```c
static int __init dtt_init(void)
{
    return drbd_register_transport_class(&tcp_transport_class,
                                          DRBD_TRANSPORT_API_VERSION,
                                          sizeof(struct drbd_transport_class));
}
module_init(dtt_init);
```

---

## 6. `dtt_connect()` — Establishing TCP Connections

```bash
grep -n -A 200 "^static int dtt_connect\b" drbd/drbd_transport_tcp.c
```

Full annotated trace:

```
dtt_connect(transport)
│
├─ 1. Get path (first path in transport->paths)
│   └── path = list_first_entry(&transport->paths, struct drbd_path, list)
│
├─ 2. Create data socket (TCP client or server)
│   └── dtt_create_listener() or dtt_try_connect()
│       │
│       ├─ sock_create_kern(&init_net, my_addr.ss_family,
│       │                    SOCK_STREAM, IPPROTO_TCP, &sock)
│       ├─ kernel_bind(sock, &my_addr, my_addr_len)
│       ├─ kernel_connect(sock, &peer_addr, peer_addr_len, 0)
│       │   // O_NONBLOCK: non-blocking, retried on EINPROGRESS
│       └─ Returns socket or error
│
├─ 3. Create meta socket (same process, different port)
│   └── Second kernel_connect() to peer_addr with different port
│       // Meta port = data port + 1 (convention)
│       // or both addresses specified in net-conf
│
├─ 4. Set socket options
│   ├─ tcp_sock_set_nodelay(sock->sk)    // disable Nagle
│   ├─ drbd_setbufsize(sock, sndbuf_size, rcvbuf_size)
│   │   → sock_setsockopt(SO_SNDBUF, SO_RCVBUF)
│   ├─ sock->sk->sk_sndtimeo = timeout * HZ
│   └─ sock->sk->sk_rcvtimeo = ping_timeout * HZ
│
├─ 5. Store sockets
│   └── tcp_transport->stream[DATA_STREAM]    = data_sock
│       tcp_transport->stream[CONTROL_STREAM] = meta_sock
│
└─ 6. Mark path as established
    └── path->established = true
```

**Who calls** vs **who listens:** The connect-int timer fires on both sides simultaneously. DRBD uses `RESOLVE_CONFLICTS` flag to determine which side acts as client vs server (the one with the lower IP address acts as server):

```bash
grep -n "RESOLVE_CONFLICTS\|dtt_create_listener\|dtt_try_connect\b" drbd/drbd_transport_tcp.c | head -15
```

---

## 7. `dtt_recv()` — Receiving Data

```bash
grep -n -A 60 "^static int dtt_recv\b" drbd/drbd_transport_tcp.c
```

```c
static int dtt_recv(struct drbd_transport *transport,
                     enum drbd_stream stream,
                     void **buf, size_t size, int flags)
{
    struct drbd_tcp_transport *tcp_transport =
        container_of(transport, struct drbd_tcp_transport, transport);
    struct socket *sock = tcp_transport->stream[stream];
    void *buffer = tcp_transport->rbuf[stream];
    int rv;

    // Use MSG_WAITALL to get exactly `size` bytes
    rv = drbd_recv_short(sock, buffer, size,
                          (flags & CALLER_BUFFER) ? MSG_WAITALL : MSG_WAITALL);

    if (rv == 0)
        return -ECONNRESET;   // peer closed connection
    if (rv < 0)
        return rv;            // error (ETIMEDOUT, ECONNRESET, etc.)
    if (rv != size)
        return -EIO;          // short read (shouldn't happen with MSG_WAITALL)

    *buf = buffer;
    return rv;
}
```

The `drbd_recv_short()` helper:
```bash
grep -n -A 30 "^static int drbd_recv_short\b" drbd/drbd_transport_tcp.c
```

```c
static int drbd_recv_short(struct socket *sock, void *buf, size_t size, int flags)
{
    struct kvec iov = { .iov_base = buf, .iov_len = size };
    struct msghdr msg = { .msg_flags = flags | MSG_NOSIGNAL };
    return kernel_recvmsg(sock, &msg, &iov, 1, size, msg.msg_flags);
}
```

---

## 8. `dtt_send_page()` — Zero-Copy Page Send

```bash
grep -n -A 50 "^static int dtt_send_page\b" drbd/drbd_transport_tcp.c
```

```c
static int dtt_send_page(struct drbd_transport *transport,
                          enum drbd_stream stream,
                          struct page *page,
                          int offset, size_t size,
                          unsigned msg_flags)
{
    struct socket *sock = tcp_transport->stream[stream];
    int len = size;
    int sent;

    // Use kernel_sendpage() = splice / sendfile path
    // Data sent directly from page cache → TCP buffer, no copy
    do {
        sent = kernel_sendpage(sock, page, offset, len,
                                msg_flags | MSG_NOSIGNAL);
        if (sent <= 0)
            break;
        offset += sent;
        len    -= sent;
    } while (len > 0);

    return size - len;  // bytes actually sent
}
```

Why `do-while`? `kernel_sendpage()` can return short — especially if TCP send buffer is full. DRBD loops until all bytes are sent.

---

## 9. `dtt_recv_pages()` — Zero-Copy Page Receive

For large data payloads (P_DATA, P_RS_DATA_REPLY), DRBD receives directly into page cache pages:

```bash
grep -n -A 60 "^static int dtt_recv_pages\b" drbd/drbd_transport_tcp.c
```

```c
static int dtt_recv_pages(struct drbd_transport *transport,
                           struct drbd_page_chain_head *chain,
                           size_t size)
{
    struct page *page;

    // Allocate pages from the DRBD page pool
    err = drbd_alloc_page_chain(&transport->class->rcu_head, chain, size);

    // Receive into each page
    page = chain->head;
    while (size > 0) {
        size_t chunk = min(size, (size_t)PAGE_SIZE);
        struct kvec iov = { .iov_base = page_address(page),
                            .iov_len  = chunk };
        struct msghdr msg = { .msg_flags = MSG_WAITALL | MSG_NOSIGNAL };

        err = kernel_recvmsg(sock, &msg, &iov, 1, chunk, msg.msg_flags);
        if (err != chunk)
            return -EIO;

        page = page->private;   // next page in chain
        size -= chunk;
    }
    return 0;
}
```

---

## 10. TCP Socket Timeout and Keepalive

```bash
grep -n "sk_rcvtimeo\|sk_sndtimeo\|ping.timeout\|ko.count\|TCP_KEEPIDLE\|TCP_KEEPINTVL" \
    drbd/drbd_transport_tcp.c | head -20
```

DRBD implements its own keepalive on top of TCP:
- `ping-timeout`: if no packet received in N seconds → disconnect
- `ko-count`: maximum consecutive timeouts before disconnect

The receiver thread tracks `connection->last_received`:
```bash
grep -n "last_received\|ping_timeout\|P_PING\b\|P_PING_ACK\b" drbd/drbd_receiver.c | head -15
```

```c
// In receiver main loop:
if (time_after(jiffies, connection->last_received + ping_timeout)) {
    // No data received: send P_PING
    drbd_send_ping(connection);
    connection->ko_count++;
    if (connection->ko_count > net_conf->ko_count)
        goto out_disconnect;
}
connection->last_received = jiffies;
```

---

## 11. Transport Error Handling — `dtt_stream_ok()`

```bash
grep -n -A 20 "^static bool dtt_stream_ok\b" drbd/drbd_transport_tcp.c
```

```c
static bool dtt_stream_ok(struct drbd_transport *transport,
                            enum drbd_stream stream)
{
    struct socket *sock = tcp_transport->stream[stream];
    if (!sock || sock->state != SS_CONNECTED)
        return false;
    return true;
}
```

Called from the receiver/sender before attempting I/O:
```bash
grep -n "stream_ok\b\|transport->ops->stream_ok\b" drbd/drbd_receiver.c drbd/drbd_sender.c | head -10
```

---

## 12. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Read `drbd_transport_tcp.c` completely
~1500 lines. For every `dtt_*` function, write its purpose and which `drbd_transport_ops` field it implements.

### Exercise 2 (40 min): Trace the connect retry loop
```bash
grep -n "dtt_connect\|schedule_timeout\|connect_int\|retry" drbd/drbd_transport_tcp.c | head -20
```
What happens when `kernel_connect()` returns `-ECONNREFUSED`? How long does DRBD wait before retrying? What is `connect-int` and where is it applied?

### Exercise 3 (35 min): Understand `RESOLVE_CONFLICTS`
```bash
grep -n "RESOLVE_CONFLICTS\b" drbd/drbd_transport_tcp.c drbd/drbd_transport.h | head -10
grep -n -A 20 "RESOLVE_CONFLICTS" drbd/drbd_transport_tcp.c | head -30
```
Which node acts as TCP server when `resolve-conflicts yes` is set? What determines the "lower" node?

### Exercise 4 (35 min): Trace the full receive path for a `P_DATA` packet
From `drbd_recv_header()` call in `drbd_receiver.c` down to `dtt_recv()` in `drbd_transport_tcp.c`:
1. What function in `drbd_receiver.c` calls `transport->ops->recv()`?
2. What buffer is used for the header vs the payload?
3. For the data payload: does DRBD use `dtt_recv()` or `dtt_recv_pages()`?

### Exercise 5 (40 min): Understand multi-path support
```bash
grep -n "struct drbd_path\b\|path_added\|path_removed\|paths_lock\b" \
    drbd/drbd_transport_tcp.c drbd/drbd_transport.h | head -20
grep -n -A 40 "^static int dtt_path_added\b" drbd/drbd_transport_tcp.c
```
How many paths can a single DRBD connection have? What happens when a path fails but another is available?

---

## Summary

The transport abstraction (`drbd_transport_ops` vtable) decouples the DRBD protocol from the network implementation. `drbd_transport_tcp.c` implements this vtable using Linux kernel sockets: `dtt_connect()` establishes two TCP connections (data + meta), `dtt_send_page()` uses `kernel_sendpage()` for zero-copy sends, `dtt_recv_pages()` receives large payloads directly into page chain allocations, and `dtt_stream_ok()` provides health checks. DRBD's own ping mechanism (`P_PING` / `P_PING_ACK`) supplements TCP's keepalive with application-level timeout detection.

**Next:** Day 17 — UUID system: generation, comparison, split-brain detection, and role in resync decisions.
