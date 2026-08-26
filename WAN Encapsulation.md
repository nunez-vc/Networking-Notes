<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# WAN Encapsulation

> **Core idea:** WAN encapsulation is the **link-layer framing used to carry Layer 3 packets across a WAN link or service attachment**. The encapsulation used on a customer edge depends on the circuit or provider handoff—commonly Ethernet today, with PPP or Cisco HDLC still relevant on point-to-point serial links.

---

## 1. What It Is

**WAN encapsulation** defines how a Layer 3 packet is framed for transmission across a WAN-facing link. It is a **per-link function**: routers remove the incoming Layer 2 encapsulation, route the IP packet, and build the Layer 2 framing required by the next link.

```text
IP Packet
   ↓
WAN Link Encapsulation
   ↓
Physical / Provider Transport
```

> There is no single protocol called "WAN encapsulation." The encapsulation is determined by the WAN technology and provider/customer handoff.

---

## 2. How It Works

## Hop-by-Hop Encapsulation

Assume an IP packet crosses a LAN, a WAN circuit, and another LAN:

```text
Host A ──Ethernet── R1 ══ WAN ══ R2 ──Ethernet── Host B
```

The Layer 3 packet normally keeps the same source and destination IP addresses end to end:

```text
SRC IP = Host A
DST IP = Host B
```

The Layer 2 encapsulation changes at each routed boundary:

```text
Host A → R1
= Ethernet frame

R1 → R2
= WAN-specific framing

R2 → Host B
= New Ethernet frame
```

Router processing is conceptually:

```text
Receive frame
    ↓
Remove Layer 2 encapsulation
    ↓
Inspect destination IP
    ↓
Route lookup
    ↓
Select next hop / exit interface
    ↓
Apply required outgoing encapsulation
    ↓
Transmit
```

---

# Common WAN Encapsulation Models

## Ethernet WAN

Modern WAN services commonly present an **Ethernet handoff** to the customer router or firewall.

Examples include:

```text
Dedicated Internet Access (DIA)
Metro Ethernet
Carrier Ethernet
Many MPLS L3VPN customer handoffs
Cloud/interconnect services
```

The customer edge therefore uses normal Ethernet framing:

```text
Destination MAC
Source MAC
Optional 802.1Q tag
EtherType
IP packet
FCS
```

The provider may carry the traffic internally using technologies such as:

```text
MPLS
EVPN
Segment Routing
Provider Ethernet
Other transport mechanisms
```

but those internal mechanisms do not automatically change the customer's local Ethernet encapsulation.

---

## 802.1Q on WAN Handoffs

A provider may deliver one or more logical WAN services over an 802.1Q-tagged Ethernet interface.

Example:

```text
Physical WAN Interface
        |
        +-- VLAN 100 → Internet
        |
        +-- VLAN 200 → Private WAN
```

On IOS/IOS XE this is commonly implemented with subinterfaces:

```cisco
interface GigabitEthernet0/0.100
 encapsulation dot1Q 100
 ip address 192.0.2.2 255.255.255.252
```

The VLAN tag is Layer 2 information and is removed/recreated as required by each local Ethernet segment.

> The provider-assigned VLAN ID and tagging model must match exactly. A correct IP configuration cannot compensate for an incorrect 802.1Q handoff.

---

## PPP

**Point-to-Point Protocol (PPP)** is a standards-based Layer 2 protocol used on point-to-point links and as the logical session layer in technologies such as PPPoE.

PPP provides:

```text
Link establishment
Encapsulation of Layer 3 protocols
Optional peer authentication
Link-quality negotiation
Layer 3 protocol configuration
```

A simplified PPP frame is:

```text
+------+---------+---------+----------+---------+------+
| Flag | Address | Control | Protocol | Payload | FCS  |
+------+---------+---------+----------+---------+------+
```

The **Protocol** field identifies the payload carried by PPP.

---

### PPP Control Sequence

PPP brings the link into service in stages:

```text
Physical link available
        ↓
LCP negotiation
        ↓
Optional authentication
        ↓
NCP negotiation
        ↓
Layer 3 traffic can pass
```

### LCP — Link Control Protocol

LCP establishes and maintains the PPP link.

It negotiates or validates link parameters before Layer 3 traffic is allowed.

---

### Authentication

PPP can optionally authenticate the peer.

Common mechanisms:

```text
PAP
CHAP
```

**PAP**

```text
Username/password sent through the PPP authentication exchange
No challenge-response protection
```

**CHAP**

```text
Challenge-response authentication
Password itself is not transmitted as plaintext in the CHAP exchange
```

> PAP and CHAP are legacy PPP authentication mechanisms. They should not be treated as substitutes for modern encrypted WAN security such as IPsec.

---

### NCP — Network Control Protocol

After LCP and any required authentication succeed, PPP uses an NCP for the Layer 3 protocol being carried.

Examples:

```text
IPCP   → IPv4
IPv6CP → IPv6
```

If LCP succeeds but the relevant NCP fails, the PPP link may exist while the expected Layer 3 protocol is not operational.

---

## Cisco HDLC

Cisco IOS/IOS XE serial interfaces can use **Cisco HDLC**, Cisco's proprietary variation of HDLC.

It is a simple point-to-point encapsulation with low operational complexity.

```text
IP packet
   ↓
Cisco HDLC framing
   ↓
Serial link
```

On Cisco serial interfaces, Cisco HDLC has traditionally been the default encapsulation.

Important interoperability point:

```text
Cisco HDLC
≠ Generic standards-based HDLC interoperability
```

If the remote side is not using compatible Cisco HDLC framing, use a mutually supported encapsulation such as PPP.

---

## PPPoE

**PPP over Ethernet (PPPoE)** carries a PPP session across Ethernet.

Conceptually:

```text
IP Packet
   ↓
PPP
   ↓
PPPoE
   ↓
Ethernet
```

It is still encountered on broadband and provider access services.

Because PPPoE adds additional headers over Ethernet, the common IP MTU is often:

```text
1492 bytes
```

instead of the normal Ethernet IP MTU of:

```text
1500 bytes
```

Exact MTU requirements depend on the provider and platform.

---

# What Is Not the Same as WAN Layer 2 Encapsulation

## GRE, IPsec, VXLAN

These technologies also encapsulate packets, but they are **tunnel/overlay encapsulations**, not simply the Layer 2 framing of a WAN circuit.

Example:

```text
Original IP Packet
      ↓
GRE / IPsec / Other Tunnel
      ↓
Outer IP Packet
      ↓
Ethernet / PPP / Other WAN Link Encapsulation
```

The underlying WAN link still requires its own Layer 2 framing.

---

## MPLS

MPLS adds labels between the Layer 2 and Layer 3 information inside the provider network.

Conceptually:

```text
Ethernet
   ↓
MPLS Label Stack
   ↓
IP Packet
```

For many enterprise MPLS L3VPN services, the customer edge sees only an Ethernet or other normal provider handoff. The provider's MPLS label operations occur inside the service-provider network.

> Do not assume that buying an "MPLS WAN" means the customer router is directly configuring MPLS encapsulation.

---

# Encapsulation Agreement

Both ends of a direct WAN link must use compatible framing.

Example:

```text
R1 serial encapsulation = PPP
R2 serial encapsulation = Cisco HDLC
```

Result:

```text
Layer 1 may be Up
Layer 2 cannot operate correctly
```

A common symptom on serial links is:

```text
Interface Up
Line protocol Down
```

The exact cause must still be verified; encapsulation mismatch is only one possibility.

---

# MTU and Encapsulation Overhead

Each encapsulation adds bytes.

```text
Original IP packet
      +
WAN / VLAN / PPPoE / Tunnel headers
      =
Larger transmitted frame
```

Examples:

```text
Ethernet IP MTU commonly = 1500 bytes
PPPoE IP MTU commonly    = 1492 bytes
```

Additional overlays such as GRE or IPsec add more overhead.

Operational symptoms of an MTU problem can include:

```text
Small pings succeed
Large pings fail
TCP sessions establish but transfers stall
Some applications work while others fail
```

MTU planning must include the **entire encapsulation stack**, not only the customer-facing WAN frame.

---

## Error Detection vs Recovery

WAN Layer 2 encapsulations commonly include a frame-check mechanism such as an FCS.

```text
FCS detects corruption
```

but Layer 2 error detection does not necessarily provide retransmission.

For example:

```text
PPP / HDLC
= Detect corrupted frames
= Do not provide TCP-style end-to-end recovery
```

Upper-layer protocols or applications handle any required retransmission.

---

## 3. Why and When It Is Used

WAN encapsulation is required whenever Layer 3 traffic must cross a WAN link because the IP packet must be represented in the framing expected by that link.

The correct encapsulation is determined by the service:

```text
Ethernet provider handoff
→ Ethernet / optional 802.1Q

Point-to-point serial circuit
→ PPP or compatible HDLC

PPPoE access service
→ PPP over Ethernet

Overlay VPN across Internet
→ Tunnel encapsulation over the underlying Ethernet/WAN service
```

Use the encapsulation specified by the provider or required by the directly connected peer.

It is generally **not a design choice made independently of the WAN service**.

---

## 4. Key Configuration, Parameters, or CLI

> **Platform:** Cisco IOS / IOS XE.  
> Serial commands require a platform/interface that actually supports serial WAN interfaces. Many modern routers use Ethernet-based WAN handoffs instead.

---

## Cisco IOS / IOS XE — Cisco HDLC

```cisco
interface Serial0/0/0
 encapsulation hdlc
 ip address 192.0.2.1 255.255.255.252
 no shutdown
```

Verify:

```cisco
show interfaces Serial0/0/0
```

Look for:

```text
Encapsulation
Interface state
Line protocol state
Input/output errors
Keepalive behavior
```

---

## Cisco IOS / IOS XE — PPP

```cisco
interface Serial0/0/0
 encapsulation ppp
 ip address 192.0.2.1 255.255.255.252
 no shutdown
```

Verify:

```cisco
show interfaces Serial0/0/0
show ppp all
```

Command availability can vary by IOS/IOS XE release and platform.

---

## PPP with CHAP — IOS / IOS XE

Example peer names:

```text
R1
R2
```

R1:

```cisco
username R2 password <shared-secret>

interface Serial0/0/0
 encapsulation ppp
 ppp authentication chap
```

R2:

```cisco
username R1 password <shared-secret>

interface Serial0/0/0
 encapsulation ppp
 ppp authentication chap
```

The locally configured username entry represents the **remote peer's CHAP name**.

Use protected secrets and platform-supported secure secret storage where available.

---

## Ethernet WAN — IOS / IOS XE

Normal routed Ethernet handoff:

```cisco
interface GigabitEthernet0/0
 no switchport
 ip address 192.0.2.2 255.255.255.252
 no shutdown
```

On router platforms where the interface is Layer 3 by default, `no switchport` is not applicable.

Verify:

```cisco
show interfaces GigabitEthernet0/0
show ip interface brief
show ip route
```

---

## Tagged Ethernet WAN

```cisco
interface GigabitEthernet0/0.100
 encapsulation dot1Q 100
 ip address 192.0.2.2 255.255.255.252
```

Verify:

```cisco
show interfaces GigabitEthernet0/0.100
show ip interface brief
```

---

## Practical Troubleshooting Sequence

```text
1. Is the physical carrier/link Up?
        ↓
2. What encapsulation does the provider/peer require?
        ↓
3. Do both ends use compatible encapsulation?
        ↓
4. If PPP: does LCP establish?
        ↓
5. If configured: does authentication succeed?
        ↓
6. Does IPCP/IPv6CP establish?
        ↓
7. Is Layer 3 addressing correct?
        ↓
8. Is the route/next hop correct?
        ↓
9. Is VLAN tagging correct on Ethernet handoffs?
        ↓
10. Is MTU/encapsulation overhead causing drops?
```

Useful commands:

```cisco
show interfaces <interface>
show ip interface brief
show ip route
show ppp all
```

For Ethernet WANs also verify switchport/subinterface/VLAN information appropriate to the platform.

---

## 5. Common Gotchas and Misconceptions

### WAN Encapsulation Means PPP

**Incorrect.**

PPP is one WAN encapsulation.

Modern enterprise WAN handoffs are frequently:

```text
Ethernet
802.1Q-tagged Ethernet
```

The service determines the framing.

---

### An MPLS WAN Means the Customer Router Must Configure MPLS

**Incorrect in many enterprise L3VPN deployments.**

The customer edge may simply use Ethernet/IP routing toward the provider edge.

```text
Customer CE
→ Ethernet/IP
→ Provider PE
→ MPLS inside provider core
```

---

### GRE and IPsec Replace the Underlying WAN Encapsulation

**Incorrect.**

GRE/IPsec create additional overlay encapsulation.

The resulting outer packet still needs local link-layer framing:

```text
IPsec/GRE packet
      ↓
Ethernet / PPP / other WAN framing
```

---

### Layer 1 Up Means the WAN Link Is Working

**Incorrect.**

A direct serial link can have working physical signaling while Layer 2 is unusable.

Check both:

```text
Interface status
Line protocol status
```

and verify encapsulation/authentication where applicable.

---

### PPP Authentication Encrypts User Data

**Incorrect.**

PAP/CHAP authenticate the PPP peer.

They do not encrypt the IP payload carried by PPP.

Use technologies such as IPsec when confidentiality is required.

---

### PPP Automatically Provides Reliability

**Incorrect.**

PPP uses FCS to detect corrupted frames but does not provide TCP-style reliable delivery.

---

### VLAN Tagging Is a Layer 3 WAN Feature

**Incorrect.**

802.1Q is Layer 2 encapsulation.

The VLAN ID identifies the Ethernet broadcast domain/service instance; it is not part of the IP header.

---

### The Customer Can Choose Any Encapsulation on a Provider Handoff

**Incorrect.**

The CE encapsulation must match the provider handoff.

Examples:

```text
Provider expects untagged Ethernet
→ Send untagged Ethernet

Provider expects VLAN 100
→ Tag VLAN 100

Provider expects PPP
→ Use PPP
```

---

### MTU Is Unrelated to WAN Encapsulation

**Incorrect.**

Every additional header consumes bytes.

PPPoE, IPsec, GRE, and provider encapsulations can reduce the effective payload MTU or require larger physical/service MTUs.

---

## 6. Trade-Offs

### Best Practice

- Treat the provider handoff specification as authoritative for WAN encapsulation.
- Prefer standards-based encapsulation when direct multivendor interoperability matters.
- Verify Layer 1 and Layer 2 before troubleshooting routing.
- Account for all encapsulation overhead when setting MTU/MSS.
- Use IPsec or another appropriate security mechanism when the WAN underlay is not trusted and confidentiality/integrity are required.
- Document VLAN IDs, MTU, encapsulation type, peer addressing, and provider handoff details.

---

### Context-Dependent Trade-Off — Ethernet vs PPP

**Ethernet WAN**

```text
+ Dominant on modern provider handoffs
+ High speed and broad platform support
+ Natural VLAN/service multiplexing
- Provider-specific VLAN/MTU requirements may add complexity
```

**PPP**

```text
+ Standardized point-to-point link control
+ Optional peer authentication
+ Clear LCP/NCP state
- Primarily relevant to serial and PPPoE-style services
- Additional negotiation/overhead
```

The transport/provider service normally decides which model is applicable.

---

### Context-Dependent Trade-Off — Cisco HDLC vs PPP

**Cisco HDLC**

```text
+ Simple
+ Low configuration overhead
- Cisco-specific framing behavior
- Limited multivendor value
```

**PPP**

```text
+ Standards-based
+ Optional authentication
+ Better interoperability
- More protocol negotiation
```

On a direct multivendor serial link, PPP is generally the safer interoperability choice.

---

### Incorrect or Unsafe

- Changing encapsulation without coordinating the far end or provider.
- Treating Layer 1 Up as proof that Layer 2 is operational.
- Using PAP/CHAP as if they provide data confidentiality.
- Ignoring service MTU when adding PPPoE, GRE, IPsec, or additional tags.
- Assuming the provider's internal transport technology is the same encapsulation configured on the customer-facing interface.

---

## Quick Reference

```text
WAN Encapsulation
= Layer 2 framing used on a WAN-facing link

Core Rule
= L2 framing is hop-by-hop
= L3 IP packet is normally end-to-end

Modern WAN Handoff
= Commonly Ethernet

802.1Q WAN
= Ethernet + VLAN tag

PPP
= Standards-based point-to-point encapsulation

PPP Sequence
= LCP
→ Optional authentication
→ NCP
→ Layer 3 traffic

PAP / CHAP
= PPP peer authentication
= Not user-data encryption

Cisco HDLC
= Cisco point-to-point serial encapsulation

PPPoE
= PPP carried over Ethernet

Common PPPoE IP MTU
= 1492 bytes

GRE / IPsec
= Overlay/tunnel encapsulation
= Still require underlying WAN framing

MPLS L3VPN
= Provider may use MPLS internally
= CE often sees Ethernet/IP only

Troubleshooting
= Physical → Encapsulation → PPP state/auth → L3 → Routing → MTU
```

</div>
