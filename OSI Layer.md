<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# OSI Model

> **Core idea:** The OSI model is a **seven-layer reference model** used to describe and troubleshoot network communication by separating functions into logical layers. Modern networks primarily run TCP/IP, but OSI terminology—especially **Layer 1 through Layer 7**—remains the standard way engineers describe where a protocol, device, or fault operates.

---

## 1. What It Is

The **Open Systems Interconnection (OSI) model** is a conceptual framework that divides network communication into seven functional layers. It is **not the protocol stack normally running on modern networks**; TCP/IP is, while OSI provides a common vocabulary for architecture, protocol behavior, and troubleshooting.

---

## 2. How It Works

### The Seven Layers

| Layer | Name | Primary Responsibility | Common Examples / Concepts |
|---:|---|---|---|
| **7** | Application | Network services used by applications | HTTP(S), DNS, SSH, SMTP |
| **6** | Presentation | Data representation and transformation | Encoding, serialization, compression, encryption concepts |
| **5** | Session | Establishing and managing application sessions | Session establishment, maintenance, termination |
| **4** | Transport | End-to-end application delivery | TCP, UDP, ports, reliability, flow control |
| **3** | Network | Logical addressing and routing | IPv4, IPv6, routing |
| **2** | Data Link | Local-link framing and forwarding | Ethernet, MAC addresses, VLANs, switching |
| **1** | Physical | Transmission of bits across media | Copper, fiber, radio, signaling |

> Layers 5–7 are often implemented together by modern applications and protocols rather than as clearly separate protocol layers.

---

### Encapsulation

When a host sends data, information moves **down** the protocol stack. Each relevant layer adds the information required for its function.

```text
Layer 7-5  Application Data
              ↓
Layer 4    TCP Segment / UDP Datagram
              ↓
Layer 3    IP Packet
              ↓
Layer 2    Ethernet / Wi-Fi Frame
              ↓
Layer 1    Bits / Signals
```

The receiving device performs **decapsulation** in the opposite direction:

```text
Bits
 ↓
Frame
 ↓
Packet
 ↓
Segment / Datagram
 ↓
Application Data
```

Each layer provides services to the layer above it while relying on services from the layer below it.

---

### PDU Terminology

A **Protocol Data Unit (PDU)** is the data structure associated with a protocol layer.

```text
Layers 5-7 → Data
Layer 4    → Segment (TCP) / Datagram (UDP)
Layer 3    → Packet
Layer 2    → Frame
Layer 1    → Bits
```

These terms identify the portion of communication being discussed, not separate end-to-end messages.

---

### What Changes Across a Routed Path

Consider:

```text
Host A ---- Switch ---- Router ---- Switch ---- Host B
```

The Layer 3 packet is routed toward the final destination, while Layer 2 framing is local to each link.

```text
Layer 3:
Source IP      → normally remains end to end
Destination IP → normally remains end to end

Layer 2:
Source MAC      → changes at routed boundaries
Destination MAC → changes at routed boundaries
```

NAT is an exception because it intentionally changes Layer 3 addressing and may also modify Layer 4 identifiers.

---

### OSI vs TCP/IP

The OSI and commonly used five-layer TCP/IP models map closely at the lower layers:

| OSI | TCP/IP |
|---|---|
| **7 Application** | Application |
| **6 Presentation** | Application |
| **5 Session** | Application |
| **4 Transport** | Transport |
| **3 Network** | Network |
| **2 Data Link** | Data Link |
| **1 Physical** | Physical |

Conceptually:

```text
OSI                    TCP/IP

7 Application  ┐
6 Presentation ├────── Application
5 Session      ┘

4 Transport    ──────  Transport
3 Network      ──────  Network
2 Data Link    ──────  Data Link
1 Physical     ──────  Physical
```

This is why modern engineers still say:

```text
Layer 2 switch
Layer 3 routing
Layer 4 firewall rule
Layer 7 application inspection
```

even though the actual network is using TCP/IP rather than a complete OSI protocol suite.

---

## 3. Why and When It Is Used

The OSI model is primarily useful as a **technical abstraction**.

It helps engineers:

- Describe where a protocol or function operates.
- Separate forwarding, transport, and application behavior.
- Troubleshoot systematically instead of treating connectivity as one undifferentiated problem.
- Communicate consistently across vendors and technologies.

A practical troubleshooting flow is:

```text
Layer 1 → Is the link physically working?
   ↓
Layer 2 → Is switching/VLAN/MAC forwarding correct?
   ↓
Layer 3 → Is addressing and routing correct?
   ↓
Layer 4 → Is the required TCP/UDP transport reachable?
   ↓
Layer 5-7 → Is the application/session functioning?
```

The OSI model is **not** something that must be configured or enabled, and it should not be treated as a literal description of how every modern protocol is internally implemented.

---

## 4. Key Configuration, Parameters, or CLI

The OSI model itself has **no configuration commands**. The useful CLI is the platform-specific evidence used to isolate the failing layer.

### Cisco IOS / IOS XE

#### Layer 1 — Physical

```cisco
show interfaces status
show interfaces <interface>
```

Check for:

```text
Interface state
Speed / duplex
Errors
CRC counters
Link transitions
```

---

#### Layer 2 — Data Link

```cisco
show mac address-table
show vlan brief
show interfaces trunk
show spanning-tree
show etherchannel summary
```

Use these to validate VLAN membership, MAC learning, trunking, STP, and EtherChannel behavior.

---

#### Layer 3 — Network

```cisco
show ip interface brief
show ip route
show ip route <destination>
show ip arp
ping <destination>
traceroute <destination>
```

Use these to validate addressing, next-hop resolution, routing, and end-to-end IP reachability.

---

#### Layer 4 — Transport

For connections terminated on the IOS/IOS XE device:

```cisco
show tcp brief all
```

For transit application troubleshooting, packet capture, ACL/firewall counters, or endpoint testing is usually more useful than a router-local TCP table.

---

### Operational Troubleshooting Sequence

```text
Physical link
     ↓
VLAN / Layer 2 path
     ↓
IP addressing
     ↓
Routing / next-hop resolution
     ↓
TCP / UDP transport
     ↓
Application
```

Do not continue upward until the lower-layer dependency has been verified.

---

## 5. Common Gotchas and Misconceptions

### The OSI Model Is the Protocol Stack Used by the Internet

**Incorrect.** Modern networks primarily use the **TCP/IP protocol suite**. OSI is mainly a reference model and terminology framework.

---

### Every Protocol Fits Perfectly Into One Layer

**Incorrect.** The model is an abstraction. Some protocols and functions cross conceptual boundaries or combine responsibilities associated with multiple OSI layers.

Do not force an exact layer assignment when the protocol design does not support one cleanly.

---

### Layers 5, 6, and 7 Always Exist as Separate Protocols

**Incorrect.** In modern TCP/IP implementations, their functions are usually combined within the application layer.

---

### A Layer 3 Problem Means Everything Below Layer 3 Is Working

**Not necessarily.** Intermittent Layer 1 or Layer 2 faults can present as routing or application symptoms.

Validate evidence at each dependency layer instead of diagnosing solely from the visible symptom.

---

### Ping Tests the Entire OSI Stack

**Incorrect.** `ping` primarily tests IP/ICMP reachability. It does not prove that the required TCP/UDP port or application is functioning.

```text
Ping succeeds
≠
Application succeeds
```

---

### Layer Numbers Describe Devices Absolutely

Terms such as **Layer 2 switch**, **Layer 3 switch**, and **Layer 7 firewall** describe major functions, not necessarily the only capabilities of the device.

Modern platforms commonly operate across multiple layers.

---

## 6. Trade-Offs

### Best Practice

Use the OSI model as a **troubleshooting and communication framework**, especially when isolating faults from physical connectivity through application behavior.

---

### Context-Dependent Trade-Off

Layer-based troubleshooting is useful, but real production failures can cross layers:

```text
MTU issue
→ Layer 2 constraint
→ Layer 3 fragmentation/PMTUD behavior
→ Layer 4 retransmissions
→ Application failure
```

Use the model to organize evidence, not to artificially constrain the investigation.

---

### Incorrect or Unsafe

- Treating OSI layers as strict implementation boundaries for every modern protocol.
- Assuming a symptom observed at one layer proves the root cause exists at that layer.
- Skipping lower-layer verification because higher-layer tests partially succeed.

---

## Quick Reference

```text
7  Application   → Application services
6  Presentation  → Data representation
5  Session       → Session management
4  Transport     → TCP/UDP, ports, end-to-end delivery
3  Network       → IP addressing and routing
2  Data Link     → Frames, MAC, VLANs, switching
1  Physical      → Bits, signaling, media
```

### PDU Memory Map

```text
Layer 7-5 → Data
Layer 4   → Segment / Datagram
Layer 3   → Packet
Layer 2   → Frame
Layer 1   → Bits
```

### Troubleshooting Memory Map

```text
L1 → Physical
L2 → Switching
L3 → Routing
L4 → Transport
L5-7 → Application
```

> **The OSI model does not run the network. It gives engineers a structured way to describe how the network works and where it is failing.**

</div>
