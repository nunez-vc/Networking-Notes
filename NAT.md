# Network Address Translation (NAT)

> **Core idea:** NAT rewrites IP addressing at a network boundary and keeps enough translation state to reverse the change for return traffic. **PAT** extends this by translating transport-layer identifiers so many flows can share one or a small number of translated IP addresses.

---

## 1. What It Is

**Network Address Translation (NAT)** modifies source and/or destination IP addresses as packets cross a translating device such as a router or firewall. It sits in the forwarding path between routing and policy functions and maintains translation mappings so packets and return traffic can be associated with the correct endpoints.

NAT is **not routing** and is **not a security policy** by itself.

---

## 2. How It Works

### NAT Terminology

| Term | Meaning |
|---|---|
| **Inside Local** | Address of the inside host as represented on the inside network |
| **Inside Global** | Address that represents the inside host to the outside network |
| **Outside Global** | Address of the outside host as represented on the outside network |
| **Outside Local** | Address that represents the outside host to the inside network |

> **Local/global does not inherently mean private/public.** In the common Internet-access design, the inside local address is usually private and the inside global address is usually publicly routable.

---

### Source NAT

Source NAT changes the **source address** of a packet.

Example:

```text
Inside host:       10.10.10.10
Translated address: 203.0.113.10
Destination:        198.51.100.20
```

Outbound:

```text
Before NAT
SRC 10.10.10.10  → DST 198.51.100.20

After NAT
SRC 203.0.113.10 → DST 198.51.100.20
```

The NAT device records the mapping.

Return traffic:

```text
Before reverse translation
SRC 198.51.100.20 → DST 203.0.113.10

After reverse translation
SRC 198.51.100.20 → DST 10.10.10.10
```

The device also updates affected checksums before forwarding the packet.

---

### Destination NAT

Destination NAT changes the **destination address**, typically to publish an internal service through another address.

```text
Client
   |
   | DST 203.0.113.50
   v
NAT Device
   |
   | DST translated to 10.10.20.50
   v
Server
```

The reverse translation is applied to the return traffic so the connection remains consistent from the client's perspective.

---

### Static NAT

**Static NAT** creates a persistent one-to-one mapping.

```text
10.10.10.10 ↔ 203.0.113.10
```

Use it when a device requires a predictable translated address, particularly when inbound connections must target a stable mapping.

From a translation perspective, the mapping exists continuously. Routing and firewall/ACL policy still determine whether traffic is actually permitted.

---

### Dynamic NAT

**Dynamic NAT** temporarily maps an inside address to an available address from a NAT pool.

```text
Inside hosts
10.10.10.10
10.10.10.11
10.10.10.12
        |
        v
Public pool
203.0.113.10 - 203.0.113.20
```

The mapping is created as traffic initiates and is returned to the pool after the translation ages out.

Dynamic NAT without PAT is effectively **one translated IP per active inside host**, so the pool can be exhausted.

---

### Port Address Translation (PAT)

**PAT**, also called **NAT overload**, allows multiple connections to share the same translated IP address by tracking Layer 4 ports.

```text
10.10.10.10:50001 ─┐
10.10.10.11:50001 ─┼──> 203.0.113.5:unique-port
10.10.10.12:51020 ─┘
```

Example translation table:

| Inside Local | Inside Global |
|---|---|
| `10.10.10.10:50001` | `203.0.113.5:50001` |
| `10.10.10.11:50001` | `203.0.113.5:50002` |
| `10.10.10.12:51020` | `203.0.113.5:51020` |

PAT changes the source port when necessary to keep each translation unique.

> **Static NAT:** one-to-one, persistent  
> **Dynamic NAT:** one-to-one, temporary from a pool  
> **PAT:** many flows share one or more translated addresses

---

### Translation State

Dynamic NAT and PAT depend on a translation table.

Conceptually:

```text
Original flow
     ↓
Match NAT rule
     ↓
Create / use translation
     ↓
Rewrite address and/or port
     ↓
Forward packet
     ↓
Return packet matches translation
     ↓
Reverse translation
     ↓
Forward to original endpoint
```

If traffic returns through a different NAT device that does not share the required translation state, the return flow can fail. This makes **symmetric forwarding or state synchronization** important in stateful NAT designs.

---

## 3. Why and When It Is Used

NAT is commonly used for:

- **IPv4 Internet access:** translate RFC 1918 addresses to globally routable addresses.
- **IPv4 address conservation:** PAT allows many internal flows to share a small number of public addresses.
- **Service publishing:** map a stable external address to an internal server.
- **Overlapping address spaces:** translate one side when two networks use conflicting prefixes.
- **Address abstraction:** reduce the need to renumber internal systems when external addressing changes.

NAT is unnecessary when endpoints already have suitable end-to-end routable addressing and no address translation requirement exists.

IPv6 does not require NAT for IPv4-style public-address conservation. IPv6 NAT mechanisms exist for specific designs, but they should not be assumed to be the normal equivalent of IPv4 PAT.

---

## 4. Key Configuration, Parameters, or CLI

### Cisco IOS / IOS XE — Static Inside Source NAT

```cisco
interface GigabitEthernet0/0
 ip nat inside

interface GigabitEthernet0/1
 ip nat outside

ip nat inside source static 10.10.10.10 203.0.113.10
```

Mapping:

```text
Inside Local  10.10.10.10
      ↕
Inside Global 203.0.113.10
```

---

### Cisco IOS / IOS XE — Dynamic NAT Pool

Identify the inside addresses to translate:

```cisco
access-list 10 permit 10.10.10.0 0.0.0.255
```

Create the translated address pool:

```cisco
ip nat pool PUBLIC 203.0.113.10 203.0.113.20 netmask 255.255.255.0
```

Apply dynamic NAT:

```cisco
ip nat inside source list 10 pool PUBLIC
```

Interfaces must still be identified with:

```cisco
ip nat inside
ip nat outside
```

---

### Cisco IOS / IOS XE — PAT Using the Outside Interface

```cisco
access-list 10 permit 10.10.10.0 0.0.0.255

interface GigabitEthernet0/0
 ip nat inside

interface GigabitEthernet0/1
 ip nat outside

ip nat inside source list 10 interface GigabitEthernet0/1 overload
```

The `overload` keyword enables PAT.

---

### Cisco IOS / IOS XE — Verification

```cisco
show ip nat translations
show ip nat statistics
```

Troubleshoot in this order:

```text
1. Is routing correct?
        ↓
2. Are the correct interfaces marked inside/outside?
        ↓
3. Does the NAT match condition match the original traffic?
        ↓
4. Is a translation created?
        ↓
5. Is the translated address routable on the outside?
        ↓
6. Is ACL/firewall policy permitting the flow?
        ↓
7. Does return traffic traverse the same NAT state?
```

---

### Cisco Secure Firewall Threat Defense (FTD)

FTD NAT is normally configured through its management system rather than by manually entering IOS-style NAT commands.

FTD supports:

- **Auto NAT** — simpler object-based translation
- **Manual NAT** — supports more complex matching and source/destination translation in the same rule

NAT policy evaluation is divided into:

```text
Section 1 → Manual NAT before Auto NAT
Section 2 → Auto NAT
Section 3 → Manual NAT after Auto NAT
```

Useful FTD verification commands:

```cisco
show running-config nat
show nat detail
```

> IOS/IOS XE and FTD NAT syntax and rule-processing models are different. Do not transfer IOS NAT configuration syntax directly to FTD.

---

## 5. Common Gotchas and Misconceptions

### NAT Is a Firewall

**Incorrect.** NAT changes addressing. A firewall or ACL controls whether traffic is permitted.

A static mapping does not automatically mean inbound traffic is allowed.

---

### NAT Replaces Routing

**Incorrect.** The translated packet still requires valid routing.

```text
Translation works
        +
No route
        =
Traffic still fails
```

The translated address must also be reachable from the network where it is used.

---

### IOS Inside and Outside Are Reversed

On IOS/IOS XE, incorrect interface roles commonly prevent translations from being created.

Verify:

```cisco
show ip nat statistics
```

Confirm the expected interfaces appear under the correct inside/outside roles.

---

### The IOS NAT ACL Should Match the Translated Address

For normal IOS inside-source dynamic NAT/PAT, the ACL identifies the **original inside source addresses**, before translation.

Correct concept:

```text
Match: 10.10.10.0/24
Translate to: public address/pool
```

---

### Dynamic NAT and PAT Are the Same

They are not.

```text
Dynamic NAT
One active inside host → one pool address

PAT
Many flows → shared translated IP using ports
```

If `overload` is omitted from a configuration intended to use PAT, the device performs dynamic NAT instead and can quickly exhaust a small address pool.

---

### NAT Hides the Inside Network, Therefore It Is Security

NAT reduces direct address exposure, but it does not provide the policy enforcement, inspection, logging, or threat protection of a firewall.

> **Address translation and access control are separate functions.**

---

### Asymmetric Routing Can Break Stateful NAT

Return traffic must reach a device that knows the translation.

```text
Outbound → NAT-1 creates state
Return   → NAT-2 has no state
                 ↓
              Failure
```

HA designs must account for translation/session-state handling and platform-specific synchronization behavior.

---

### NAT Can Affect Protocols That Carry Address Information

Standard NAT rewrites packet headers, not arbitrary application payloads. Protocols that embed IP addresses or ports inside their payload may require protocol-aware inspection, an ALG, or NAT traversal support.

Do not assume every application is NAT-transparent.

---

### Active Sessions May Continue Using Existing Translations

On stateful firewalls, changing a NAT rule does not necessarily rebuild translations for already-established connections immediately.

Clearing translation/session state can force reevaluation but may disrupt active traffic.

---

## 6. Trade-Offs

### Best Practice

- Use **PAT** for general outbound IPv4 Internet access when public-address conservation is required.
- Use **static or destination NAT** only where a stable translated address is required.
- Keep NAT rules specific and avoid overlapping match criteria.
- Verify routing, NAT, and security policy independently.
- Design stateful NAT paths so return traffic reaches the correct translation state.
- Monitor translation-table and port utilization on high-scale NAT devices.

---

### Context-Dependent Trade-Off

**NAT for overlapping networks**

Useful during mergers, migrations, partner connections, and VPN integration, but it hides the underlying addressing conflict and increases troubleshooting complexity.

**Multiple layers of NAT**

Sometimes unavoidable in provider, cloud, or consumer environments, but double NAT complicates service publishing, VPNs, logging, troubleshooting, and application behavior.

**Dynamic NAT versus PAT**

Dynamic NAT preserves a unique translated IP per active host but consumes more addresses. PAT conserves addresses far more efficiently but depends on transport identifiers and translation-state capacity.

---

### Incorrect or Unsafe

- Treating NAT as a substitute for firewall policy
- Creating broad or overlapping NAT rules without validating rule order
- Publishing an internal service without explicitly validating the associated security policy
- Clearing translation state in production without considering active-session impact
- Assuming NAT configuration and processing order are identical across IOS XE, ASA, and FTD

---

## Quick Reference

```text
NAT
= Rewrites IP addresses

Source NAT
= Changes source address

Destination NAT
= Changes destination address

Static NAT
= Persistent one-to-one mapping

Dynamic NAT
= Temporary one-to-one mapping from a pool

PAT / NAT Overload
= Many flows share one or more translated IPs using ports

Inside Local
= Inside host address as represented internally

Inside Global
= Address representing the inside host externally

Outside Global
= Outside host address as represented externally

Outside Local
= Address representing the outside host internally

NAT Table
= Maintains dynamic translation state

NAT ≠ Routing
NAT ≠ Firewall
```

## CCNA Configuration

**CCNA 200-301 v2.0 — IOS-XE NAT Interface Roles**

| Command | Description |
|---|---|
| **Mark inside interface:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip nat inside` | Marks the interface as the NAT inside interface. |
| **Mark outside interface:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip nat outside` | Marks the interface as the NAT outside interface. |
| **Remove inside role:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no ip nat inside` | Removes the NAT inside designation. |
| **Remove outside role:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no ip nat outside` | Removes the NAT outside designation. |

**CCNA 200-301 v2.0 — IOS-XE Static NAT**

| Command | Description |
|---|---|
| **Create static inside mapping:**<br>`(config)#ip nat inside source static <inside-local-ip> <inside-global-ip>` | Creates a permanent one-to-one inside source translation. |
| **Remove static mapping:**<br>`(config)#no ip nat inside source static <inside-local-ip> <inside-global-ip>` | Removes the specified static inside source translation. |
| **Verify static translation:**<br>`#show ip nat translations` | Displays configured and active NAT translation entries. |
| **Verify NAT statistics:**<br>`#show ip nat statistics` | Displays NAT roles, mappings, hits, and misses. |

**CCNA 200-301 v2.0 — IOS-XE Dynamic NAT Pool**

| Command | Description |
|---|---|
| **Create NAT pool with netmask:**<br>`(config)#ip nat pool <pool-name> <start-ip> <end-ip> netmask <subnet-mask>` | Creates a dynamic NAT pool using a subnet mask. |
| **Create NAT pool with prefix:**<br>`(config)#ip nat pool <pool-name> <start-ip> <end-ip> prefix-length <prefix-length>` | Creates a dynamic NAT pool using prefix length. |
| **Create standard ACL:**<br>`(config)#ip access-list standard <acl-name>`<br>&nbsp;&nbsp;○ `(config-std-nacl)#permit <source-network> <wildcard-mask>` | Matches inside local addresses eligible for translation. |
| **Bind ACL to NAT pool:**<br>`(config)#ip nat inside source list <acl-name> pool <pool-name>` | Creates dynamic translations from the selected address pool. |
| **Verify dynamic translations:**<br>`#show ip nat translations` | Displays active dynamic NAT translation entries. |
| **Verify pool statistics:**<br>`#show ip nat statistics` | Displays pool allocation, hits, misses, and mappings. |
| **Clear dynamic translations:**<br>`#clear ip nat translation *` | Clears all dynamic NAT translation entries. |

**CCNA 200-301 v2.0 — IOS-XE PAT Using Interface Address**

| Command | Description |
|---|---|
| **Create PAT ACL:**<br>`(config)#ip access-list standard <acl-name>`<br>&nbsp;&nbsp;○ `(config-std-nacl)#permit <source-network> <wildcard-mask>` | Matches inside local addresses eligible for PAT. |
| **Configure interface PAT:**<br>`(config)#ip nat inside source list <acl-name> interface <outside-interface> overload` | Overloads translations onto the outside interface address. |
| **Verify PAT translations:**<br>`#show ip nat translations` | Displays PAT entries with translated transport identifiers. |
| **Verify PAT statistics:**<br>`#show ip nat statistics` | Displays NAT overload mapping and translation counters. |

**CCNA 200-301 v2.0 — IOS-XE PAT Using Address Pool**

| Command | Description |
|---|---|
| **Create PAT pool:**<br>`(config)#ip nat pool <pool-name> <start-ip> <end-ip> prefix-length <prefix-length>` | Creates the global address pool used for PAT. |
| **Configure pooled PAT:**<br>`(config)#ip nat inside source list <acl-name> pool <pool-name> overload` | Enables PAT using addresses from the NAT pool. |
| **Verify pooled PAT:**<br>`#show ip nat translations` | Displays active overloaded translations using pooled addresses. |
| **Clear pooled translations:**<br>`#clear ip nat translation *` | Clears dynamic PAT translations. |

## CCNP Configuration

**CCNP Enterprise — ENCOR 350-401 v1.2 — IOS-XE Outside Static NAT**

| Command | Description |
|---|---|
| **Create outside static mapping:**<br>`(config)#ip nat outside source static <outside-global-ip> <outside-local-ip>` | Creates a permanent outside source translation. |
| **Create mapping with route:**<br>`(config)#ip nat outside source static <outside-global-ip> <outside-local-ip> add-route` | Creates outside static NAT and installs a route. |
| **Remove outside mapping:**<br>`(config)#no ip nat outside source static <outside-global-ip> <outside-local-ip>` | Removes the specified outside static translation. |
| **Verify outside translation:**<br>`#show ip nat translations` | Displays outside global and outside local mappings. |

**CCNP Enterprise — ENCOR 350-401 v1.2 — IOS-XE NAT Timeout**

| Command | Description |
|---|---|
| **Set translation timeout:**<br>`(config)#ip nat translation timeout <seconds>` | Sets the inactivity timeout for dynamic NAT translations. |
| **Restore default timeout:**<br>`(config)#no ip nat translation timeout` | Restores the platform default NAT translation timeout. |
| **Verify translation state:**<br>`#show ip nat translations` | Displays active translations subject to configured timeouts. |
| **Verify NAT counters:**<br>`#show ip nat statistics` | Displays active, expired, hit, and miss statistics. |

**CCNP Enterprise — ENCOR 350-401 v1.2 — IOS-XE NAT Maintenance**

| Command | Description |
|---|---|
| **Clear all dynamic translations:**<br>`#clear ip nat translation *` | Clears all dynamic NAT translation entries. |
| **Clear address translation:**<br>`#clear ip nat translation <ip-address>` | Clears dynamic translations associated with the specified address. |
| **Show translation table:**<br>`#show ip nat translations` | Displays current static and dynamic translation entries. |
| **Show NAT statistics:**<br>`#show ip nat statistics` | Displays NAT mappings, interfaces, counters, and pool usage. |

**CCNP Security — SNCF 300-710 — Secure Firewall Threat Defense NAT Verification**

| Command | Description |
|---|---|
| **Show deployed NAT configuration:**<br>`> show running-config nat` | Displays deployed NAT policy syntax on Threat Defense. |
| **Show referenced network objects:**<br>`> show running-config object` | Displays objects referenced by deployed NAT rules. |
| **Show NAT rule details:**<br>`> show nat detail` | Displays NAT sections, priorities, types, and hit counters. |
| **Show translation table:**<br>`> show xlate` | Displays active Threat Defense translation slots. |
| **Show detailed translations:**<br>`> show xlate detail` | Displays detailed translation flags, timers, and mappings. |
| **Clear translations:**<br>`> clear xlate` | Clears active translations; FMC-managed policy remains unchanged. |
| **Show interface names:**<br>`> show nameif` | Displays physical interfaces and logical interface names. |
| **Show interface addressing:**<br>`> show interface ip brief` | Displays interface addressing and operational status. |

**CCNP Security — SNCF 300-710 — Secure Firewall Threat Defense NAT Trace**

| Command | Description |
|---|---|
| **Capture traced traffic:**<br>`> capture <capture-name> trace interface <nameif> match <protocol> <source> <destination> [eq <port>]` | Captures matching traffic with packet-processing trace data. |
| **Show capture list:**<br>`> show capture` | Displays configured packet captures and capture status. |
| **Trace captured packet:**<br>`> show capture <capture-name> packet-number <number> trace` | Displays processing phases including NAT decisions. |
