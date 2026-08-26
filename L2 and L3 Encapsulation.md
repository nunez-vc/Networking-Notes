<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# Layer 2 and Layer 3 Encapsulation

> **Core idea:** Layer 3 encapsulation places transport/application data inside an **IP packet** so it can be routed between networks. Layer 2 encapsulation then places that IP packet inside a **frame** for delivery across the current local link; at every routed hop, the Layer 2 frame is replaced while the Layer 3 packet continues toward the final destination.

---

## 1. What It Is

**Layer 3 encapsulation** adds an IPv4 or IPv6 header around upper-layer data to create a routable IP packet. **Layer 2 encapsulation** then wraps that packet in a link-layer frame—such as Ethernet or 802.11—so it can be delivered to the next device on the local link.

```text
Upper-Layer Data
      ↓
Layer 3: IP Packet
      ↓
Layer 2: Frame
      ↓
Physical Transmission
```

> **Layer 3 identifies the end-to-end IP endpoints. Layer 2 identifies the next-hop devices on the current link.**

---

## 2. How It Works

## Encapsulation at the Sender

Assume a host sends TCP data across Ethernet.

```text
Application Data
      ↓
TCP Header + Data
      ↓
IP Header + TCP Segment
      ↓
Ethernet Header + IP Packet + FCS
      ↓
Bits on the wire
```

The result is:

```text
+---------------- Ethernet Frame ----------------+
| Ethernet |        IPv4 / IPv6 Packet       |FCS|
| Header   |                                   |   |
|          | +---------- IP Packet ----------+ |   |
|          | | IP Header | TCP/UDP + Data    | |   |
|          | +-------------------------------+ |   |
+-----------------------------------------------+
```

Each layer adds only the information required for that layer's forwarding or delivery function.

---

## Layer 3 Encapsulation

An IP packet contains the logical addressing needed to move traffic across routed networks.

### Important IPv4 Fields

```text
Source IPv4 address
Destination IPv4 address
TTL
Protocol
Fragmentation information
Header checksum
Upper-layer payload
```

### Important IPv6 Fields

```text
Source IPv6 address
Destination IPv6 address
Hop Limit
Next Header
Payload Length
Upper-layer payload
```

The Layer 3 destination normally represents the **final IP endpoint**, not the next router.

Example:

```text
PC → Router → Router → Server

IPv4 Source      = 192.168.10.10
IPv4 Destination = 203.0.113.50
```

Those addresses normally remain the same across the routed path.

Exceptions include mechanisms that intentionally modify or replace Layer 3 information, such as:

```text
NAT
Tunneling
Some security/translation functions
```

At every routed hop:

```text
IPv4 TTL       → decremented
IPv6 Hop Limit → decremented
```

For IPv4, the header checksum is recalculated because the IPv4 header changed.

---

## Layer 2 Encapsulation

Ethernet encapsulation adds the information required to deliver the packet across one Layer 2 segment.

A normal Ethernet frame contains:

```text
Destination MAC
Source MAC
Optional 802.1Q VLAN tag
EtherType / Length
Payload
Frame Check Sequence (FCS)
```

For an IP payload:

```text
EtherType 0x0800 → IPv4
EtherType 0x86DD → IPv6
```

The Layer 2 destination MAC identifies the device that should receive the frame **on the current local link**.

---

## Local Destination vs Remote Destination

The sender first determines whether the destination IP is local or remote.

### Destination Is Local

```text
Host A
192.168.10.10/24

Host B
192.168.10.20/24
```

Host A sends directly to Host B:

```text
IP Source      = 192.168.10.10
IP Destination = 192.168.10.20

MAC Source      = Host A MAC
MAC Destination = Host B MAC
```

The sender learns the destination Layer 2 address using:

```text
IPv4 → ARP
IPv6 → Neighbor Discovery
```

---

### Destination Is Remote

```text
Host A
192.168.10.10/24

Server
203.0.113.50
```

Host A keeps the **server's IP address** as the Layer 3 destination but sends the Ethernet frame to its default gateway.

```text
IP Source      = 192.168.10.10
IP Destination = 203.0.113.50

MAC Source      = Host A MAC
MAC Destination = Default Gateway MAC
```

> **For remote traffic, the Layer 3 destination is the remote host, but the Layer 2 destination is the local next hop.**

---

## What a Switch Does

A normal Layer 2 switch forwards the Ethernet frame based on the destination MAC address.

```text
Frame arrives
     ↓
Learn source MAC
     ↓
Look up destination MAC
     ↓
Forward / flood as appropriate
```

For normal Layer 2 forwarding, the switch does **not** replace the source and destination IP addresses or MAC addresses simply because the frame crossed the switch.

However, the Layer 2 representation can change when required by the link—for example:

```text
Access port → Trunk
802.1Q tag added

Trunk → Access port
802.1Q tag removed
```

The switch also regenerates the transmitted frame/FCS on the outgoing interface.

---

## What a Router Does

A router terminates the incoming Layer 2 frame and creates a new Layer 2 frame for the next link.

Example:

```text
PC ---- R1 ---- R2 ---- Server
```

### Hop 1: PC → R1

```text
L2:
SRC MAC = PC
DST MAC = R1

L3:
SRC IP  = PC
DST IP  = Server
```

### Hop 2: R1 → R2

R1 removes the incoming Ethernet frame, performs a route lookup, and re-encapsulates the IP packet.

```text
L2:
SRC MAC = R1
DST MAC = R2

L3:
SRC IP  = PC
DST IP  = Server
```

### Hop 3: R2 → Server

```text
L2:
SRC MAC = R2
DST MAC = Server

L3:
SRC IP  = PC
DST IP  = Server
```

Memory:

```text
Layer 2 addresses
= Rewritten at routed boundaries

Layer 3 addresses
= Normally remain end to end
```

---

## Routed-Hop Processing Sequence

A router forwarding an Ethernet-encapsulated IP packet conceptually performs:

```text
1. Receive Ethernet frame
        ↓
2. Validate frame/FCS
        ↓
3. Remove Layer 2 encapsulation
        ↓
4. Inspect destination IP
        ↓
5. Perform longest-prefix route lookup
        ↓
6. Determine next hop / exit interface
        ↓
7. Resolve Layer 2 adjacency if needed
        ↓
8. Decrement TTL / Hop Limit
        ↓
9. Build new Layer 2 frame
        ↓
10. Transmit on next link
```

On Cisco IOS XE, the forwarding plane normally uses CEF/FIB and adjacency information to perform this operation efficiently.

---

## ARP and NDP Dependency

Before Ethernet encapsulation can be completed, the device needs the destination MAC for the local next hop.

### IPv4

```text
Routing lookup
     ↓
Next-hop IPv4 address
     ↓
ARP
     ↓
Next-hop MAC
     ↓
Build Ethernet frame
```

### IPv6

```text
Routing lookup
     ↓
Next-hop IPv6 address
     ↓
Neighbor Discovery
     ↓
Next-hop MAC
     ↓
Build Ethernet frame
```

This relationship is fundamental:

```text
Route
→ Determines next-hop IP

ARP / NDP
→ Determines next-hop Layer 2 address
```

---

## VLAN Encapsulation — 802.1Q

On an Ethernet trunk, an 802.1Q tag identifies the VLAN associated with the frame.

```text
+----------+-----------+----------+----------+------+
| DST MAC  | SRC MAC   | 802.1Q   | EtherType| Data |
+----------+-----------+----------+----------+------+
```

The 802.1Q tag contains a **12-bit VLAN ID**.

Typical behavior:

```text
Access port
= Frame normally untagged

Trunk port
= VLAN traffic normally tagged
```

The native VLAN is the common exception: native-VLAN traffic can be transmitted untagged on an 802.1Q trunk, depending on platform/configuration.

> The VLAN tag is Layer 2 information. It does not become part of the Layer 3 IP packet.

---

## MTU and Encapsulation Overhead

Every encapsulation layer adds header overhead.

For standard Ethernet, the common Layer 3 MTU is:

```text
1500 bytes
```

This means the Ethernet payload normally carries an IP packet up to 1500 bytes.

Additional encapsulation such as:

```text
802.1Q
GRE
IPsec
VXLAN
Other tunnels
```

adds overhead and can reduce the usable payload size before fragmentation or packet drops occur.

### IPv4

Depending on configuration and packet flags, oversized IPv4 packets may be fragmented or dropped.

### IPv6

Routers do **not** fragment IPv6 packets in transit. Path MTU Discovery and ICMPv6 Packet Too Big messages are therefore operationally important.

> MTU problems often appear as "small packets work, large packets fail."

---

## Nested Encapsulation

Tunneling can create multiple Layer 2 or Layer 3 headers.

Example:

```text
Original IP Packet
      ↓
GRE Header
      ↓
Outer IP Header
      ↓
Ethernet Frame
```

Conceptually:

```text
[Ethernet]
    [Outer IP]
        [GRE]
            [Inner IP]
                [TCP/UDP + Data]
```

The **inner packet** identifies the original endpoints, while the **outer header** transports that packet across the tunnel underlay.

This same principle appears in technologies such as GRE, IPsec tunnels, and overlays such as VXLAN.

---

## Decapsulation at the Destination

At the final endpoint:

```text
Receive frame
     ↓
Validate/remove Layer 2 header
     ↓
Process destination IP
     ↓
Remove Layer 3 header
     ↓
Deliver TCP/UDP payload
     ↓
Application receives data
```

Each layer processes only the information relevant to its function.

---

## 3. Why and When It Is Used

Layer 2 and Layer 3 encapsulation solve different forwarding problems:

```text
Layer 2
= How do I deliver this packet across the current local link?

Layer 3
= How do I deliver this data across multiple routed networks?
```

They are used whenever IP traffic traverses a link-layer technology such as:

```text
Ethernet
Wi-Fi
Point-to-point links
Tunnels
Overlay networks
```

The specific Layer 2 encapsulation may change between hops while the Layer 3 packet continues toward its destination.

Understanding this distinction is essential when troubleshooting:

```text
ARP / NDP
VLANs and trunks
MAC forwarding
Default gateways
Routing
MTU
Tunnels
NAT
Packet captures
Asymmetric paths
```

---

## 4. Key Configuration, Parameters, or CLI

> **Platform:** Cisco IOS / IOS XE.

The encapsulation process itself is automatic. The most useful CLI verifies the Layer 2 and Layer 3 information used to build and forward frames and packets.

---

### Layer 2 Verification

```cisco
show interfaces <interface>
show interfaces switchport
show interfaces trunk
show mac address-table
show vlan brief
```

Use these to verify:

```text
Interface state
Access/trunk mode
VLAN tagging
MAC learning
Frame/error counters
```

---

### Layer 3 Verification

```cisco
show ip interface brief
show ip route <destination>
show ip cef <destination> detail
show ip arp
```

For IPv6:

```cisco
show ipv6 interface brief
show ipv6 route <destination>
show ipv6 neighbors
```

Use these to verify:

```text
IP addressing
Route selection
Next hop
CEF adjacency
ARP/NDP resolution
```

---

### 802.1Q Trunk Configuration

```cisco
interface GigabitEthernet1/0/48
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
```

Verify:

```cisco
show interfaces trunk
```

---

### Routed Interface Example

```cisco
interface GigabitEthernet1/0/1
 no switchport
 ip address 192.0.2.1 255.255.255.252
 no shutdown
```

Verify:

```cisco
show ip interface brief
show ip route connected
```

---

## Practical Troubleshooting Sequence

```text
1. Is the physical interface operational?
        ↓
2. Is the frame in the correct VLAN?
        ↓
3. Is the destination MAC being learned/resolved?
        ↓
4. Is the IP address/prefix correct?
        ↓
5. Which route wins for the destination?
        ↓
6. What next hop / adjacency will be used?
        ↓
7. Is the packet being re-encapsulated toward the expected neighbor?
        ↓
8. Is MTU/encapsulation overhead causing drops?
```

A packet capture on both sides of a routed hop is often the clearest proof:

```text
Compare L2 headers
→ Should change

Compare L3 endpoints
→ Should normally remain the same

Check TTL/Hop Limit
→ Should decrease
```

---

## 5. Common Gotchas and Misconceptions

### The Destination MAC Is Always the Final Host's MAC

**Incorrect.**

For remote traffic, the destination MAC is the **next-hop router's MAC**, while the destination IP remains the final remote host.

```text
Remote packet:

DST IP  = Final server
DST MAC = Default gateway / next hop
```

---

### Routers Forward the Original Ethernet Frame End to End

**Incorrect.**

Routers remove the incoming Layer 2 frame and create a new frame for the next link.

```text
Old L2 header
    X
IP packet
    ↓
New L2 header
```

---

### Switches Normally Rewrite Source and Destination MAC Addresses

**Incorrect.**

A normal Layer 2 switch forwards frames using the existing source/destination MAC addresses.

A switch may add/remove VLAN tags or otherwise re-emit the frame for the outgoing medium, but it does not normally replace endpoint MAC addresses simply because it forwarded the frame.

---

### IP Addresses Change at Every Router

**Incorrect.**

Source and destination IP addresses normally remain unchanged across routers.

What changes at each routed hop is primarily:

```text
Layer 2 header
TTL / Hop Limit
IPv4 header checksum
```

NAT and tunneling are important exceptions.

---

### VLAN Tags Are Part of the IP Header

**Incorrect.**

802.1Q tagging is Layer 2 encapsulation.

```text
Ethernet
  ↓
802.1Q tag
  ↓
IP packet
```

The IP packet itself does not contain the VLAN ID.

---

### ARP or NDP Determines the Route

**Incorrect.**

Routing determines the next-hop Layer 3 destination first.

ARP/NDP then resolves that next hop into the Layer 2 information required to transmit the frame.

```text
Routing
→ Next-hop IP

ARP/NDP
→ Next-hop MAC
```

---

### FCS Protects the Entire End-to-End Packet Path

**Incorrect.**

Ethernet FCS protects the frame only across its current Layer 2 transmission.

At a routed boundary, that frame terminates and a new frame/FCS is created for the next link.

---

### Encapsulation Overhead Does Not Affect MTU

**Incorrect.**

Additional headers consume bytes.

Tunnels and security encapsulation can therefore cause:

```text
Fragmentation
PMTUD dependence
Packet drops
MSS adjustments
Application failures
```

Always account for the full encapsulation stack.

---

## 6. Trade-Offs

### Best Practice

- Troubleshoot Layer 2 and Layer 3 separately before assuming an application problem.
- For remote traffic, verify both the **route** and the **next-hop ARP/NDP adjacency**.
- Use packet captures to compare L2 and L3 headers across a routed hop when behavior is unclear.
- Account for encapsulation overhead when designing MTU across trunks, tunnels, VPNs, and overlays.
- Preserve required ICMP/ICMPv6 signaling so Path MTU Discovery can function.

---

### Context-Dependent Trade-Off — Larger Encapsulation Stacks

Encapsulation enables tunnels, overlays, encryption, and segmentation, but each additional header consumes MTU and adds processing complexity.

```text
More encapsulation
+ Enables overlays / tunnels / security
- More overhead
- More MTU planning
- More troubleshooting layers
```

The design is justified when the overlay, isolation, or security benefit outweighs that operational cost.

---

### Context-Dependent Trade-Off — Layer 2 Extension vs Layer 3 Boundary

```text
Extended Layer 2
+ Preserves one VLAN/broadcast domain
+ Useful for specific mobility/application requirements
- Larger failure/broadcast domain
- More STP/L2 dependency

Layer 3 boundary
+ Smaller failure domains
+ Clear routing and convergence behavior
+ Better scaling
- Requires routed addressing between segments
```

Choose based on application requirements and failure-domain design, not merely convenience.

---

### Incorrect or Unsafe

- Treating MAC addresses as end-to-end routed identifiers.
- Assuming a valid route guarantees that the next-hop Layer 2 adjacency exists.
- Ignoring MTU when adding GRE, IPsec, VXLAN, or other encapsulations.
- Blocking ICMP/ICMPv6 messages required for Path MTU Discovery without providing an alternative MTU strategy.
- Troubleshooting only Layer 3 when VLAN/tagging or Layer 2 adjacency is still unverified.

---

## Quick Reference

```text
Layer 3 Encapsulation
= IP header + upper-layer payload

Layer 2 Encapsulation
= Frame header/trailer + Layer 3 packet

Ethernet L2
= Source MAC
= Destination MAC
= Optional 802.1Q tag
= EtherType
= FCS

IPv4 EtherType
= 0x0800

IPv6 EtherType
= 0x86DD

Local Destination
= L2 destination is final host

Remote Destination
= L2 destination is next-hop router
= L3 destination remains final host

At a Switch
= Frame normally keeps endpoint MAC/IP addresses
= VLAN tag may be added/removed

At a Router
= Incoming L2 frame removed
= Route lookup performed
= TTL/Hop Limit decremented
= New L2 frame created

Routing
= Determines next-hop IP

ARP / NDP
= Determines next-hop MAC

802.1Q
= Layer 2 VLAN tagging

Typical Ethernet L3 MTU
= 1500 bytes

Tunnel Encapsulation
= Adds outer headers
= Reduces usable MTU

Core Rule
= L2 is hop-by-hop
= L3 is normally end-to-end
```

</div>
