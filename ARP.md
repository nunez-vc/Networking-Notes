<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# Address Resolution Protocol (ARP)

## 1. What is ARP?

**ARP (Address Resolution Protocol)** resolves an **IPv4 address to a Layer 2 MAC address** for a directly reachable neighbor on an Ethernet LAN.

ARP is a local neighbor-resolution mechanism: routing identifies the destination or next-hop IP address, and ARP supplies the destination MAC address required to build the Ethernet frame.

> **Routing determines the next-hop IP. ARP determines the next-hop MAC.**

---

## 2. How it works

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

## 3. Why and When it is used

ARP solves one specific problem:

> **An IPv4 device knows which local or next-hop IP address it must reach, but Ethernet requires a destination MAC address.**

ARP is therefore required when IPv4 traffic is forwarded over technologies where an IPv4-to-Layer-2 mapping is needed, most commonly Ethernet.

It is not used to resolve remote hosts directly, and **IPv6 does not use ARP**; IPv6 performs equivalent neighbor resolution with ICMPv6 Neighbor Discovery.

---

## 4. Key Configuration, Parameters, or CLI

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
Route
  |
  v
Next-hop IP
  |
  v
ARP: IP -> MAC
  |
  v
MAC table: MAC -> switch port
```

---

## 5. Common Gotchas and Misconceptions

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

```text
Local destination:
Destination IP
     |
     v
ARP
     |
     v
Destination MAC


Remote destination:
Destination IP
     |
     v
Routing
     |
     v
Next-hop IP
     |
     v
ARP
     |
     v
Next-hop MAC
```

**ARP is IPv4's local IP-to-MAC resolution mechanism. It allows a host or router to turn a directly reachable IPv4 next hop into the Layer 2 destination needed to transmit the Ethernet frame.**

</div>
