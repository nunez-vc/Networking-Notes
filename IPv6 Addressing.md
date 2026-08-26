<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# IPv6 Addressing

> **Core idea:** IPv6 addressing uses **128-bit addresses** and classless prefixes to identify interfaces and subnets. A normal IPv6 interface can hold multiple addresses at once—typically a **link-local address** plus one or more global or unique-local addresses—and relies on **Neighbor Discovery (NDP)** rather than ARP or broadcast.

---

## 1. What It Is

**IPv6 addressing** is the Layer 3 addressing system used by IPv6 to identify interfaces and organize them into routable prefixes. IPv6 addresses are 128 bits long, written in hexadecimal, and interpreted with a prefix length such as `/64`.

Example:

```text
2001:db8:10:20::25/64
```

```text
Address = 2001:db8:10:20::25
Prefix  = /64
```

> IPv6 has **no broadcast address**. It uses multicast and unicast mechanisms instead.

---

## 2. How It Works

## Address Format

An IPv6 address contains **128 bits**, written as eight 16-bit hexadecimal groups:

```text
2001:0db8:0010:0020:0000:0000:0000:0025
```

Each group contains four hexadecimal digits:

```text
0000 - ffff
```

A `/64` means:

```text
First 64 bits  = Prefix
Last 64 bits   = Interface identifier
```

For most normal LAN subnets:

```text
/64
```

is the standard prefix length and is required for normal SLAAC operation.

---

## Address Compression

IPv6 addresses can be shortened using two rules.

### Remove Leading Zeros

```text
2001:0db8:0010:0020:0000:0000:0000:0025

becomes

2001:db8:10:20:0:0:0:25
```

### Replace One Consecutive Run of Zero Groups with `::`

```text
2001:db8:10:20:0:0:0:25

becomes

2001:db8:10:20::25
```

`::` can appear **only once** in an IPv6 address because otherwise the number of omitted zero groups would be ambiguous.

---

## Core IPv6 Address Types

### Global Unicast Address (GUA)

Globally routable IPv6 unicast addresses are primarily allocated from:

```text
2000::/3
```

Example:

```text
2001:db8:10:20::25/64
```

`2001:db8::/32` is reserved for documentation and examples and should not be used as real production Internet addressing.

---

### Unique Local Address (ULA)

ULA space is:

```text
fc00::/7
```

Locally assigned ULAs commonly use:

```text
fd00::/8
```

Example:

```text
fd12:3456:789a:10::1/64
```

ULAs are suitable for private internal addressing where global Internet routing is not required.

> ULA is not the IPv6 equivalent of "IPv4 private + NAT required." IPv6 normally preserves end-to-end addressing rather than depending on address translation.

---

### Link-Local Address

Link-local addresses use:

```text
fe80::/10
```

Example:

```text
fe80::1
```

Every IPv6-enabled interface requires link-local functionality.

Link-local addresses are used for:

```text
Neighbor Discovery
Router Advertisements
Routing-protocol adjacencies
Next-hop communication
Other local-link control traffic
```

They are valid only on the local link and are **not routed** between links.

Because the same link-local address can exist on multiple interfaces, a link-local next hop often requires an interface context.

Example:

```text
fe80::2
```

alone does not identify which local link contains that neighbor.

---

### Multicast Address

IPv6 multicast uses:

```text
ff00::/8
```

Important examples:

```text
ff02::1
= All IPv6 nodes on the local link

ff02::2
= All IPv6 routers on the local link
```

IPv6 uses multicast instead of broadcast for many discovery and control functions.

---

### Anycast Address

IPv6 anycast uses a normal unicast address assigned to multiple interfaces/devices.

```text
Same IPv6 address
        ↓
Advertised from multiple locations
        ↓
Routing selects the nearest/best instance
```

There is no special general-purpose "anycast prefix" that identifies all anycast addresses.

---

## Important Special Addresses

| Address / Range | Purpose |
|---|---|
| `::/128` | Unspecified address |
| `::1/128` | Loopback |
| `2000::/3` | Global unicast allocation space |
| `2001:db8::/32` | Documentation |
| `fc00::/7` | Unique local |
| `fe80::/10` | Link-local |
| `ff00::/8` | Multicast |

---

## No Broadcast in IPv6

IPv6 does not use:

```text
255.255.255.255
```

or per-subnet broadcast addresses.

Functions that use broadcast in IPv4 are generally replaced by:

```text
Multicast
Neighbor Discovery
Router Advertisements
Other protocol-specific mechanisms
```

This reduces unnecessary processing by devices that are not interested in the traffic.

---

## Multiple Addresses per Interface

An IPv6 interface commonly has several addresses simultaneously:

```text
Link-local address
+
Global unicast and/or ULA
+
Temporary/privacy addresses when supported
+
Multicast group memberships
```

Example:

```text
Ethernet Interface

fe80::a1              Link-local
2001:db8:10:20::10    Global unicast
2001:db8:10:20:9c2... Temporary address
```

This is normal IPv6 behavior.

---

## Local vs Remote Destination

IPv6 uses the configured prefix to determine whether a destination is on-link.

For an on-link destination:

```text
Destination is local
        ↓
Neighbor Discovery resolves neighbor
        ↓
Send directly to destination
```

For an off-link destination:

```text
Destination is remote
        ↓
Use default router
        ↓
Resolve next-hop neighbor information
        ↓
Send to router
```

IPv6 does not use ARP.

```text
IPv4 → ARP
IPv6 → ICMPv6 Neighbor Discovery
```

---

## Neighbor Discovery (NDP)

NDP uses ICMPv6 and performs functions that IPv4 splits across ARP and other mechanisms.

Important NDP messages include:

| Message | Purpose |
|---|---|
| **Router Solicitation (RS)** | Host asks routers to send Router Advertisements |
| **Router Advertisement (RA)** | Router advertises prefixes and default-router information |
| **Neighbor Solicitation (NS)** | Resolve or verify a neighbor |
| **Neighbor Advertisement (NA)** | Respond with neighbor information |
| **Redirect** | Router suggests a better first hop |

For address resolution, IPv6 normally sends a Neighbor Solicitation to the target's **solicited-node multicast address**, rather than broadcasting to every host.

---

## Duplicate Address Detection (DAD)

Before using most newly configured IPv6 unicast addresses, a host performs **Duplicate Address Detection**.

Conceptually:

```text
Tentative IPv6 address
        ↓
Neighbor Solicitation
        ↓
Does another node claim the address?
       / \
     Yes  No
      |    |
Duplicate Address can be used
```

DAD prevents two interfaces on the same link from using the same unicast address.

---

## Router Advertisements and Default Gateway

IPv6 routers advertise themselves using **Router Advertisements**.

RA messages can provide:

```text
IPv6 prefix information
Prefix length
Default-router information
SLAAC parameters
Other Neighbor Discovery information
```

> **DHCPv6 does not provide the IPv6 default gateway.**  
> IPv6 hosts learn their default router through Router Advertisements.

---

## IPv6 Address Assignment Methods

### Static Addressing

The administrator manually assigns the address and prefix.

```text
2001:db8:10:20::10/64
```

Useful for infrastructure and systems requiring predictable addressing.

---

### SLAAC

**Stateless Address Autoconfiguration (SLAAC)** allows a host to construct its own IPv6 address from prefix information advertised by a router.

```text
Router Advertisement
        ↓
Advertised /64 prefix
        ↓
Host generates interface identifier
        ↓
Host forms IPv6 address
        ↓
DAD
        ↓
Address becomes usable
```

Modern hosts commonly use stable or temporary/privacy interface identifiers rather than assuming classic modified EUI-64 generation.

---

### DHCPv6

DHCPv6 can provide IPv6 addressing and/or configuration options.

```text
Stateful DHCPv6
= DHCPv6 assigns addresses and options

Stateless DHCPv6
= SLAAC creates address
  DHCPv6 supplies selected options
```

Router Advertisements indicate whether hosts should use SLAAC, DHCPv6, or a combination based on RA flags and prefix information.

Again:

```text
Default gateway
= Learned through RA

Not from DHCPv6
```

---

## Prefix Lengths

### /64 — Normal LAN Prefix

```text
2001:db8:10:20::/64
```

A `/64` is the normal subnet size for most IPv6 LANs and supports SLAAC.

---

### /127 — Point-to-Point

A `/127` contains two addresses and is commonly used on router-to-router point-to-point links when supported by the connected devices.

Example:

```text
R1 2001:db8:0:1::/127
R2 2001:db8:0:1::1/127
```

It is not suitable for normal SLAAC client LANs.

---

### /128 — Single Address

A `/128` represents exactly one IPv6 address.

Common uses:

```text
Loopbacks
Host routes
Specific policy/routing entries
```

Example:

```text
2001:db8:ffff::1/128
```

---

## Routing and Next-Hop Behavior

Routers make IPv6 forwarding decisions using **longest-prefix match**, just as with IPv4.

Example routes:

```text
2001:db8:10::/48
2001:db8:10:20::/64
::/0
```

For:

```text
2001:db8:10:20::25
```

the `/64` route wins because it is the most specific match.

A next hop may be expressed using a link-local address.

Example:

```text
Next hop = fe80::2
```

Because link-local addresses are scoped to a link, the outgoing interface must be known.

---

## 3. Why and When It Is Used

IPv6 addressing is appropriate wherever IPv6 connectivity is required, including:

```text
Enterprise LANs
Data centers
WANs
Cloud environments
Internet-facing services
Dual-stack migrations
IPv6-only networks
```

It solves major IPv4 limitations by providing:

- Vast address space
- End-to-end globally unique addressing
- Hierarchical prefix allocation
- Automatic address configuration
- Built-in link-local addressing
- Multicast-based neighbor discovery instead of broadcast

IPv6 addressing is unnecessary only where the environment does not use IPv6 at all.

In modern networks, **dual stack** is common during migration:

```text
IPv4 + IPv6
```

Both protocol families operate independently and require their own addressing, routing, policy, and troubleshooting.

---

## 4. Key Configuration, Parameters, or CLI

> **Platform:** Cisco IOS / IOS XE.

---

## Enable IPv6 Routing

On IOS/IOS XE routers and multilayer switches:

```cisco
ipv6 unicast-routing
```

This enables the device to forward IPv6 packets between interfaces.

---

## Static Global Unicast Address

```cisco
interface GigabitEthernet0/0
 ipv6 address 2001:db8:10:20::1/64
 no shutdown
```

IOS/IOS XE automatically creates a link-local address when IPv6 is enabled on the interface.

---

## Manually Configure a Link-Local Address

```cisco
interface GigabitEthernet0/0
 ipv6 address fe80::1 link-local
```

A manually configured link-local address can simplify routing-protocol and troubleshooting references.

---

## Enable IPv6 Without Configuring a Global Address

```cisco
interface GigabitEthernet0/0
 ipv6 enable
```

This enables IPv6 processing and creates a link-local address.

---

## SLAAC Client

```cisco
interface GigabitEthernet0/0
 ipv6 address autoconfig
```

The interface learns prefix/default-router information through Router Advertisements and forms an address through SLAAC.

---

## Point-to-Point /127

```cisco
interface GigabitEthernet0/1
 ipv6 address 2001:db8:0:1::/127
```

Peer:

```text
2001:db8:0:1::1/127
```

Verify platform support before standardizing `/127` across heterogeneous point-to-point links.

---

## Loopback /128

```cisco
interface Loopback0
 ipv6 address 2001:db8:ffff::1/128
```

---

## Static Route Using a Global Next Hop

```cisco
ipv6 route 2001:db8:20::/64 2001:db8:12::2
```

---

## Static Route Using a Link-Local Next Hop

When the next hop is link-local, specify the exit interface:

```cisco
ipv6 route 2001:db8:20::/64 GigabitEthernet0/0 fe80::2
```

---

## Verification

```cisco
show ipv6 interface brief
show ipv6 interface <interface>
show ipv6 neighbors
show ipv6 route
show ipv6 route <destination>
ping ipv6 <destination>
traceroute ipv6 <destination>
```

A practical troubleshooting sequence is:

```text
IPv6 enabled?
      ↓
Correct address/prefix?
      ↓
Link-local address present?
      ↓
DAD successful?
      ↓
Neighbor Discovery working?
      ↓
RA/default router present?
      ↓
Correct IPv6 route?
      ↓
Return route valid?
      ↓
Security policy allows ICMPv6 and application traffic?
```

---

## 5. Common Gotchas and Misconceptions

### IPv6 Does Not Need a Subnet Mask

**Misleading.**

IPv6 does not use dotted-decimal masks, but it absolutely uses **prefix lengths**.

```text
2001:db8:10:20::1/64
```

The `/64` determines the subnet boundary.

---

### Every IPv6 Address Starting with `fe80` Is Globally Routable

**Incorrect.**

`fe80::/10` is link-local.

```text
Valid on local link
Not routed between links
```

---

### ULA Is Internet-Routable Because It Is IPv6

**Incorrect.**

ULA space is intended for local/private use and is not globally routed on the public Internet.

---

### IPv6 Uses ARP

**Incorrect.**

IPv6 uses ICMPv6 Neighbor Discovery:

```text
IPv4 → ARP
IPv6 → NDP
```

---

### IPv6 Uses Broadcast

**Incorrect.**

IPv6 has no broadcast address.

It uses multicast and unicast mechanisms instead.

---

### DHCPv6 Supplies the Default Gateway

**Incorrect.**

The default router is learned through **Router Advertisements**.

This remains true even when DHCPv6 assigns the IPv6 address.

---

### SLAAC Always Uses EUI-64

**Incorrect.**

Modern operating systems commonly use stable or temporary/privacy interface identifiers.

Do not assume the interface identifier can be derived from the MAC address.

---

### `/64` Means Only 64 Hosts Are Available

**Incorrect.**

A `/64` leaves:

```text
64 interface-identifier bits
```

not 64 addresses.

The address space is:

```text
2^64 addresses
```

The large size exists primarily because of IPv6 architecture and autoconfiguration behavior, not because a LAN is expected to contain that many hosts.

---

### Link-Local Addresses Must Be Unique Across the Entire Network

**Incorrect.**

They only need to be unique on their local link.

The same link-local value can exist on different interfaces or links.

---

### Blocking ICMPv6 Is Safe

**Incorrect or Unsafe.**

ICMPv6 is fundamental to normal IPv6 operation, including:

```text
Neighbor Discovery
Router Discovery
Path MTU Discovery
Error signaling
```

Do not apply IPv4-style "block all ICMP" thinking to IPv6.

Use specific security policy rather than indiscriminate ICMPv6 blocking.

---

### IPv6 Eliminates the Need for Security Because NAT Is Not Required

**Incorrect.**

Globally unique addressing does not imply unrestricted reachability.

Use:

```text
Firewalls
ACLs
Segmentation
Host security
Routing policy
```

to enforce security.

NAT and security policy are separate functions.

---

## 6. Trade-Offs

### Best Practice

- Use `/64` for normal IPv6 LANs unless a specific design requires otherwise.
- Use hierarchical address allocation that supports summarization.
- Keep link-local addressing predictable on infrastructure links when that simplifies operations.
- Preserve required ICMPv6/NDP traffic in security policies.
- Treat IPv4 and IPv6 as separate protocol stacks during troubleshooting.
- Document GUA, ULA, loopback, infrastructure, and delegated prefix assignments clearly.

---

### Context-Dependent Trade-Off — SLAAC vs DHCPv6

**SLAAC**

```text
+ Simple host autoconfiguration
+ No stateful address server required
+ Native IPv6 behavior
- Less centralized address-assignment control
```

**Stateful DHCPv6**

```text
+ Centralized address assignment
+ Centralized lease/state visibility
- Requires DHCPv6 infrastructure
- Still depends on RA for default-router information
```

The correct choice depends on endpoint support, operational visibility, policy, and management requirements.

---

### Context-Dependent Trade-Off — GUA vs ULA

**GUA**

```text
+ Globally unique
+ Native end-to-end routing
+ No address overlap by design
- Requires globally allocated prefix planning
```

**ULA**

```text
+ Useful for internal-only addressing
+ Independent of public prefix changes
- Not globally Internet-routable
- Can create dual-addressing and source-selection complexity when combined with GUA
```

ULA should solve a specific design requirement, not be added automatically because IPv4 used RFC 1918 space.

---

### Context-Dependent Trade-Off — /64 vs /127

```text
/64
= Normal LAN / SLAAC subnet

/127
= Efficient point-to-point router link
```

Do not use `/127` on normal client LANs that depend on SLAAC.

---

### Incorrect or Unsafe

- Blocking required ICMPv6/NDP traffic.
- Treating IPv6 as "IPv4 with longer addresses."
- Assuming DHCPv6 provides the default gateway.
- Using non-/64 prefixes on SLAAC LANs without validating endpoint behavior and protocol requirements.
- Assuming NAT is required for IPv6 security.
- Deploying IPv6 without equivalent firewall, routing, monitoring, and operational controls used for IPv4.

---

## Quick Reference

```text
IPv6 Address
= 128 bits

Notation
= Hexadecimal
= 8 groups of 16 bits

Compression
= Remove leading zeros
= Use :: once for consecutive zero groups

GUA
= 2000::/3

ULA
= fc00::/7
= Common locally assigned space: fd00::/8

Link-Local
= fe80::/10

Multicast
= ff00::/8

Unspecified
= ::

Loopback
= ::1

Documentation
= 2001:db8::/32

Normal LAN Prefix
= /64

Point-to-Point
= /127

Single Address
= /128

IPv6 Neighbor Resolution
= ICMPv6 NDP

Default Gateway
= Learned through Router Advertisement

DHCPv6
= Does not provide default gateway

SLAAC
= Address autoconfiguration from RA prefix information

DAD
= Verifies address uniqueness on the local link

IPv6
= No broadcast
= No ARP
= Multiple addresses per interface are normal
```

</div>
