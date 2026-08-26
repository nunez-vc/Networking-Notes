# TCP/IP

> **Core idea:** TCP/IP is the protocol architecture used to move application data across interconnected IP networks. It combines application protocols, transport protocols such as TCP/UDP, IP addressing and routing, and link-layer technologies such as Ethernet or Wi-Fi.

---

## 1. What It Is

**TCP/IP** is a protocol suite and networking model, not a single protocol. It defines how applications exchange data end to end using transport protocols, how IP addresses and routes deliver packets between networks, and how each local link carries those packets.

Cisco commonly represents TCP/IP with five layers:

| TCP/IP Layer | Main Function | Common Examples |
|---|---|---|
| **Application** | Network services used by applications | HTTP(S), DNS, DHCP, SSH |
| **Transport** | Application-to-application delivery | TCP, UDP |
| **Network** | Logical addressing and packet forwarding | IPv4, IPv6, ICMP |
| **Data Link** | Local-link framing and delivery | Ethernet, 802.11 |
| **Physical** | Transmission of bits | Copper, fiber, radio |

> The original TCP/IP model is often shown with four layers by combining Data Link and Physical into a single **Network Access/Link** layer. Both representations describe the same architecture.

---

## 2. How It Works

### Encapsulation and Decapsulation

When an application sends data, each layer adds the information required for its function:

```text
Application Data
      ↓
TCP Segment / UDP Datagram
      ↓
IP Packet
      ↓
Ethernet / Wi-Fi Frame
      ↓
Bits
```

The receiver reverses the process:

```text
Bits
 ↓
Frame
 ↓
IP Packet
 ↓
TCP Segment / UDP Datagram
 ↓
Application Data
```

Typical PDU terminology:

```text
Transport layer → Segment (TCP) / Datagram (UDP)
Network layer   → Packet
Data-link layer → Frame
```

---

### Application Layer

Application protocols define how network applications exchange information.

Examples:

```text
HTTPS → Web traffic
DNS   → Name resolution
DHCP  → Address configuration
SSH   → Secure remote access
```

The application layer passes data to either TCP, UDP, or another appropriate transport mechanism.

DNS and DHCP are useful TCP/IP services, but they are **not required for IP forwarding itself**. A host can communicate using manually configured addressing and direct IP addresses.

---

### Transport Layer

The transport layer provides **process-to-process communication** using port numbers.

```text
IP address → Identifies the host
Port       → Identifies the application/process
```

A flow is identified by information such as:

```text
Source IP
Destination IP
Transport protocol
Source port
Destination port
```

### TCP

**TCP** provides a connection-oriented, reliable, ordered byte stream.

Connection establishment uses the three-way handshake:

```text
Client                     Server

SYN ----------------------->
    <------------------ SYN, ACK
ACK ----------------------->

Connection established
```

TCP then uses:

- **Sequence numbers** to track transmitted data
- **Acknowledgments** to confirm received data
- **Retransmission** to recover lost data
- **Receive windows** for flow control
- **Ordered delivery** to present data correctly to the application

TCP connection termination commonly uses FIN/ACK exchanges from both endpoints.

TCP provides reliability **between endpoints**; routers simply forward the IP packets carrying TCP segments.

---

### UDP

**UDP** provides connectionless datagram delivery with minimal transport-layer overhead.

UDP provides:

- Source and destination ports
- Multiplexing between applications
- Datagram integrity checking

UDP does **not** provide:

- Connection establishment
- Retransmission
- Ordered delivery
- TCP-style flow control

Applications using UDP either tolerate loss or implement any required recovery themselves.

> UDP is not inherently "worse" than TCP. It is appropriate when low overhead, low latency, or application-controlled reliability is more important than TCP's built-in reliability.

---

### IP and Routing

IP provides **logical addressing and best-effort packet delivery** across interconnected networks.

A host evaluates the destination address:

```text
Destination is local?
        |
   +----+----+
   |         |
  Yes        No
   |         |
Send to      Send to
destination  default gateway
```

For a remote destination:

```text
Host
  ↓
Default Gateway
  ↓
Router
  ↓
Router
  ↓
Destination
```

Each router:

1. Removes the incoming Layer 2 frame.
2. Examines the destination IP address.
3. Selects the best route.
4. Decrements the IP TTL/Hop Limit.
5. Builds a new Layer 2 frame for the next link.
6. Forwards the packet.

Therefore:

```text
IP source/destination
= Normally remain end to end

Layer 2 source/destination
= Change at every routed hop
```

NAT is an exception because it intentionally modifies IP addressing and sometimes transport identifiers.

---

### Local Next-Hop Resolution

Before an IP packet can be transmitted on a local Ethernet segment, the sender needs the Layer 2 address of the next hop.

```text
IPv4 → ARP
IPv6 → Neighbor Discovery (NDP)
```

For a local destination, the host resolves the destination itself.

For a remote destination, the host resolves the **default gateway/next hop**, not the remote endpoint.

---

### Control Plane vs Data Plane

At a router:

```text
Control Plane
= Builds and maintains routing/neighbor information

Data Plane
= Uses forwarding information to move packets
```

Routing protocols, static routes, and connected networks populate routing information. The forwarding plane then uses that information for each packet.

TCP and UDP reliability/session behavior occurs primarily at the **end hosts**, not in transit routers.

---

## 3. Why and When It Is Used

TCP/IP provides a common architecture for communication across:

- LANs
- Enterprise networks
- Data centers
- WANs
- Cloud networks
- The Internet

It solves four core requirements:

```text
Application communication
        ↓
Process identification with ports
        ↓
End-to-end IP addressing and routing
        ↓
Delivery across each physical/local link
```

TCP is appropriate when the application requires reliable, ordered delivery.

UDP is appropriate when low overhead, timeliness, or application-controlled recovery is preferred.

IP is used whenever hosts must communicate across IP networks, regardless of whether the underlying link is Ethernet, Wi-Fi, MPLS transport, tunnels, or another supported medium.

---

## 4. Key Configuration, Parameters, or CLI

TCP/IP itself is a protocol architecture, so there is no single "enable TCP/IP" command on modern Cisco routers. The most relevant configuration is IP addressing and routing.

### Cisco IOS / IOS XE — IPv4 Interface

```cisco
interface GigabitEthernet0/0
 ip address 192.0.2.1 255.255.255.0
 no shutdown
```

Verify addressing and interface state:

```cisco
show ip interface brief
show ip interface GigabitEthernet0/0
```

Verify routing:

```cisco
show ip route
show ip route <destination>
```

Verify local IPv4 neighbor resolution:

```cisco
show ip arp
```

Test end-to-end IP reachability and path:

```cisco
ping <destination>
traceroute <destination>
```

Inspect locally terminated TCP connections when relevant:

```cisco
show tcp brief all
```

A useful troubleshooting sequence is:

```text
Interface up?
     ↓
Correct IP/mask?
     ↓
Correct route?
     ↓
Next-hop ARP/NDP resolved?
     ↓
Packet reaches destination?
     ↓
Transport port/session working?
     ↓
Application responding?
```

---

## 5. Common Gotchas and Misconceptions

### TCP/IP Means Only TCP and IP

**Incorrect.** TCP/IP refers to the entire protocol architecture. TCP and IP are only two protocols within it.

---

### TCP Guarantees That an Application Will Work

**Incorrect.** TCP can provide reliable byte delivery between endpoints, but application failures can still occur because of DNS, TLS, authentication, server state, application logic, firewall policy, or other dependencies.

---

### UDP Is Unreliable Therefore It Is Bad

**Incorrect.** UDP intentionally provides fewer transport services. Real-time applications and protocols that implement their own recovery may prefer UDP.

---

### IP Provides Reliable Delivery

**Incorrect.** IP provides **best-effort** packet delivery. Reliability, when required, is provided by TCP or the application.

---

### Port Numbers Identify Hosts

**Incorrect.**

```text
IP address → Host/interface
Port       → Application/process
```

Both are needed to identify an application endpoint.

---

### MAC Addresses Are End-to-End

**Incorrect.** Layer 2 addressing is local to each link.

```text
IP packet
= Routed end to end

Ethernet frame
= Rebuilt at each routed hop
```

---

### DNS Is Required for TCP/IP Communication

**Incorrect.** DNS resolves names to addresses. IP communication can work without DNS when the destination IP is already known.

---

### TCP/IP Provides Encryption

**Incorrect.** TCP and IP do not inherently encrypt application traffic.

Encryption is provided by mechanisms such as:

```text
TLS
IPsec
SSH
Application-specific encryption
```

---

### A Successful Ping Proves the Application Works

**Incorrect.** Ping verifies ICMP reachability, not TCP/UDP port availability or application health.

```text
Ping succeeds
≠
TCP/UDP application succeeds
```

---

### MTU Problems Affect Only Layer 2

**Incorrect.** The Layer 2 MTU constrains the IP packet that can cross a link. MTU or Path MTU problems can cause fragmentation, packet drops, or application behavior where small packets work but larger transfers fail.

---

## 6. Trade-Offs

### Best Practice

- Troubleshoot TCP/IP **layer by layer**, starting with IP addressing and reachability before investigating transport or application behavior.
- Use the routing table to determine the next hop, then ARP/NDP to verify Layer 2 resolution.
- Choose TCP or UDP based on application requirements rather than assuming one is universally better.
- Treat encryption and access control as separate security requirements.

---

### Context-Dependent Trade-Off — TCP vs UDP

| TCP | UDP |
|---|---|
| Reliable, ordered delivery | No built-in retransmission or ordering |
| Connection-oriented | Connectionless |
| Flow control | No TCP-style flow control |
| More protocol state/overhead | Lower protocol overhead |
| Appropriate for loss-intolerant data | Appropriate for latency-sensitive or application-controlled traffic |

The correct transport depends on the application.

---

### Incorrect or Unsafe

- Treating TCP reliability as equivalent to security
- Assuming UDP traffic does not require security controls
- Troubleshooting an application before verifying IP addressing, routing, and next-hop resolution
- Assuming a working Layer 3 path means the required transport port or application is reachable
- Assuming IP addresses or transport ports remain unchanged when NAT is present

---

## Quick Reference

```text
TCP/IP
= Networking protocol architecture

Application
= Network services and application protocols

TCP
= Reliable, ordered, connection-oriented byte stream

UDP
= Connectionless datagram transport

Ports
= Identify applications/processes

IP
= Logical addressing + best-effort packet delivery

Routing
= Selects the path/next hop for IP packets

ARP
= IPv4 next-hop IP → MAC resolution

NDP
= IPv6 neighbor resolution

Encapsulation
= Data → Segment/Datagram → Packet → Frame → Bits

At a Router
= Remove old frame → route IP packet → build new frame

IP addresses
= Normally end-to-end

Layer 2 addresses
= Change at routed hops

TCP/IP ≠ Encryption
TCP/IP ≠ Firewall
Ping success ≠ Application success
```
