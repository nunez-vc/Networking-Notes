# Port Address Translation (PAT)

> **Core idea:** PAT allows many flows to share one or a small number of translated IPv4 addresses by maintaining a unique translation for each connection, usually with TCP or UDP port numbers.

---

## 1. What It Is

**Port Address Translation (PAT)** is a form of NAT that translates an IP address and, when necessary, a transport-layer identifier so multiple internal flows can share the same translated IPv4 address.

PAT is commonly called **NAT overload**. Dynamic PAT is primarily used for outbound many-to-one translation; **static PAT** uses a fixed IP/port mapping to publish a specific internal service.

---

## 2. How It Works

### Dynamic PAT

Assume two inside hosts open TCP connections to the same web server:

```text
10.10.10.10:50000 → 198.51.100.20:443
10.10.10.11:50000 → 198.51.100.20:443
```

Both flows can be translated to one outside address:

```text
203.0.113.10:50000 → 198.51.100.20:443
203.0.113.10:50001 → 198.51.100.20:443
```

The PAT device keeps a translation entry for each flow:

| Inside Local | Inside Global |
|---|---|
| `10.10.10.10:50000` | `203.0.113.10:50000` |
| `10.10.10.11:50000` | `203.0.113.10:50001` |

If the original source port can remain unique, the device may preserve it. If a collision exists, it selects another available translated port.

### Packet Processing

Outbound flow:

```text
Inside packet arrives
        ↓
Match PAT policy
        ↓
Find existing translation?
      /     \
    Yes      No
     |        |
 Use it   Allocate translated
     |     IP + identifier
      \       /
       ↓     ↓
Rewrite source address/port
        ↓
Forward packet
```

Return traffic is matched against the translation state:

```text
SRC 198.51.100.20:443
DST 203.0.113.10:50001
        ↓
PAT lookup
        ↓
DST translated to
10.10.10.11:50000
        ↓
Forward inside
```

The translation device updates affected checksums before forwarding.

For TCP and UDP, PAT normally distinguishes flows with port numbers. Other protocols may require protocol-specific identifiers or different NAT handling.

---

### Translation State

Dynamic PAT is stateful. The device tracks enough information to uniquely identify each translated flow.

Conceptually, the distinguishing information includes:

```text
Protocol
Original source IP
Original source identifier/port
Translated IP
Translated identifier/port
Destination information
```

Translations age out according to platform- and protocol-specific timers.

A single translated address therefore supports many simultaneous flows, but **not an unlimited number**. Capacity depends on available translation identifiers, connection state, platform limits, and traffic characteristics.

---

### PAT with Multiple Global Addresses

PAT can use:

```text
One interface/global address
```

or:

```text
A pool of global addresses
```

With a pool, the device can use additional translated IP addresses as translation capacity is consumed.

```text
Inside hosts
     ↓
PAT
     ↓
203.0.113.10
203.0.113.11
203.0.113.12
```

This increases available translation capacity compared with using one global address.

---

### Static PAT

**Static PAT**, often called **port forwarding** or **port redirection**, creates a fixed mapping between an outside IP/port and an internal IP/port.

Example:

```text
203.0.113.10:8443
        ↓
Static PAT
        ↓
10.10.20.10:443
```

This is different from dynamic PAT:

```text
Dynamic PAT
= Many outbound flows share translated addresses dynamically

Static PAT
= A specific translated IP/port maps to a specific internal service
```

Access-control policy still determines whether the traffic is permitted.

---

## 3. Why and When It Is Used

Dynamic PAT is appropriate when many IPv4 clients need outbound connectivity but only one or a small number of globally routable IPv4 addresses are available.

Typical uses:

- Enterprise or branch Internet access
- IPv4 public-address conservation
- Small-office edge routers and firewalls
- Large NAT gateways using PAT address pools

Static PAT is appropriate when an internal service must be reachable through a specific translated TCP or UDP port.

PAT is unnecessary when no address translation is required and end-to-end addressing is already routable.

IPv6 does not require PAT for IPv4-style public-address conservation.

---

## 4. Key Configuration, Parameters, or CLI

### Cisco IOS / IOS XE — Dynamic PAT Using an Interface Address

Identify the inside addresses to translate:

```cisco
ip access-list standard PAT_INSIDE
 permit 10.10.10.0 0.0.0.255
```

Define NAT roles:

```cisco
interface GigabitEthernet0/0
 ip nat inside

interface GigabitEthernet0/1
 ip nat outside
```

Enable PAT using the outside interface address:

```cisco
ip nat inside source list PAT_INSIDE interface GigabitEthernet0/1 overload
```

The key keyword is:

```text
overload
```

Without `overload`, the equivalent pool-based configuration performs dynamic one-to-one NAT rather than PAT.

---

### Cisco IOS / IOS XE — PAT Using an Address Pool

```cisco
ip nat pool PUBLIC 203.0.113.10 203.0.113.14 netmask 255.255.255.0

ip nat inside source list PAT_INSIDE pool PUBLIC overload
```

The translated addresses must be routable toward the NAT device from the outside network.

---

### Cisco IOS / IOS XE — Verification

```cisco
show ip nat translations
show ip nat statistics
```

A PAT translation includes protocol and port/identifier information:

```text
Inside Local
10.10.10.10:50000

Inside Global
203.0.113.10:50000
```

Troubleshoot in this order:

```text
Correct routing?
      ↓
Correct inside/outside interfaces?
      ↓
Does the ACL match the original inside source?
      ↓
Is overload configured?
      ↓
Is a PAT translation created?
      ↓
Is the translated address routable?
      ↓
Is return traffic reaching the same NAT state?
```

Clearing translations can disrupt active connections:

```cisco
clear ip nat translation *
```

Use this deliberately, not as a first troubleshooting step.

---

### Cisco Secure Firewall Threat Defense (FTD)

FTD uses a different NAT policy model from IOS/IOS XE.

Dynamic PAT can use:

- The **destination/egress interface IP**
- A configured **PAT pool**

When managed by Firewall Management Center, PAT is configured through NAT policy and deployed to the Threat Defense device.

Useful CLI verification:

```cisco
show running-config nat
show nat detail
show xlate detail
```

Do not apply IOS/IOS XE NAT syntax directly to FTD.

---

## 5. Common Gotchas and Misconceptions

### PAT Always Changes the Source Port

**Incorrect.** A PAT device can preserve the original port when the translated tuple remains unique. It changes the port when required to avoid a collision or according to platform behavior.

---

### PAT Means Unlimited Connections Behind One Address

**Incorrect.** PAT has finite translation capacity.

Constraints include:

```text
Available port/identifier space
Platform translation limits
Connection-table capacity
Protocol behavior
Timeouts
```

High-scale deployments may require a PAT pool rather than one translated address.

---

### PAT Is the Same as Dynamic NAT

**Incorrect.**

```text
Dynamic NAT
= One inside address receives one address from a pool

Dynamic PAT
= Many flows can share translated addresses using unique ports/identifiers
```

On IOS/IOS XE, the `overload` keyword is what enables PAT in the common inside-source configuration.

---

### The PAT ACL Matches the Translated Address

**Incorrect.** In the normal IOS/IOS XE inside-source PAT design, the ACL matches the **original inside source address before translation**.

```text
Correct:
permit 10.10.10.0/24

Not:
permit the translated public subnet
```

---

### PAT Is a Firewall

**Incorrect.** PAT performs translation. A firewall or ACL determines whether traffic is permitted.

Dynamic PAT often prevents unsolicited inbound traffic from matching an existing translation, but that behavior is not a substitute for explicit security policy.

---

### Return Traffic Can Use Any NAT Device

**Incorrect.** Dynamic PAT relies on translation state.

```text
Outbound → PAT device A creates translation
Return   → PAT device B has no matching state
```

Unless the platform synchronizes the required translation/session state, asymmetric forwarding can break the flow.

---

### All Protocols Behave Like TCP or UDP Through PAT

**Incorrect.** TCP and UDP provide ports that make multiplexing straightforward. Protocols without equivalent transport ports or applications that embed addresses/ports in payloads can require additional NAT handling or protocol-aware inspection.

---

## 6. Trade-Offs

### Best Practice

- Use dynamic PAT for general outbound IPv4 client access when public-address conservation is required.
- Keep the PAT match criteria specific to the intended inside prefixes.
- Monitor translation and connection utilization on high-scale edges.
- Ensure return traffic reaches the NAT device that owns the translation state.
- Use explicit firewall/ACL policy independently of PAT.

---

### Context-Dependent Trade-Off

**Interface PAT vs PAT Pool**

```text
Interface PAT
+ Simple
+ Minimal public addressing
- Lower translation capacity

PAT Pool
+ More translation capacity
+ Better fit for larger deployments
- Consumes more addresses
- Slightly more policy/configuration complexity
```

**Static PAT**

Useful for publishing selected services through a shared global IP address, but it creates intentional inbound reachability and must be paired with tightly scoped security policy.

---

### Incorrect or Unsafe

- Treating PAT as a replacement for firewall policy
- Assuming one translated IP provides unlimited scale
- Using overly broad PAT match rules without validating traffic scope
- Clearing translation state in production without considering active sessions
- Assuming PAT syntax, processing order, and state behavior are identical across IOS XE, ASA, and FTD

---

## Quick Reference

```text
PAT
= NAT + transport identifier translation

NAT Overload
= Common Cisco term for dynamic PAT

Dynamic PAT
= Many flows share one or more translated IP addresses

Static PAT
= Fixed IP/port mapping for a specific service

IOS / IOS XE keyword
= overload

Inside Local
= Original inside IP:port

Inside Global
= Translated IP:port

PAT state
= Required to reverse the translation for return traffic

PAT ≠ Firewall
PAT ≠ Unlimited scale
```

## CCNA Configuration

**IOS-XE — PAT Interface Roles**

| Command | Description |
|---|---|
| **Mark inside interface:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip nat inside` | Marks the interface as the NAT inside interface. |
| **Mark outside interface:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip nat outside` | Marks the interface as the NAT outside interface. |
| **Remove inside role:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no ip nat inside` | Removes the NAT inside designation. |
| **Remove outside role:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no ip nat outside` | Removes the NAT outside designation. |

**IOS-XE — PAT Using Outside Interface Address**

| Command | Description |
|---|---|
| **Create standard ACL:**<br>`(config)#ip access-list standard <acl-name>`<br>&nbsp;&nbsp;○ `(config-std-nacl)#permit <source-network> <wildcard-mask>` | Matches inside addresses eligible for PAT. |
| **Configure interface PAT:**<br>`(config)#ip nat inside source list <acl-name> interface <outside-interface> overload` | Overloads translations onto the outside interface address. |
| **Remove interface PAT:**<br>`(config)#no ip nat inside source list <acl-name> interface <outside-interface> overload` | Removes the interface-based PAT rule. |
| **Show PAT translations:**<br>`#show ip nat translations` | Displays active PAT address and port translations. |
| **Show PAT statistics:**<br>`#show ip nat statistics` | Displays translation counts, mappings, hits, and misses. |
| **Clear dynamic PAT entries:**<br>`#clear ip nat translation *` | Clears all dynamic PAT translations. |

**IOS-XE — PAT Using Address Pool**

| Command | Description |
|---|---|
| **Create PAT pool with netmask:**<br>`(config)#ip nat pool <pool-name> <start-ip> <end-ip> netmask <subnet-mask>` | Creates a PAT global-address pool using a subnet mask. |
| **Create PAT pool with prefix:**<br>`(config)#ip nat pool <pool-name> <start-ip> <end-ip> prefix-length <prefix-length>` | Creates a PAT global-address pool using prefix length. |
| **Configure pooled PAT:**<br>`(config)#ip nat inside source list <acl-name> pool <pool-name> overload` | Enables PAT using addresses from the configured pool. |
| **Remove pooled PAT:**<br>`(config)#no ip nat inside source list <acl-name> pool <pool-name> overload` | Removes the pooled PAT rule. |
| **Show pooled PAT:**<br>`#show ip nat translations` | Displays active PAT entries using pooled global addresses. |
| **Show pool utilization:**<br>`#show ip nat statistics` | Displays PAT pool mappings and translation statistics. |

## CCNP Configuration

**CCNP Enterprise — IOS-XE — PAT Translation Tuning**

| Command | Description |
|---|---|
| **Set translation timeout:**<br>`(config)#ip nat translation timeout <seconds>` | Sets inactivity timeout for dynamic NAT/PAT translations. |
| **Restore default timeout:**<br>`(config)#no ip nat translation timeout` | Restores the default dynamic translation timeout. |
| **Clear one address translation:**<br>`#clear ip nat translation <ip-address>` | Clears dynamic translations associated with the specified address. |
| **Clear all translations:**<br>`#clear ip nat translation *` | Clears all dynamic NAT/PAT translations. |
| **Show translation table:**<br>`#show ip nat translations` | Displays PAT protocol, addresses, and translated ports. |
| **Show translation statistics:**<br>`#show ip nat statistics` | Displays active translations, mappings, hits, and misses. |

**CCNP Security — ASA 9.x — Dynamic PAT Using Interface Address**

| Command | Description |
|---|---|
| **Create source object:**<br>`(config)#object network <source-object>`<br>&nbsp;&nbsp;○ `(config-network-object)#subnet <network> <subnet-mask>` | Creates the real source network object. |
| **Configure interface PAT:**<br>`(config)#object network <source-object>`<br>&nbsp;&nbsp;○ `(config-network-object)#nat (<real-interface>,<mapped-interface>) dynamic interface` | Dynamically PATs the source object to mapped interface address. |
| **Remove interface PAT:**<br>`(config)#object network <source-object>`<br>&nbsp;&nbsp;○ `(config-network-object)#no nat (<real-interface>,<mapped-interface>) dynamic interface` | Removes interface-address dynamic PAT from the object. |

**CCNP Security — ASA 9.x — Dynamic PAT Pool**

| Command | Description |
|---|---|
| **Create PAT pool object:**<br>`(config)#object network <pat-pool-object>`<br>&nbsp;&nbsp;○ `(config-network-object)#range <start-ip> <end-ip>` | Creates the mapped address range used as PAT pool. |
| **Create source object:**<br>`(config)#object network <source-object>`<br>&nbsp;&nbsp;○ `(config-network-object)#subnet <network> <subnet-mask>` | Creates the real source network object. |
| **Configure PAT pool:**<br>`(config)#object network <source-object>`<br>&nbsp;&nbsp;○ `(config-network-object)#nat (<real-interface>,<mapped-interface>) dynamic pat-pool <pat-pool-object>` | Dynamically PATs the source object through the PAT pool. |
| **Enable flat PAT pool:**<br>`(config)#object network <source-object>`<br>&nbsp;&nbsp;○ `(config-network-object)#nat (<real-interface>,<mapped-interface>) dynamic pat-pool <pat-pool-object> flat` | Uses flat port allocation across the PAT pool. |
| **Reserve privileged ports:**<br>`(config)#object network <source-object>`<br>&nbsp;&nbsp;○ `(config-network-object)#nat (<real-interface>,<mapped-interface>) dynamic pat-pool <pat-pool-object> flat include-reserve` | Includes reserved ports in flat PAT pool allocation. |

**CCNP Security — ASA 9.x — Static PAT / Port Redirection**

| Command | Description |
|---|---|
| **Create real server object:**<br>`(config)#object network <real-server-object>`<br>&nbsp;&nbsp;○ `(config-network-object)#host <real-server-ip>` | Creates the internal server network object. |
| **Create mapped server object:**<br>`(config)#object network <mapped-server-object>`<br>&nbsp;&nbsp;○ `(config-network-object)#host <mapped-ip>` | Creates the externally mapped server address object. |
| **Configure TCP static PAT:**<br>`(config)#object network <real-server-object>`<br>&nbsp;&nbsp;○ `(config-network-object)#nat (<real-interface>,<mapped-interface>) static <mapped-server-object> service tcp <real-port> <mapped-port>` | Maps one external TCP port to an internal TCP port. |
| **Configure UDP static PAT:**<br>`(config)#object network <real-server-object>`<br>&nbsp;&nbsp;○ `(config-network-object)#nat (<real-interface>,<mapped-interface>) static <mapped-server-object> service udp <real-port> <mapped-port>` | Maps one external UDP port to an internal UDP port. |

**CCNP Security — ASA 9.x — PAT Verification and Maintenance**

| Command | Description |
|---|---|
| **Show NAT rules:**<br>`#show nat detail` | Displays NAT/PAT rules, sections, priorities, and hit counters. |
| **Show active translations:**<br>`#show xlate` | Displays active NAT and PAT translation slots. |
| **Show detailed translations:**<br>`#show xlate detail` | Displays detailed PAT flags, ports, timers, and mappings. |
| **Show running NAT configuration:**<br>`#show running-config nat` | Displays configured ASA NAT and PAT rules. |
| **Show network objects:**<br>`#show running-config object` | Displays objects referenced by Auto NAT/PAT rules. |
| **Clear NAT counters:**<br>`#clear nat counters` | Resets NAT/PAT rule hit counters. |
| **Clear translation table:**<br>`#clear xlate` | Clears active NAT/PAT translation slots. |

**CCNP Security — Secure Firewall Threat Defense — PAT Verification**

| Command | Description |
|---|---|
| **Show deployed NAT configuration:**<br>`> show running-config nat` | Displays deployed Threat Defense NAT/PAT configuration. |
| **Show deployed objects:**<br>`> show running-config object` | Displays network objects referenced by deployed PAT rules. |
| **Show NAT rule details:**<br>`> show nat detail` | Displays NAT/PAT sections, priorities, and hit counters. |
| **Show active translations:**<br>`> show xlate detail` | Displays active PAT mappings, ports, flags, and timers. |
| **Clear NAT counters:**<br>`> clear nat counters` | Resets deployed NAT/PAT rule hit counters. |
| **Clear translations:**<br>`> clear xlate` | Clears active Threat Defense translation slots. |
