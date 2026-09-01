<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# Address Resolution Protocol (ARP)

## What is ARP?

**ARP (Address Resolution Protocol)** resolves an **IPv4 address to a Layer 2 MAC address** for a directly reachable neighbor on an Ethernet LAN.

ARP is a local neighbor-resolution mechanism: routing identifies the destination or next-hop IP address, and ARP supplies the destination MAC address required to build the Ethernet frame.

> **Routing determines the next-hop IP. ARP determines the next-hop MAC.**

---

## How it works

When an IPv4 device needs to transmit over Ethernet, it first determines whether the destination is **local or remote**.

### Local Destination

If `172.16.10.10/24` sends to `172.16.10.20`, the destination is on the same subnet. The sender checks its ARP table for:

```text
172.16.10.20 -> MAC address
```

If no entry exists, it sends an **ARP Request** as an Ethernet broadcast:

```text
Destination MAC: FFFF.FFFF.FFFF

Who has 172.16.10.20?
```

The request contains the sender's IP/MAC and the target IP. Every device in the local broadcast domain receives it, but the device owning the target IP should respond.

The target normally sends a **unicast ARP Reply** containing its IP/MAC mapping:

```text
172.16.10.20 -> BBBB.BBBB.BBBB
```

The requester stores the mapping in its ARP cache and can then construct the Ethernet frame. The receiver can also learn the sender's mapping from the sender fields in the ARP request.

```text
ARP Request -> Broadcast
ARP Reply   -> Normally unicast
```

ARP over Ethernet uses EtherType **0x0806**.

### Remote Destination

If the destination is outside the local subnet, the host does **not** ARP for the remote destination. It ARPs for the local **default gateway or next-hop router**.

Example:

```text
Host:        172.16.10.10/24
Gateway:     172.16.10.1
Destination: 8.8.8.8
```

The transmitted frame contains:

```text
Destination MAC = gateway MAC
Destination IP  = 8.8.8.8
```

ARP therefore resolves only devices reachable on the local Layer 2 segment; ARP broadcasts are not routed between broadcast domains.

At each routed Ethernet hop, the IP destination normally remains the final destination while the source/destination MAC addresses are rewritten for the new Layer 2 segment.

### ARP Cache

Learned mappings are cached:

```text
IP address       MAC address
172.16.10.1      0011.2233.4455
172.16.10.20     00aa.bbcc.ddee
```

Dynamic entries age out and are relearned when required. Conceptually, ARP resolution occurs in the control plane; the resulting neighbor/adjacency information is then used by the forwarding plane.

---

## Why and when it is used

ARP solves one specific problem:

> **An IPv4 device knows which local or next-hop IP address it must reach, but Ethernet requires a destination MAC address.**

ARP is therefore required when IPv4 traffic is forwarded over technologies where an IPv4-to-Layer-2 mapping is needed, most commonly Ethernet.

It is not used to resolve remote hosts directly, and **IPv6 does not use ARP**; IPv6 performs equivalent neighbor resolution with ICMPv6 Neighbor Discovery.

---

## Key Configuration, Parameters, or CLI

### Cisco IOS / IOS XE

ARP normally requires no explicit configuration.

View the ARP table:

```cisco
show ip arp
```

Filter for a specific address:

```cisco
show ip arp <ip-address>
```

Useful forwarding correlation:

```cisco
show ip route <destination>
show ip cef <destination> detail
show ip arp <next-hop>
```

On a Catalyst switch, correlate the resolved MAC with its Layer 2 location:

```cisco
show mac address-table address <mac-address>
```

Operational troubleshooting chain:

```text
Route > Next-hop IP > ARP: IP -> MAC > MAC table: MAC -> Switchport
```

---

## Common Gotchas and Misconceptions

**ARP does not resolve the final remote destination.** For remote traffic, ARP resolves the local next hop or default gateway.

**ARP table and MAC address table are different.**

```text
ARP table:         IP -> MAC
MAC address table: MAC -> switch port
```

**ARP is limited to the local broadcast domain.** If ARP resolution fails, check VLAN membership, Layer 2 forwarding, interface state, addressing/subnet masks, and security features before assuming a routing-protocol problem.

**Gratuitous ARP (GARP)** is an unsolicited ARP announcement used to update other devices' mappings, commonly during IP/MAC changes or high-availability events.

**Proxy ARP** is different from normal ARP: a router answers an ARP request on behalf of another IP destination and then routes the resulting traffic. It can be legitimate, but it can also conceal incorrect subnetting or routing assumptions.

**ARP spoofing/poisoning** introduces false IP-to-MAC mappings. Cisco Catalyst switches can mitigate this with **Dynamic ARP Inspection (DAI)**, commonly validating ARP messages against the DHCP Snooping binding database.

### IOS / IOS XE - DAI Example

```cisco
ip arp inspection vlan 10

interface GigabitEthernet1/0/24
 ip arp inspection trust
```

Verify:

```cisco
show ip arp inspection vlan 10
show ip arp inspection interfaces
```

**Incorrect or Unsafe:** enabling DAI without valid DHCP Snooping bindings, ARP ACLs for required static-address devices, and correct trusted-port design can cause legitimate ARP traffic to be dropped.

---

## Core Mental Model

<div align="center">
  <img
    src="Images/ARP/ARP Model.png"
    alt="ARP Model"
    width="600"
  />
</div>

**ARP is IPv4's local IP-to-MAC resolution mechanism. It allows a host or router to turn a directly reachable IPv4 next hop into the Layer 2 destination needed to transmit the Ethernet frame.**

## CCNA Configuration Cheatsheet

**CCNA 200-301 v2.0 — IOS-XE ARP Cache Verification**

| Command | Description |
|---|---|
| **Show ARP table:**<br>`#show ip arp` | Displays the IPv4 ARP cache. |
| **Show ARP table (alternate):**<br>`#show arp` | Displays the IPv4 ARP cache. |
| **Clear dynamic ARP entries:**<br>`#clear ip arp [<ip-address>]` | Clears all or one dynamically learned ARP entry. |

**CCNA 200-301 v2.0 — IOS-XE DHCP Snooping Prerequisite for DAI**

| Command | Description |
|---|---|
| **Enable DHCP snooping:**<br>`(config)#ip dhcp snooping` | Enables DHCP snooping globally. |
| **Enable DHCP snooping on VLANs:**<br>`(config)#ip dhcp snooping vlan <vlan-list>` | Enables DHCP snooping on specified VLANs. |
| **Trust DHCP snooping uplink:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip dhcp snooping trust` | Marks the interface trusted for DHCP snooping. |
| **Verify DHCP snooping bindings:**<br>`#show ip dhcp snooping binding` | Displays learned DHCP snooping IP-to-MAC bindings. |

**CCNA 200-301 v2.0 — IOS-XE Dynamic ARP Inspection**

| Command | Description |
|---|---|
| **Enable DAI on VLANs:**<br>`(config)#ip arp inspection vlan <vlan-range>` | Enables Dynamic ARP Inspection on specified VLANs. |
| **Trust DAI interface:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip arp inspection trust` | Marks the interface trusted for Dynamic ARP Inspection. |
| **Restore DAI untrusted state:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no ip arp inspection trust` | Restores the interface to the untrusted DAI state. |
| **Show DAI status:**<br>`#show ip arp inspection` | Displays DAI configuration, state, and counters. |
| **Show DAI interface state:**<br>`#show ip arp inspection interfaces [<interface-id>]` | Displays DAI trust state and interface rate limits. |
| **Show DAI VLAN state:**<br>`#show ip arp inspection vlan <vlan-range>` | Displays DAI state for specified VLANs. |
| **Show DAI statistics:**<br>`#show ip arp inspection statistics [vlan <vlan-range>]` | Displays forwarded, dropped, and validation counters. |

## CCNP Configuration Cheatsheet

**CCNP Security — SCOR 350-701 v1.1 — IOS-XE ARP ACLs for DAI**

| Command | Description |
|---|---|
| **Create ARP ACL:**<br>`(config)#arp access-list <acl-name>`<br>&nbsp;&nbsp;○ `(config-arp-nacl)#permit ip host <sender-ip> mac host <sender-mac>` | Creates an ARP ACL with a static IP-to-MAC binding. |
| **Permit any ARP binding:**<br>`(config-arp-nacl)#permit ip any mac any` | Permits any IP-to-MAC binding in the ARP ACL. |
| **Apply ARP ACL to VLANs:**<br>`(config)#ip arp inspection filter <arp-acl-name> vlan <vlan-range> [static]` | Applies the ARP ACL to specified DAI VLANs. |
| **Verify ARP ACL:**<br>`#show arp access-list [<acl-name>]` | Displays configured ARP access-list entries. |

**CCNP Security — SCOR 350-701 v1.1 — IOS-XE DAI Validation**

| Command | Description |
|---|---|
| **Enable additional DAI validation:**<br>`(config)#ip arp inspection validate {[src-mac] [dst-mac] [ip]}` | Enables selected MAC and IP ARP validation checks. |
| **Verify validation settings:**<br>`#show ip arp inspection` | Displays enabled source-MAC, destination-MAC, and IP validation. |

**CCNP Security — SCOR 350-701 v1.1 — IOS-XE DAI Rate Limiting**

| Command | Description |
|---|---|
| **Configure ARP rate limit:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip arp inspection limit {rate <pps> [burst interval <seconds>]\|none}` | Sets or removes the interface DAI packet-rate limit. |
| **Restore default rate limit:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no ip arp inspection limit` | Restores the interface default DAI rate limit. |
| **Enable ARP-inspection errdisable detection:**<br>`(config)#errdisable detect cause arp-inspection` | Enables errdisable detection for ARP inspection violations. |
| **Enable automatic recovery:**<br>`(config)#errdisable recovery cause arp-inspection` | Enables automatic recovery from ARP-inspection errdisable events. |
| **Set recovery interval:**<br>`(config)#errdisable recovery interval <seconds>` | Sets the global errdisable recovery timer. |
| **Verify rate limits:**<br>`#show ip arp inspection interfaces [<interface-id>]` | Displays configured DAI trust state and rate limits. |
| **Verify errdisable recovery:**<br>`#show errdisable recovery` | Displays enabled errdisable recovery causes and timers. |

**CCNP Security — SCOR 350-701 v1.1 — IOS-XE DAI Monitoring**

| Command | Description |
|---|---|
| **Show DAI statistics:**<br>`#show ip arp inspection statistics [vlan <vlan-range>]` | Displays DAI forwarding, drop, and validation statistics. |
| **Clear DAI statistics:**<br>`#clear ip arp inspection statistics` | Clears Dynamic ARP Inspection statistics. |
| **Show DAI log:**<br>`#show ip arp inspection log` | Displays the Dynamic ARP Inspection log buffer. |
| **Clear DAI log:**<br>`#clear ip arp inspection log` | Clears the Dynamic ARP Inspection log buffer. |

**CCNP Enterprise — ENCOR 350-401 v1.1 — IOS-XE Proxy ARP Hardening**

| Command | Description |
|---|---|
| **Disable Proxy ARP:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no ip proxy-arp` | Disables Proxy ARP on the selected interface. |
| **Re-enable Proxy ARP:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip proxy-arp` | Enables Proxy ARP on the selected interface. |


</div>
