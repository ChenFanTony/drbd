# Day 15 — The Netlink Interface: How `drbdadm` Commands Reach the Kernel

> **Estimated study time: 3–4 hours**
> **Primary files:** `drbd/drbd_nl.c` (~5000 lines), `drbd/drbd_nla.c`, `drbd/linux/drbd_genl.h`, `drbd/linux/drbd_genl_api.h`

---

## 1. Why Generic Netlink?

DRBD uses **Generic Netlink (genl)** as its kernel↔userspace interface rather than ioctl or procfs because:
- Structured attribute-value pairs (no fixed struct layout to version)
- Bidirectional: kernel can send async events to userspace
- Multicast groups: multiple userspace listeners (drbdadm + monitoring tools)
- Policy enforcement: kernel validates attribute types and sizes automatically

```bash
grep -n "genl_register_family\|drbd_genl_family\|GENL_ID_GENERATE" drbd/drbd_nl.c drbd/drbd_genl_api.h | head -10
```

---

## 2. The Generic Netlink Family Registration

```bash
grep -n "struct genl_family drbd_genl_family\|drbd_genl_family\b" drbd/drbd_nl.c | head -5
grep -n -A 20 "struct genl_family drbd_genl_family" drbd/drbd_nl.c
```

```c
// drbd/drbd_nl.c
static struct genl_family drbd_genl_family = {
    .name       = "drbd",          // userspace opens this by name
    .version    = GENL_MAGIC_VERSION,
    .maxattr    = DRBD_NLA_CFG_REPLY + 1,
    .ops        = drbd_genl_ops,   // array of operation handlers
    .n_ops      = ARRAY_SIZE(drbd_genl_ops),
    .mcgrps     = drbd_mcgrps,     // multicast groups for async events
    .n_mcgrps   = ARRAY_SIZE(drbd_mcgrps),
    .module     = THIS_MODULE,
};
```

Registration in `drbd_init()`:
```bash
grep -n "genl_register_family\b" drbd/drbd_nl.c
```

---

## 3. The Operations Table

```bash
grep -n "drbd_genl_ops\b\|struct genl_ops\b" drbd/drbd_nl.c | head -5
grep -n -A 5 "drbd_genl_ops\[\]" drbd/drbd_nl.c | head -60
```

Each entry maps a command number to a handler function:

```c
static const struct genl_ops drbd_genl_ops[] = {
    {
        .cmd    = DRBD_ADM_NEW_RESOURCE,
        .doit   = drbd_adm_new_resource,
        .policy = drbd_tla_nl_policy,
    },
    {
        .cmd    = DRBD_ADM_DEL_RESOURCE,
        .doit   = drbd_adm_del_resource,
    },
    {
        .cmd    = DRBD_ADM_NEW_MINOR,
        .doit   = drbd_adm_new_minor,
    },
    {
        .cmd    = DRBD_ADM_DEL_MINOR,
        .doit   = drbd_adm_del_minor,
    },
    {
        .cmd    = DRBD_ADM_ATTACH,
        .doit   = drbd_adm_attach,    // Day 14
    },
    {
        .cmd    = DRBD_ADM_DETACH,
        .doit   = drbd_adm_detach,
    },
    {
        .cmd    = DRBD_ADM_CONNECT,
        .doit   = drbd_adm_connect,
    },
    {
        .cmd    = DRBD_ADM_DISCONNECT,
        .doit   = drbd_adm_disconnect,
    },
    {
        .cmd    = DRBD_ADM_PRIMARY,
        .doit   = drbd_adm_primary,
    },
    {
        .cmd    = DRBD_ADM_SECONDARY,
        .doit   = drbd_adm_secondary,
    },
    {
        .cmd    = DRBD_ADM_PAUSE_SYNC,
        .doit   = drbd_adm_pause_sync,
    },
    {
        .cmd    = DRBD_ADM_RESUME_SYNC,
        .doit   = drbd_adm_resume_sync,
    },
    {
        .cmd    = DRBD_ADM_START_OV,
        .doit   = drbd_adm_start_ov,
    },
    {
        .cmd    = DRBD_ADM_NEW_C_UUID,
        .doit   = drbd_adm_new_c_uuid,
    },
    {
        .cmd    = DRBD_ADM_GET_RESOURCES,
        .doit   = drbd_adm_get_resources,    // info dump
        .dumpit = drbd_adm_dump_resources,   // multi-message dump
    },
    {
        .cmd    = DRBD_ADM_GET_DEVICES,
        .dumpit = drbd_adm_dump_devices,
    },
    {
        .cmd    = DRBD_ADM_GET_CONNECTIONS,
        .dumpit = drbd_adm_dump_connections,
    },
    {
        .cmd    = DRBD_ADM_GET_PEER_DEVICES,
        .dumpit = drbd_adm_dump_peer_devices,
    },
    // ... more
};
```

```bash
grep -n "DRBD_ADM_\w\+\b" drbd/linux/drbd_genl.h | head -40
```

---

## 4. The Attribute Schema — `drbd_genl.h`

DRBD uses a macro-generated attribute schema. The schema is defined in:

```bash
grep -n "GENL_op\|GENL_struct\|GENL_mc_group" drbd/linux/drbd_genl.h | head -30
```

The magic is in `drbd_genl_api.h` — a set of macros that, depending on which macro is defined before inclusion, generate different things:

```bash
grep -n "#define GENL_op\|#define GENL_struct\|GENL_MAGIC_VERSION" drbd/linux/drbd_genl_api.h | head -20
```

**First expansion:** Generates `enum drbd_nlattr_*` constants (attribute IDs)
**Second expansion:** Generates `struct drbd_cfg_context` etc. (C structs)
**Third expansion:** Generates `drbd_tla_nl_policy[]` (attribute validation policy)

This single-source approach means the C structs in the kernel and the attribute IDs in userspace always stay in sync.

---

## 5. `drbd_adm_prepare()` — Common Preamble for All Handlers

Every `drbd_adm_*()` function starts with:

```bash
grep -n -A 80 "^static int drbd_adm_prepare\b\|^int drbd_adm_prepare\b" drbd/drbd_nl.c
```

```c
static int drbd_adm_prepare(struct drbd_config_context *adm_ctx,
                              struct sk_buff *skb,
                              struct genl_info *info,
                              unsigned int flags)
{
    // 1. Extract the cfg_context NLA (nested attribute)
    //    contains: resource name, volume number, peer node ID
    err = drbd_cfg_context_from_attrs(&adm_ctx->ctx, info);

    // 2. Look up the resource by name
    adm_ctx->resource = drbd_find_resource(adm_ctx->ctx.resource_name);
    // drbd_find_resource() walks drbd_resources global list

    // 3. If volume number provided: look up the device
    if (adm_ctx->ctx.has_volume) {
        adm_ctx->device = idr_find(&adm_ctx->resource->devices,
                                    adm_ctx->ctx.volume);
    }

    // 4. If peer node ID provided: look up the connection
    if (adm_ctx->ctx.has_peer_node_id) {
        adm_ctx->connection = drbd_connection_by_node_id(
            adm_ctx->resource, adm_ctx->ctx.peer_node_id);
    }

    // 5. Acquire adm_mutex to serialise admin commands
    if (flags & DRBD_ADM_NEED_RESOURCE)
        mutex_lock(&adm_ctx->resource->adm_mutex);

    // 6. Allocate reply skb
    adm_ctx->reply_skb = genlmsg_new(NLMSG_GOODSIZE, GFP_KERNEL);
    adm_ctx->reply_dh  = genlmsg_put_reply(adm_ctx->reply_skb, info,
                                             &drbd_genl_family, 0,
                                             info->genlhdr->cmd);
    return 0;
}
```

```bash
grep -n "drbd_adm_prepare\b" drbd/drbd_nl.c | head -20
```

---

## 6. `drbd_adm_finish()` — Sending the Reply

```bash
grep -n -A 40 "^static int drbd_adm_finish\b\|^int drbd_adm_finish\b" drbd/drbd_nl.c
```

```c
static int drbd_adm_finish(struct drbd_config_context *adm_ctx,
                             struct genl_info *info, int retcode)
{
    // 1. Put the return code into the reply
    struct drbd_genlmsghdr *dhdr = adm_ctx->reply_dh;
    dhdr->ret_code = retcode;

    // 2. Finalise and send the reply netlink message
    genlmsg_end(adm_ctx->reply_skb, adm_ctx->reply_dh);
    genlmsg_reply(adm_ctx->reply_skb, info);

    // 3. Release adm_mutex
    if (adm_ctx->resource)
        mutex_unlock(&adm_ctx->resource->adm_mutex);

    // 4. Drop reference to resource
    if (adm_ctx->resource)
        kref_put(&adm_ctx->resource->kref, drbd_destroy_resource);

    return 0;
}
```

---

## 7. Tracing `drbd_adm_primary()` — Full Path

```bash
grep -n -A 100 "^int drbd_adm_primary\b\|^static int drbd_adm_primary\b" drbd/drbd_nl.c
```

```
drbdadm primary r0
    → userspace: sends DRBD_ADM_PRIMARY genl message
        attributes: [DRBD_NLA_CFG_CONTEXT: {resource="r0"}]
                    [DRBD_NLA_SET_ROLE_PARMS: {assume_uptodate=0}]

kernel receives:
    → drbd_genl_ops[DRBD_ADM_PRIMARY].doit = drbd_adm_primary()
        │
        ├─ drbd_adm_prepare(adm_ctx, skb, info, DRBD_ADM_NEED_RESOURCE)
        │   → lock adm_mutex, find resource "r0", allocate reply_skb
        │
        ├─ set_role_parms_from_attrs(&parms, info)
        │   → parse DRBD_NLA_SET_ROLE_PARMS attributes
        │   → parms.assume_uptodate = nla_get_u8(...)
        │
        ├─ Validate: cannot promote if quorum lost
        │
        ├─ change_role(resource, R_PRIMARY, CS_VERBOSE, ...)
        │   → (Day 3 state machine)
        │   → role[NOW] = R_PRIMARY
        │   → __after_state_change(): resume I/O, send P_STATE to peers
        │
        ├─ drbd_adm_finish(adm_ctx, info, retcode)
        │   → sends reply with retcode = NO_ERROR or ERR_*
        └─ return 0
```

---

## 8. Tracing `drbd_adm_connect()` — Connecting to a Peer

```bash
grep -n -A 150 "^int drbd_adm_connect\b\|^static int drbd_adm_connect\b" drbd/drbd_nl.c
```

```
drbdadm connect r0
    → DRBD_ADM_CONNECT genl message
        attributes: resource name, peer node ID, net-conf (addresses, protocol, …)

drbd_adm_connect():
    │
    ├─ drbd_adm_prepare(): find resource + connection by peer_node_id
    │
    ├─ net_conf_from_attrs(&new_net_conf, info)
    │   → parse all net-conf NLAs:
    │     protocol (A/B/C), ping-timeout, connect-int,
    │     my_addr, peer_addr, cram-hmac-alg, shared-secret,
    │     sndbuf-size, rcvbuf-size, ko-count, ...
    │
    ├─ Validate: addresses parseable, protocol valid
    │
    ├─ drbd_adm_set_connection_net_conf(connection, new_net_conf)
    │   → rcu_assign_pointer(connection->net_conf, new_net_conf)
    │
    ├─ change_cstate(connection, C_UNCONNECTED, CS_VERBOSE)
    │   → __after_state_change(): drbd_thread_start(connection->receiver)
    │       → drbd_receiver() starts running
    │       → drbd_connect() attempts TCP
    │
    └─ drbd_adm_finish(): send reply
```

---

## 9. Async Event Notifications — `drbd_bcast_event()`

DRBD sends async events to userspace (not just replies to commands):

```bash
grep -n "drbd_bcast_event\b\|drbd_notify_\|DRBD_EVENTS_MCGRP\|notify_resource_state\b" \
    drbd/drbd_nl.c drbd/drbd_state.c | head -20
```

```c
// drbd/drbd_nl.c
void drbd_bcast_event(struct drbd_device *device,
                       const struct sib_info *sib)
{
    struct sk_buff *msg = genlmsg_new(NLMSG_GOODSIZE, GFP_NOIO);
    // Build event message with current device state
    // ...
    genlmsg_multicast(&drbd_genl_family, msg,
                       0, DRBD_EVENTS_MCGRP, GFP_NOIO);
    // All userspace processes subscribed to DRBD_EVENTS_MCGRP receive this
}
```

Events sent on state changes:
```bash
grep -n "notify_resource_state\|notify_device_state\|notify_connection_state\|notify_peer_device_state" \
    drbd/drbd_nl.c drbd/drbd_state.c | head -20
```

This is how `drbdadm events2` (the event monitor) works — it subscribes to the multicast group and prints every state change.

---

## 10. The Dump Operations — Reading State

For commands like `drbdadm status` or `drbdsetup show`, DRBD uses **dump** operations that send multiple netlink messages:

```bash
grep -n "drbd_adm_dump_resources\b\|drbd_adm_dump_devices\b\|drbd_adm_dump_connections\b" drbd/drbd_nl.c | head -10
grep -n -A 80 "^static int drbd_adm_dump_resources\b" drbd/drbd_nl.c
```

```c
static int drbd_adm_dump_resources(struct sk_buff *skb,
                                    struct netlink_callback *cb)
{
    struct drbd_resource *resource;
    int idx = 0, start_idx = cb->args[0];

    rcu_read_lock();
    list_for_each_entry_rcu(resource, &drbd_resources, resources) {
        if (idx++ < start_idx)
            continue;  // resume from where we left off (multi-message)

        // Build one netlink message per resource
        err = drbd_resource_state_to_skb(skb, resource, ...);
        if (err == -EMSGSIZE) {
            // skb full — return now, kernel will call us again
            cb->args[0] = idx - 1;  // resume cursor
            break;
        }
    }
    rcu_read_unlock();
    return skb->len;
}
```

The `cb->args[]` array is the resume cursor for multi-message dumps — critical for large clusters with many resources.

---

## 11. `drbd_nla.c` — Attribute Helpers

```bash
cat drbd/drbd_nla.c
# Short file (~100 lines) — read it completely
```

```bash
grep -n "drbd_nla_check_mandatory\|drbd_nla_parse_nested" drbd/drbd_nla.c
```

The key helper `drbd_nla_parse_nested()` wraps `nla_parse_nested()` with DRBD-specific mandatory-attribute checking:

```c
int drbd_nla_parse_nested(struct nlattr *tb[], int maxtype,
                            struct nlattr *nla,
                            const struct nla_policy *policy)
{
    int err = nla_parse_nested(tb, maxtype, nla, policy, NULL);
    if (err)
        return err;
    return drbd_nla_check_mandatory(maxtype, tb);
    // → verifies that all "mandatory" attributes are present
    // → DRBD marks some attributes as mandatory in drbd_genl.h
}
```

---

## 12. Hands-On Exercises (3–4 hours)

### Exercise 1 (50 min): Read `drbd_adm_new_resource()` + `drbd_adm_del_resource()`
```bash
grep -n -A 80 "^static int drbd_adm_new_resource\b\|^int drbd_adm_new_resource\b" drbd/drbd_nl.c
grep -n -A 60 "^static int drbd_adm_del_resource\b" drbd/drbd_nl.c
```
Trace: what objects are created/destroyed, in what order, what locks are taken?

### Exercise 2 (40 min): Trace `drbd_adm_disconnect()`
```bash
grep -n -A 80 "^static int drbd_adm_disconnect\b\|^int drbd_adm_disconnect\b" drbd/drbd_nl.c
```
What state change is triggered? How does it relate to `drbd_disconnect()` in `drbd_receiver.c`?

### Exercise 3 (40 min): Read the genl.h macro expansion
```bash
# Define the first expansion and include the file
grep -n "GENL_struct\|GENL_op\|GENL_mc_group" drbd/linux/drbd_genl.h | head -40
# Then:
grep -n -A 10 "net_conf\|disk_conf\|set_role" drbd/linux/drbd_genl.h | head -50
```
Identify: what NLA attributes make up `net_conf`? What are their types (u8, u32, string, flag)?

### Exercise 4 (35 min): Find and trace `drbd_adm_new_c_uuid()`
```bash
grep -n -A 60 "^static int drbd_adm_new_c_uuid\b\|^int drbd_adm_new_c_uuid\b" drbd/drbd_nl.c
```
When would an admin call `drbdadm new-current-uuid`? What does it do to UUIDs and the bitmap?

### Exercise 5 (35 min): Trace the `drbdadm status` path
```bash
grep -n "drbd_adm_dump_devices\b\|drbd_device_state_to_skb\b\|drbd_resource_state_to_skb\b" drbd/drbd_nl.c | head -10
grep -n -A 80 "^static int drbd_adm_dump_devices\b" drbd/drbd_nl.c
```
How does the kernel encode the current state of a device into a netlink message? What attributes are included?

---

## Summary

DRBD's kernel↔userspace interface is built on Generic Netlink. A family named "drbd" registers operation handlers (`drbd_genl_ops[]`) mapped to `DRBD_ADM_*` command codes. Every handler calls `drbd_adm_prepare()` (parse attributes, find objects, lock, allocate reply) and `drbd_adm_finish()` (send reply, unlock). Async state change notifications are multicast to the `DRBD_EVENTS_MCGRP` group. Dump operations use a cursor (`cb->args[]`) for multi-message responses. The attribute schema in `drbd_genl.h` is macro-expanded into struct definitions, attribute IDs, and validation policies from a single source.

**Next:** Day 16 — Transport abstraction: `drbd_transport.h`, `drbd_transport.c`, and the TCP implementation in `drbd_transport_tcp.c`.
