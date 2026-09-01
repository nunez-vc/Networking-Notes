<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# Cisco Discovery Protocol (CDP)

> **Core idea:** CDP is a Cisco Layer 2 neighbor-discovery protocol that lets directly connected Cisco devices advertise identity, interface, platform, capabilities, and selected operational information. It works without IP reachability and is useful for topology discovery and troubleshooting, but it also exposes device information and should be limited to trusted links.

---

## 1. What It Is

**CDP (Cisco Discovery Protocol)** is a Cisco-proprietary Layer 2 discovery protocol used between directly connected devices. It advertises local device and interface information so neighbors can identify what is connected on each link.

CDP does **not** provide routing, switching, or reachability by itself.

```text
CDP
= Neighbor discovery

Not
= Routing protocol
= Management protocol
= Security protocol
```

---

## 2. How It Works

## Link-Local Operation

CDP operates directly at Layer 2 and does not require:

```text
IPv4
IPv6
TCP
UDP
```

A device periodically sends CDP advertisements out interfaces where CDP is enabled.

```text
R1 Gi0/0
   |
   | CDP advertisement
   v
SW1 Gi1/0/24
```

The receiving device stores the information in its **CDP neighbor table**.

CDP is link-local:

```text
Neighbor A ---- Neighbor B
```

It discovers directly connected peers only.

CDP advertisements are not routed between IP networks.

---

## Ethernet Encapsulation

On Ethernet, CDP uses a Cisco multicast destination MAC:

```text
01:00:0C:CC:CC:CC
```

CDP uses IEEE 802.2 LLC/SNAP encapsulation rather than an IP packet.

Conceptually:

```text
Ethernet Frame
   |
   +-- Destination MAC: 01:00:0C:CC:CC:CC
   +-- LLC/SNAP
   +-- CDP payload
```

Because CDP is not IP-based, two devices can discover each other even if their Layer 3 addressing is missing or incorrect.

---

## Advertisement Timers

Common Cisco IOS/IOS XE defaults are:

```text
Advertisement interval = 60 seconds
Holdtime               = 180 seconds
```

The holdtime tells the receiving device how long to retain the neighbor entry if no further CDP advertisements arrive.

```text
CDP update received
       ↓
Neighbor stored
       ↓
No more updates
       ↓
Holdtime expires
       ↓
Neighbor removed
```

---

## Information Advertised

CDP uses **Type-Length-Value (TLV)** fields to carry neighbor information.

Common information includes:

```text
Device ID
Local/remote port identifiers
Platform
Software version
Device capabilities
Management/IP address information
Native VLAN
Duplex information
VTP domain
Power-related information on supported platforms
```

The exact TLVs depend on device type, CDP version, platform, and software release.

---

## CDP Versions

Modern Cisco platforms normally use **CDPv2**.

CDPv2 supports additional information compared with CDPv1, including fields useful for detecting issues such as:

```text
Native VLAN mismatch
Duplex mismatch
```

Do not assume every CDP-capable device exposes every TLV.

---

## Neighbor Table Behavior

Assume:

```text
SW1 Gi1/0/24 ---- Gi0/0 R1
```

SW1 can learn:

```text
Neighbor Device ID  = R1
Local Interface     = Gi1/0/24
Neighbor Port       = Gi0/0
Platform            = Cisco router
Capabilities        = Router
Management Address  = If advertised
```

This creates a direct physical/logical link map:

```text
Local interface
     ↓
Remote device
     ↓
Remote interface
```

That makes CDP especially useful when tracing cabling or validating topology.

---

## CDP and IP Reachability

A CDP neighbor relationship proves only that:

```text
Layer 2 CDP frames are being exchanged
```

It does **not** prove:

```text
IP addressing is correct
Routing is correct
VLANs are correct end to end
ACL/firewall policy permits traffic
Applications are reachable
```

Example:

```text
CDP neighbor visible
        +
Wrong IP subnet
        =
No Layer 3 connectivity
```

---

## CDP and VLANs

CDP can operate across:

```text
Access links
802.1Q trunks
Routed Ethernet links
```

On Cisco switching infrastructure, CDP information can help identify:

```text
Unexpected neighbor
Wrong physical port
Native VLAN mismatch
Duplex mismatch
Incorrect uplink cabling
```

> CDP does not negotiate or create VLANs. It only advertises information.

---

## CDP and Cisco IP Phones

Cisco IP phones can use CDP to learn information such as:

```text
Voice VLAN
Power-related parameters
Switch/port information
```

In some environments, LLDP-MED can provide similar functionality.

Therefore, disabling CDP on phone-facing ports should be validated against the specific phone, switch, power, and voice-VLAN design.

---

## 3. Why and When It Is Used

CDP is useful when operating Cisco infrastructure and you need to identify directly connected devices without relying on documentation or IP reachability.

Typical uses:

- Discover physical topology.
- Identify which switch port connects to a router, switch, AP, phone, or other Cisco device.
- Verify expected uplink relationships.
- Find remote interface names.
- Detect unexpected Cisco devices.
- Troubleshoot native VLAN or duplex inconsistencies.
- Confirm adjacency when Layer 3 addressing is not yet working.

CDP is most appropriate on **trusted Cisco infrastructure links**.

It is less appropriate on:

```text
Internet-facing links
Provider handoffs
Untrusted external connections
General user-facing ports where CDP is not required
```

because it can disclose useful device information to a directly connected system.

---

## 4. Key Configuration, Parameters, or CLI

> **Platform:** Cisco IOS / IOS XE.

---

## Global Enable / Disable

Enable CDP globally:

```cisco
cdp run
```

Disable CDP globally:

```cisco
no cdp run
```

Global disable stops CDP on all interfaces.

---

## Per-Interface Enable / Disable

Enable CDP on an interface:

```cisco
interface GigabitEthernet1/0/24
 cdp enable
```

Disable CDP on an interface:

```cisco
interface GigabitEthernet1/0/24
 no cdp enable
```

This is the preferred control when CDP should remain available on trusted infrastructure links but be disabled on selected untrusted ports.

---

## Adjust Timers

Change advertisement interval:

```cisco
cdp timer 30
```

Change holdtime:

```cisco
cdp holdtime 120
```

Timer changes are normally unnecessary unless there is a specific operational requirement.

---

## Verification

### Summary Neighbor View

```cisco
show cdp neighbors
```

Useful fields typically include:

```text
Device ID
Local interface
Holdtime
Capability
Platform
Remote Port ID
```

---

### Detailed Neighbor View

```cisco
show cdp neighbors detail
```

Use it to obtain additional information such as:

```text
Management address
Software version
Platform
Capabilities
Remote interface
Native VLAN / duplex information when advertised
```

---

### CDP Interface Status

```cisco
show cdp interface
```

Use it to verify which interfaces are participating in CDP and the active timers.

---

### Protocol Counters

```cisco
show cdp traffic
```

Use it to inspect CDP packet counters.

---

## Practical Troubleshooting Sequence

```text
Expected neighbor missing?
        ↓
Is CDP enabled globally?
        ↓
Is CDP enabled on both interfaces?
        ↓
Is the physical interface Up?
        ↓
Is the link directly connected?
        ↓
Are CDP frames being sent/received?
        ↓
Check show cdp traffic
        ↓
Validate platform/interface support
```

Useful commands:

```cisco
show cdp
show cdp neighbors
show cdp neighbors detail
show cdp interface
show cdp traffic
show interfaces status
show interfaces <interface>
```

---

## 5. Common Gotchas and Misconceptions

### CDP Requires IP Connectivity

**Incorrect.**

CDP operates at Layer 2.

Two directly connected devices can discover each other even when:

```text
No IP address exists
IP addressing is wrong
Routing is broken
```

---

### CDP Discovers Devices Multiple Hops Away

**Incorrect.**

CDP is a directly connected neighbor-discovery protocol.

```text
R1 ---- SW1 ---- SW2

R1 learns SW1
R1 does not learn SW2 through SW1
```

---

### A CDP Neighbor Proves the Network Path Is Working

**Incorrect.**

CDP confirms Layer 2 neighbor discovery only.

It does not prove:

```text
VLAN forwarding
IP reachability
Routing
ACL/firewall policy
Application connectivity
```

---

### CDP Is Standards-Based

**Incorrect.**

CDP is Cisco proprietary.

The standards-based alternative is:

```text
LLDP — IEEE 802.1AB
```

Use LLDP where multivendor interoperability is required.

---

### CDP Is Safe Everywhere Because It Is Only Informational

**Incorrect or Unsafe.**

CDP can reveal:

```text
Device names
Platforms
Interfaces
Software versions
Management addressing
Network role/capabilities
```

That information can help an attacker profile the network.

Disable CDP on untrusted links unless there is a defined operational requirement.

---

### CDP and LLDP Cannot Run Together

**Incorrect.**

Many Cisco platforms can run CDP and LLDP simultaneously.

Whether both are needed depends on endpoint and multivendor requirements.

---

### Disabling CDP on Every Access Port Is Always Best

**Too broad.**

For a normal user-only access port, disabling CDP may reduce unnecessary information exposure.

However, Cisco IP phones and other Cisco endpoints may rely on CDP for functions such as:

```text
Voice VLAN discovery
Power negotiation
Device-specific integration
```

Validate endpoint behavior before disabling it.

---

## 6. Trade-Offs

### Best Practice

- Keep CDP enabled on **trusted infrastructure links** where it materially improves operations.
- Disable CDP on **untrusted or externally connected interfaces** unless it is explicitly required.
- Use `show cdp neighbors detail` to validate physical topology rather than assuming documentation is current.
- Treat CDP information as operational evidence, not proof of Layer 3 reachability.
- Use LLDP when standards-based multivendor discovery is required.

---

### Context-Dependent Trade-Off — CDP vs LLDP

**CDP**

```text
+ Strong Cisco integration
+ Useful Cisco-specific TLVs
+ Helpful with Cisco phones and infrastructure
- Cisco proprietary
```

**LLDP**

```text
+ IEEE standard
+ Better multivendor interoperability
+ LLDP-MED supports endpoint/voice use cases
- May expose different/less vendor-specific information
```

Many enterprise designs enable both where multivendor discovery is useful.

---

### Context-Dependent Trade-Off — Discovery vs Information Exposure

```text
CDP enabled
+ Faster topology discovery
+ Easier troubleshooting
+ Better device visibility
- Exposes infrastructure information to directly connected peers
```

The correct choice depends on:

```text
Trust boundary
Endpoint type
Operational need
Security policy
```

---

### Incorrect or Unsafe

- Leaving CDP enabled on Internet-facing or untrusted links without a specific requirement.
- Assuming a CDP neighbor proves IP or application reachability.
- Globally disabling CDP without checking dependencies such as Cisco IP phones.
- Using CDP as a substitute for inventory, monitoring, or configuration-management systems.
- Assuming CDP and LLDP expose identical information or behave identically on every platform.

---

## Quick Reference

```text
CDP
= Cisco Discovery Protocol

Type
= Cisco proprietary

Layer
= Layer 2

Purpose
= Directly connected neighbor discovery

Requires IP?
= No

IPv4 / IPv6 routed?
= No

Ethernet Destination MAC
= 01:00:0C:CC:CC:CC

Default Advertisement
= 60 seconds

Default Holdtime
= 180 seconds

Common Information
= Device ID
= Platform
= Capabilities
= Local/remote interfaces
= Management address
= Software version
= Native VLAN / duplex when advertised

Global Enable
= cdp run

Interface Enable
= cdp enable

Summary
= show cdp neighbors

Detail
= show cdp neighbors detail

Interfaces
= show cdp interface

Counters
= show cdp traffic

Standards-Based Alternative
= LLDP

Core Rule
= CDP proves direct Layer 2 discovery, not Layer 3 connectivity
```

## CCNA Configuration

**IOS-XE — Global CDP Control**

| Command | Description |
|---|---|
| **Enable CDP globally:**<br>`(config)#cdp run` | Enables CDP globally; CDP is not explicit in CCNA v1.1. |
| **Disable CDP globally:**<br>`(config)#no cdp run` | Disables CDP globally on the device. |

**IOS-XE — Interface CDP Control**

| Command | Description |
|---|---|
| **Enable CDP on interface:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#cdp enable` | Enables CDP advertisements and processing on the interface. |
| **Disable CDP on interface:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no cdp enable` | Disables CDP advertisements and processing on the interface. |

**IOS-XE — CDP Timers**

| Command | Description |
|---|---|
| **Set advertisement interval:**<br>`(config)#cdp timer <5-254>` | Sets CDP advertisement transmission interval in seconds. |
| **Set holdtime:**<br>`(config)#cdp holdtime <10-255>` | Sets advertised CDP neighbor holdtime in seconds. |

**IOS-XE — CDP Verification**

| Command | Description |
|---|---|
| **Show global CDP status:**<br>`#show cdp` | Displays global CDP timers and advertisement version. |
| **Show CDP interfaces:**<br>`#show cdp interface [<interface-id>]` | Displays CDP status and timers per interface. |
| **Show CDP neighbors:**<br>`#show cdp neighbors [<interface-id>]` | Displays summarized directly connected CDP neighbors. |
| **Show detailed neighbors:**<br>`#show cdp neighbors [<interface-id>] detail` | Displays detailed information for discovered CDP neighbors. |
| **Show specific neighbor:**<br>`#show cdp entry <device-name>` | Displays detailed information for one named CDP neighbor. |
| **Filter neighbor details:**<br>`#show cdp entry <device-name> [protocol|version]` | Displays selected protocol or software-version neighbor details. |
| **Show CDP traffic counters:**<br>`#show cdp traffic` | Displays CDP transmit, receive, and error counters. |

## CCNP Configuration

**CCNP Security — IOS-XE CDPv2 Control**

| Command | Description |
|---|---|
| **Disable CDPv2 advertisements:**<br>`(config)#no cdp advertise-v2` | Disables CDPv2 advertisements; CDP is not explicit in SCOR v2.0. |
| **Enable CDPv2 advertisements:**<br>`(config)#cdp advertise-v2` | Enables CDPv2 advertisements globally. |

**CCNP Security — IOS-XE CDP Maintenance**

| Command | Description |
|---|---|
| **Clear CDP counters:**<br>`#clear cdp counters` | Resets global CDP traffic counters. |
| **Clear CDP neighbor table:**<br>`#clear cdp table` | Deletes learned CDP neighbor-table entries. |


</div>
