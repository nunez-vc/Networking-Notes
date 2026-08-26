<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# IPv4 Addressing

> **Core idea:** IPv4 addressing gives Layer 3 interfaces a **32-bit logical address** and a **prefix length/subnet mask** that defines which bits identify the subnet and which bits identify a host within that subnet. The address and prefix together determine local-subnet membership, routing boundaries, and the usable address range.

---

## 1. What It Is

**IPv4 addressing** is the Layer 3 addressing system used by IPv4 to identify interfaces and organize them into routable subnets. An IPv4 address is 32 bits long and is interpreted together with a subnet mask or prefix length.

Example:

```text
192.168.10.77/26
```

```text
Address       = 192.168.10.77
Prefix length = /26
Subnet mask   = 255.255.255.192
```

> In modern networks, IPv4 addressing is **classless**: the prefix length defines the subnet boundary. Do not infer the mask from old Class A/B/C rules.

---

## 2. How It Works

## Address Format

IPv4 addresses contain **32 bits**, written as four decimal octets:

```text
11000000.10101000.00001010.01001101
     192.     168.      10.       77
```

Each octet ranges from:

```text
0-255
```

The subnet mask divides the address into:

```text
Prefix / Subnet bits
        +
Host bits
```

Example:

```text
192.168.10.77/26

Prefix = 26 bits
Host   = 6 bits
```

A `/26` mask is:

```text
11111111.11111111.11111111.11000000
255.255.255.192
```

---

## Prefix Length and Subnet Mask

A valid IPv4 subnet mask consists of contiguous binary `1` bits followed by contiguous `0` bits.

```text
1 bits → Prefix/subnet portion
0 bits → Host portion
```

Common examples:

| Prefix | Subnet Mask | Addresses per Subnet |
|---:|---|---:|
| `/24` | `255.255.255.0` | 256 |
| `/26` | `255.255.255.192` | 64 |
| `/28` | `255.255.255.240` | 16 |
| `/30` | `255.255.255.252` | 4 |
| `/31` | `255.255.255.254` | 2 |
| `/32` | `255.255.255.255` | 1 |

For a normal subnet with `H` host bits:

```text
Total addresses = 2^H
Usable hosts    = 2^H - 2
```

The two normally reserved values are:

```text
All host bits 0 → Subnet ID
All host bits 1 → Subnet broadcast
```

> `/31` and `/32` are important exceptions to the normal `2^H - 2` usable-host rule.

---

## Subnet ID, Broadcast, and Usable Range

Example:

```text
Address: 192.168.10.77/26
Mask:    255.255.255.192
```

A `/26` creates blocks of 64 addresses:

```text
192.168.10.0   - 192.168.10.63
192.168.10.64  - 192.168.10.127
192.168.10.128 - 192.168.10.191
192.168.10.192 - 192.168.10.255
```

`192.168.10.77` belongs to:

```text
Subnet ID:       192.168.10.64
First usable:    192.168.10.65
Last usable:     192.168.10.126
Broadcast:       192.168.10.127
```

Memory:

```text
Lowest address  = Subnet ID
Highest address = Broadcast
Between them    = Normal host addresses
```

---

## /31 Point-to-Point Subnets

A `/31` contains two addresses and is specifically useful on point-to-point links.

Example:

```text
192.0.2.10/31
192.0.2.11/31
```

On a `/31` point-to-point subnet, both addresses can be used by the two endpoints; there is no traditional subnet-ID/broadcast reservation.

```text
R1 192.0.2.10/31 -------- 192.0.2.11/31 R2
```

This avoids wasting two addresses as a `/30` would.

> Confirm `/31` support on every device participating in the point-to-point link.

---

## /32 Addresses

A `/32` represents exactly one IPv4 address.

```text
10.10.10.10/32
```

Common uses include:

```text
Loopback interfaces
Host routes
Endpoint-specific policy/routing entries
```

A `/32` does not describe a normal multi-host subnet.

---

## Local vs Remote Destination Decision

A host uses its own prefix length to determine whether a destination is on the same subnet.

Conceptually:

```text
Own IP + mask
Destination IP + same mask
        ↓
Compare resulting subnet IDs
```

If the subnet IDs match:

```text
Destination is local
        ↓
Resolve destination MAC with ARP
        ↓
Send directly to destination
```

If the subnet IDs differ:

```text
Destination is remote
        ↓
Resolve default-gateway MAC with ARP
        ↓
Send frame to default gateway
```

Example:

```text
Host:    192.168.10.77/26
Gateway: 192.168.10.65
```

Destination:

```text
192.168.10.100
```

Both addresses are in:

```text
192.168.10.64/26
```

Therefore the destination is local.

Destination:

```text
192.168.10.200
```

belongs to:

```text
192.168.10.192/26
```

Therefore the host sends the packet to its default gateway.

> The IPv4 packet contains source and destination IP addresses, but **does not carry a subnet mask**. Hosts and routers learn prefix information from interface configuration, DHCP, static routes, and routing protocols.

---

## Connected Routes on a Router

When an IOS/IOS XE Layer 3 interface is operational with an IPv4 address, the router normally installs:

```text
Connected route → The attached subnet
Local route     → The router's exact interface address /32
```

Example interface:

```text
192.168.10.1/24
```

Conceptually:

```text
C 192.168.10.0/24
L 192.168.10.1/32
```

The connected route identifies the directly attached network; the local route represents traffic addressed to the router itself.

---

## Public and Private IPv4 Addresses

### RFC 1918 Private Space

| Range | Prefix |
|---|---:|
| `10.0.0.0 - 10.255.255.255` | `10.0.0.0/8` |
| `172.16.0.0 - 172.31.255.255` | `172.16.0.0/12` |
| `192.168.0.0 - 192.168.255.255` | `192.168.0.0/16` |

Private addresses can be routed normally **inside private networks**, but they are not intended to be globally routed across the public Internet.

IPv4 Internet access from private addressing normally uses NAT/PAT at the network boundary.

---

## Important Special IPv4 Ranges

| Address / Range | Purpose |
|---|---|
| `0.0.0.0` | Unspecified address in relevant host/protocol contexts |
| `0.0.0.0/0` | Default route prefix |
| `127.0.0.0/8` | Loopback |
| `169.254.0.0/16` | IPv4 link-local / automatic local addressing |
| `100.64.0.0/10` | Shared address space, commonly used for carrier-grade NAT |
| `224.0.0.0/4` | IPv4 multicast |
| `255.255.255.255` | Limited broadcast |

> `100.64.0.0/10` is **not RFC 1918 private space**.

---

## CIDR and VLSM

Modern IPv4 networks use **CIDR (Classless Inter-Domain Routing)** notation:

```text
10.20.30.0/24
```

The prefix length directly defines the subnet size without relying on address classes.

**VLSM (Variable-Length Subnet Masking)** allows different subnet sizes within an address block.

Example:

```text
10.10.0.0/24
   |
   +-- 10.10.0.0/25
   |
   +-- 10.10.0.128/26
   |
   +-- 10.10.0.192/27
   |
   +-- 10.10.0.224/28
```

VLSM allows address space to be allocated according to actual subnet requirements rather than forcing every subnet to use the same size.

A well-designed hierarchical address plan also makes route summarization easier.

---

## 3. Why and When It Is Used

IPv4 addressing provides the structure required to:

- Identify IPv4 interfaces.
- Group interfaces into Layer 3 subnets.
- Determine whether traffic is local or requires a router.
- Build routing tables using destination prefixes.
- Allocate address space efficiently with CIDR and VLSM.
- Separate public and private addressing domains.

IPv4 remains appropriate wherever IPv4 connectivity is required, including IPv4-only and dual-stack networks.

IPv4 addressing alone does not solve public-address exhaustion. Private addressing, NAT/PAT, and IPv6 are commonly used to address that constraint.

---

## 4. Key Configuration, Parameters, or CLI

> **Platform:** Cisco IOS / IOS XE.

---

### Router or Routed Interface

```cisco
interface GigabitEthernet0/0
 ip address 192.168.10.1 255.255.255.0
 no shutdown
```

---

### Catalyst IOS XE — Routed Physical Port

A switchport must first be converted to a Layer 3 routed port:

```cisco
interface GigabitEthernet1/0/1
 no switchport
 ip address 192.0.2.1 255.255.255.252
 no shutdown
```

---

### Catalyst IOS XE — SVI

```cisco
interface Vlan10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
```

For the switch to route traffic between Layer 3 interfaces/SVIs:

```cisco
ip routing
```

---

### /31 Point-to-Point Link

```cisco
interface GigabitEthernet0/1
 ip address 192.0.2.10 255.255.255.254
 no shutdown
```

Peer:

```text
192.0.2.11/31
```

---

### Loopback /32

```cisco
interface Loopback0
 ip address 10.255.255.1 255.255.255.255
```

---

### Verification

```cisco
show ip interface brief
show ip interface <interface>
show running-config interface <interface>
show ip route connected
show ip route <destination>
show ip arp
ping <destination>
traceroute <destination>
```

A practical troubleshooting sequence is:

```text
Correct IP address?
      ↓
Correct subnet mask/prefix?
      ↓
Interface Up/Up?
      ↓
Correct subnet ID?
      ↓
Local or remote destination?
      ↓
Correct ARP entry / default gateway?
      ↓
Correct route?
      ↓
Return path valid?
```

---

## 5. Common Gotchas and Misconceptions

### The First Octet Determines the Subnet Mask

**Incorrect.**

Modern IPv4 routing is classless.

```text
10.10.10.1/24
```

is a `/24` because the prefix says `/24`, not because `10.x.x.x` was historically considered Class A.

---

### Private IPv4 Addresses Cannot Be Routed

**Incorrect.**

RFC 1918 addresses can be routed normally inside private networks.

They are simply not intended to be globally routed on the public Internet.

---

### Every Subnet Uses `2^H - 2` Host Addresses

**Incorrect.**

That rule applies to normal broadcast-capable IPv4 subnets.

Important exceptions:

```text
/31 → Two usable point-to-point addresses
/32 → One individual address
```

---

### The Subnet Mask Is Carried Inside Every IPv4 Packet

**Incorrect.**

The IPv4 header contains source and destination addresses, not the subnet mask.

The sender and routers use locally known prefix information to make forwarding decisions.

---

### The Default Gateway Must Be the `.1` Address

**Incorrect.**

The gateway can use any valid host address in the subnet.

```text
192.168.10.1
192.168.10.254
192.168.10.65
```

can all be valid gateway addresses depending on the subnet and design.

---

### The Network and Broadcast Addresses Can Be Assigned to Hosts

For a normal IPv4 subnet, **incorrect**.

Example:

```text
192.168.10.0/24

192.168.10.0   = Subnet ID
192.168.10.255 = Broadcast
```

Neither is a normal host address.

The `/31` point-to-point exception behaves differently.

---

### Same First Three Octets Means Same Subnet

**Incorrect.**

The prefix length determines the subnet.

Example:

```text
192.168.10.65/26
192.168.10.130/26
```

They share the first three octets but are in different `/26` subnets.

---

### Overlapping Subnets Are Harmless

**Incorrect or Unsafe.**

Within the same routing context/VRF, overlapping connected address ranges create ambiguous addressing and forwarding behavior.

Plan subnets so they do not overlap unless the design intentionally isolates them in separate routing domains.

---

### `169.254.x.x` Is a Normal Private Enterprise Range

**Incorrect.**

`169.254.0.0/16` is IPv4 link-local space. A host using it unexpectedly often indicates that normal address assignment, such as DHCP, failed.

---

## 6. Trade-Offs

### Best Practice

- Use classless CIDR/VLSM addressing.
- Design hierarchical, non-overlapping address blocks that allow summarization.
- Size subnets for realistic host growth rather than maximum theoretical density.
- Keep infrastructure/static addressing organized separately from dynamic DHCP ranges.
- Use `/31` on supported point-to-point IPv4 links when address conservation is useful.
- Document prefixes, gateways, DHCP scopes, reservations, and routing boundaries.

---

### Context-Dependent Trade-Off — Subnet Size

**Larger subnet**

```text
+ More host addresses
+ Fewer routed boundaries
- Larger Layer 2 broadcast/failure domain
- More address consumption if underused
```

**Smaller subnet**

```text
+ Better address efficiency
+ Smaller Layer 2 domain
+ Clearer segmentation boundaries
- More subnets/routes to operate
```

The correct size depends on endpoint count, Layer 2 design, growth, routing architecture, and segmentation requirements.

---

### Context-Dependent Trade-Off — Private vs Public Addressing

**Private IPv4**

```text
+ Conserves public address space
+ Flexible internal allocation
- Internet access normally requires NAT/PAT
- Overlap can complicate mergers, VPNs, and partner connectivity
```

**Public IPv4**

```text
+ Globally unique and directly routable where policy permits
- Scarce
- Requires allocation and stronger exposure/security consideration
```

---

### Incorrect or Unsafe

- Designing new networks around classful default masks.
- Reusing overlapping prefixes in the same routing domain.
- Assigning normal subnet-ID or broadcast addresses to hosts.
- Using public addresses internally without owning or being assigned the prefix.
- Treating private addressing or NAT as a substitute for security policy.

---

## Quick Reference

```text
IPv4 Address
= 32 bits

Notation
= Dotted decimal
= Example: 192.168.10.77

Prefix
= Defines subnet bits
= Example: /26

Subnet Mask
= 255.255.255.192 for /26

Normal Subnet:
Lowest address  = Subnet ID
Highest address = Broadcast

Normal Host Count
= 2^H - 2

/31
= Two-address point-to-point subnet

/32
= One individual address

RFC 1918:
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16

Loopback
= 127.0.0.0/8

Link-Local
= 169.254.0.0/16

Shared CGN Space
= 100.64.0.0/10

Multicast
= 224.0.0.0/4

Limited Broadcast
= 255.255.255.255

Local Destination
= ARP for destination

Remote Destination
= ARP for default gateway

Modern IPv4
= Classless CIDR/VLSM
```

</div>
