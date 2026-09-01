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

## CCNA Configuration

**VLAN Creation and Management — Cisco IOS XE**

| Command | Description |
|---|---|
| **Create VLAN:**<br>`(config)#vlan <vlan-id>` | Creates a VLAN and enters VLAN configuration mode. |
| **Name VLAN:**<br>`(config-vlan)#name <vlan-name>` | Assigns a descriptive name to the current VLAN. |
| **Enable VLAN:**<br>`(config-vlan)#no shutdown` | Enables the current VLAN. |
| **Disable VLAN:**<br>`(config-vlan)#shutdown` | Administratively disables the current VLAN. |
| **Verify VLANs:**<br>`#show vlan brief` | Lists VLANs, status, and assigned access ports. |
| **Verify VLAN by ID:**<br>`#show vlan id <vlan-id>` | Displays detailed information for one VLAN. |
| **Verify VLAN by name:**<br>`#show vlan name <vlan-name>` | Displays detailed information for one named VLAN. |

**Access Port VLAN Assignment — Cisco IOS XE**

| Command | Description |
|---|---|
| **Configure access port:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#switchport mode access`<br>&nbsp;&nbsp;○ `(config-if)#switchport access vlan <vlan-id>` | Forces access mode and assigns the port to one VLAN. |
| **Configure access port range:**<br>`(config)#interface range <interface-range>`<br>&nbsp;&nbsp;○ `(config-if-range)#switchport mode access`<br>&nbsp;&nbsp;○ `(config-if-range)#switchport access vlan <vlan-id>` | Assigns multiple access ports to the same VLAN. |
| **Verify switchport VLAN:**<br>`#show interfaces <interface-id> switchport` | Displays administrative, operational, access, native, and voice VLAN settings. |
| **Verify interface VLANs:**<br>`#show interfaces status` | Lists access VLANs and identifies operational trunk ports. |

**Data and Voice VLANs — Cisco IOS XE**

| Command | Description |
|---|---|
| **Configure data and voice VLANs:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#switchport mode access`<br>&nbsp;&nbsp;○ `(config-if)#switchport access vlan <data-vlan-id>`<br>&nbsp;&nbsp;○ `(config-if)#switchport voice vlan <voice-vlan-id>` | Assigns separate data and voice VLANs to one access port. |
| **Verify voice VLAN:**<br>`#show interfaces <interface-id> switchport` | Displays configured access and voice VLAN assignments. |

**802.1Q Trunk Configuration — Cisco IOS XE**

| Command | Description |
|---|---|
| **Configure static trunk:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#switchport mode trunk` | Statically configures the interface as an 802.1Q trunk. |
| **Set native VLAN:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#switchport trunk native vlan <vlan-id>` | Sets the native VLAN for the trunk. |
| **Set allowed VLAN list:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#switchport trunk allowed vlan <vlan-list>` | Replaces the trunk allowed-VLAN list. |
| **Add allowed VLANs:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#switchport trunk allowed vlan add <vlan-list>` | Adds VLANs to the existing allowed-VLAN list. |
| **Remove allowed VLANs:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#switchport trunk allowed vlan remove <vlan-list>` | Removes VLANs from the existing allowed-VLAN list. |
| **Set allowed VLAN policy:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#switchport trunk allowed vlan {all | none | except <vlan-list>}` | Sets all, none, or all-except VLAN trunk membership. |
| **Verify operational trunks:**<br>`#show interfaces trunk` | Lists operational trunks, native VLANs, and allowed VLANs. |
| **Verify one trunk:**<br>`#show interfaces <interface-id> trunk` | Displays trunking status and VLAN lists for one interface. |
| **Verify trunk switchport:**<br>`#show interfaces <interface-id> switchport` | Displays administrative and operational trunking parameters. |

**Router-on-a-Stick Inter-VLAN Routing — Cisco IOS XE Router**

| Command | Description |
|---|---|
| **Create VLAN subinterface:**<br>`(config)#interface <physical-interface>.<subinterface-number>`<br>&nbsp;&nbsp;○ `(config-subif)#encapsulation dot1q <vlan-id>`<br>&nbsp;&nbsp;○ `(config-subif)#ip address <ip-address> <subnet-mask>` | Creates a routed 802.1Q subinterface for one VLAN. |
| **Configure native VLAN subinterface:**<br>`(config)#interface <physical-interface>.<subinterface-number>`<br>&nbsp;&nbsp;○ `(config-subif)#encapsulation dot1q <vlan-id> native`<br>&nbsp;&nbsp;○ `(config-subif)#ip address <ip-address> <subnet-mask>` | Associates the native VLAN with a routed subinterface. |
| **Enable parent interface:**<br>`(config)#interface <physical-interface>`<br>&nbsp;&nbsp;○ `(config-if)#no shutdown` | Enables the physical interface carrying the 802.1Q trunk. |
| **Verify router VLANs:**<br>`#show vlans` | Displays router VLAN trunk configuration and statistics. |
| **Verify connected routes:**<br>`#show ip route connected` | Lists connected routes created by VLAN subinterfaces. |

**SVI Inter-VLAN Routing — Cisco IOS XE Layer 3 Switch**

| Command | Description |
|---|---|
| **Create SVI:**<br>`(config)#interface vlan <vlan-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip address <ip-address> <subnet-mask>`<br>&nbsp;&nbsp;○ `(config-if)#no shutdown` | Creates and addresses a Layer 3 VLAN interface. |
| **Enable IPv4 routing:**<br>`(config)#ip routing` | Enables IPv4 routing on the Layer 3 switch. |
| **Verify SVI status:**<br>`#show ip interface brief` | Displays SVI addressing and line-protocol status. |
| **Verify SVI details:**<br>`#show interfaces vlan <vlan-id>` | Displays detailed status and counters for one SVI. |
| **Verify connected routes:**<br>`#show ip route connected` | Lists connected routes learned from active SVIs. |

## CCNP Configuration

**Dynamic 802.1Q Trunking — CCNP Enterprise (ENCOR) — Cisco IOS XE**

| Command | Description |
|---|---|
| **Configure dynamic desirable:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#switchport mode dynamic desirable` | Actively negotiates trunk formation using DTP. |
| **Configure dynamic auto:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#switchport mode dynamic auto` | Passively negotiates trunk formation using DTP. |
| **Disable DTP on static trunk:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#switchport mode trunk`<br>&nbsp;&nbsp;○ `(config-if)#switchport nonegotiate` | Configures a static trunk and disables DTP negotiation. |
| **Verify dynamic trunk:**<br>`#show interfaces <interface-id> trunk` | Displays configured mode and operational trunk status. |
| **Verify DTP state:**<br>`#show interfaces <interface-id> switchport` | Displays trunk negotiation and operational switchport parameters. |

**VLAN Trunking Protocol (VTP) — CCNP Enterprise (ENCOR) — Cisco IOS XE**

| Command | Description |
|---|---|
| **Set VTP version:**<br>`(config)#vtp version {1 | 2 | 3}` | Selects the VTP protocol version. |
| **Set VTP domain:**<br>`(config)#vtp domain <domain-name>` | Assigns the switch to a VTP domain. |
| **Set VTP mode:**<br>`(config)#vtp mode {server | client | transparent | off}` | Configures the switch VTP operating role. |
| **Set VTP password:**<br>`(config)#vtp password <password>` | Configures VTP domain authentication. |
| **Set VTPv3 primary server:**<br>`#vtp primary` | Promotes the VTPv3 server to primary status. |
| **Verify VTP:**<br>`#show vtp status` | Displays VTP version, domain, mode, VLANs, and revision. |


</div>
