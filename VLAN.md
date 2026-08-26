<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# VLAN Fundamentals

> **Core idea:** A VLAN logically separates Layer 2 networks.  
> **One VLAN = one Layer 2 broadcast domain.**

---

## Local Area Network (LAN)

A **LAN** is a single broadcast domain, including all devices in that broadcast domain.

A single physical LAN can contain **multiple VLANs**, and therefore multiple broadcast domains.

---

## Broadcast Domain

A **broadcast domain** is the group of Layer 2 devices that receive a broadcast frame sent by any member of that domain.

Ethernet broadcast destination MAC:

```text
FFFF.FFFF.FFFF
```

Broadcast traffic is flooded only within the same VLAN/broadcast domain.

---

## Broadcast Frame

A **broadcast frame** is an Ethernet frame intended for every device in the local broadcast domain.

Common examples include:

```text
ARP requests
DHCP discovery
```

Excessive broadcast traffic consumes bandwidth and endpoint processing resources, which is one reason VLANs are used to limit broadcast scope.

---

# VLAN

A **VLAN (Virtual Local Area Network)** logically separates devices at Layer 2.

```text
One VLAN = One Layer 2 broadcast domain
```

Devices can be connected to the same physical switch but belong to different VLANs and therefore different Layer 2 networks.

### Purpose of VLANs

**1. Network Performance**

VLANs reduce the scope of broadcast and unknown-unicast flooding, which helps limit unnecessary Layer 2 traffic.

**2. Segmentation and Security**

VLANs separate Layer 2 traffic between groups of devices. Traffic between VLANs must cross a Layer 3 boundary, where routing and security policy can be applied.

> **Important:** A VLAN provides segmentation, but it is not a complete security control by itself.

---

## Inter-VLAN Communication

A Layer 2 switch does not forward frames directly between different VLANs.

```text
VLAN 10  --X--  VLAN 20
```

Communication between VLANs requires a **Layer 3 device**, such as:

```text
Router
Multilayer switch using SVIs
Firewall
```

This process is called **inter-VLAN routing**.

---

# Access Ports

An **access port** normally carries traffic for **one data VLAN** and is typically connected to an endpoint.

Key characteristics:

```text
One data VLAN
Typically endpoint-facing
Frames are normally untagged on the wire
```

### Cisco IOS / IOS XE

```cisco
interface GigabitEthernet1/0/10
 switchport mode access
 switchport access vlan 10
```

---

# Trunk Ports

A **trunk port** carries traffic for **multiple VLANs** over one physical link.

Typical connections include:

```text
Switch <-> Switch
Switch <-> Router
Switch <-> Firewall
Switch <-> Virtualization host
```

Cisco Ethernet trunks normally use **IEEE 802.1Q** tagging.

```text
VLAN 10 -> tagged
VLAN 20 -> tagged
VLAN 30 -> tagged
```

### Cisco IOS / IOS XE

```cisco
interface GigabitEthernet1/0/48
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
```

---

# Native VLAN

The **native VLAN** is the VLAN associated with **untagged frames received on an 802.1Q trunk**.

On Cisco switches, the default native VLAN is typically:

```text
VLAN 1
```

It can be changed manually.

```cisco
interface GigabitEthernet1/0/48
 switchport mode trunk
 switchport trunk native vlan 999
```

### Why the Native VLAN Must Match

If the two sides of a trunk use different native VLANs, the same untagged frame can be placed into different VLANs on each switch.

> **Rule:** The native VLAN should match on both ends of an 802.1Q trunk.

---

# VLAN Ranges

The 802.1Q VLAN ID field is 12 bits.

Usable VLAN IDs are:

```text
1-4094
```

Cisco commonly classifies them as:

| VLAN Range | Type |
|---|---|
| **1-1005** | Normal-range VLANs |
| **1006-4094** | Extended-range VLANs |

VLAN IDs **0** and **4095** are reserved and are not used as normal VLAN IDs.

---

# Quick Reference

```text
LAN
= Local network that may contain multiple VLANs

Broadcast Domain
= Devices that receive the same Layer 2 broadcast

VLAN
= Logical Layer 2 segmentation

1 VLAN
= 1 broadcast domain

Access Port
= Normally carries one data VLAN

Trunk Port
= Carries multiple VLANs

802.1Q
= VLAN tagging standard

Native VLAN
= VLAN assigned to untagged frames on a trunk

Different VLANs
= Layer 3 routing required

Usable VLAN IDs
= 1-4094

Normal Range
= 1-1005

Extended Range
= 1006-4094
```

</div>
