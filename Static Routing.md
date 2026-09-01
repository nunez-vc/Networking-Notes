# Static Routing

> **Core idea:** A static route is a manually configured path to a destination prefix. It gives deterministic control over forwarding, but it does not automatically learn or adapt to topology changes unless additional tracking mechanisms are used.

---

## 1. What It Is

**Static routing** is the manual installation of routes that tell a router how to reach specific destination prefixes. Unlike dynamic routing protocols, static routes do not exchange routing information with neighbors; they rely entirely on administrator-defined next hops, exit interfaces, or both.

On Cisco IOS/IOS XE, a normal IPv4 static route has an administrative distance of **1** by default. fileciteturn26file2

---

## 2. How It Works

### Basic Forwarding Logic

A static route provides three pieces of information:

```text
Destination prefix
        ↓
Next hop / exit interface
        ↓
Forwarding path
```

Example:

```text
Destination: 10.20.20.0/24
Next hop:    192.0.2.2
```

When a packet arrives:

```text
Packet destination
       ↓
Routing-table lookup
       ↓
Longest-prefix match
       ↓
Static route selected
       ↓
Resolve next hop / adjacency
       ↓
Forward packet
```

Static routing participates in the same routing-table and forwarding logic as dynamically learned routes. The router still uses **longest-prefix match first**; administrative distance matters when competing routes exist for the same prefix.

---

### Route Installation

A configured static route is useful only if the router can resolve its forwarding information.

Conceptually:

```text
Static route configured
        ↓
Is exit path / next hop resolvable?
       / \
     Yes  No
      |    |
Install   Do not install
in RIB    usable route
      |
      ↓
Program forwarding table / adjacency
```

A static route can disappear from the routing table if its required exit interface or recursive next-hop resolution is no longer valid.

However, static routing has no inherent awareness of failures beyond the information used to resolve the route. A remote failure can therefore leave a static route installed even though the destination is no longer reachable.

---

## Static Route Types

Cisco commonly describes three forwarding forms for IPv4 static routes. fileciteturn26file0

### Recursive Static Route

Specifies only the next-hop IP address:

```cisco
ip route 10.20.20.0 255.255.255.0 192.0.2.2
```

The router performs a recursive lookup to determine how to reach `192.0.2.2`.

```text
10.20.20.0/24
      ↓
Next hop 192.0.2.2
      ↓
Routing-table lookup for 192.0.2.2
      ↓
Exit interface / adjacency
```

This is commonly appropriate on Ethernet and other multiaccess networks.

---

### Directly Attached Static Route

Specifies only the exit interface:

```cisco
ip route 10.20.20.0 255.255.255.0 Serial0/0/0
```

This is most appropriate on true **point-to-point links**, where there is only one possible device at the far end. Cisco specifically identifies this form as appropriate for point-to-point interfaces. fileciteturn26file1

Using only an Ethernet exit interface can cause the router to treat remote destinations as directly reachable on that segment and may trigger ARP behavior for destination addresses. Avoid this form on multiaccess Ethernet unless the design specifically requires it.

---

### Fully Specified Static Route

Specifies both the exit interface and the next-hop address:

```cisco
ip route 10.20.20.0 255.255.255.0 GigabitEthernet0/0 192.0.2.2
```

This explicitly identifies both:

```text
Where to send it → GigabitEthernet0/0
Who to send it to → 192.0.2.2
```

Cisco documents this form as a **fully specified static route**. fileciteturn26file3

---

## Default Route

A default route matches any destination for which no more-specific route exists.

### IPv4

```text
0.0.0.0/0
```

```cisco
ip route 0.0.0.0 0.0.0.0 192.0.2.2
```

Typical use:

```text
Branch / edge router
        ↓
Unknown destination
        ↓
Default route
        ↓
Upstream router / firewall / ISP
```

Because routing uses longest-prefix match, any more-specific route overrides the default route.

---

## Host Route

A host route matches one specific IP address.

### IPv4

```text
/32
```

Example:

```cisco
ip route 10.20.20.50 255.255.255.255 192.0.2.2
```

A host route is more specific than any broader network route that contains the same address.

---

## Floating Static Route

A **floating static route** is a backup static route configured with an administrative distance higher than the preferred route.

Example: primary route learned through OSPF:

```text
OSPF AD = 110
```

Backup static route:

```cisco
ip route 10.20.20.0 255.255.255.0 192.0.2.6 200
```

Normal state:

```text
OSPF route
AD 110
  ↓
Installed

Static route
AD 200
  ↓
Not preferred
```

If the OSPF route is withdrawn:

```text
OSPF route removed
       ↓
Floating static becomes best available route
       ↓
Installed in the RIB
```

Cisco specifically describes floating static routes as static routes with a higher AD used for backup connectivity. fileciteturn26file2

> The backup route must have an AD higher than the primary route it is intended to back up.

---

## Static Route to Null0

A static route can intentionally discard traffic:

```cisco
ip route 10.20.0.0 255.255.0.0 Null0
```

Traffic matching the prefix is dropped unless a more-specific route exists.

Common purpose:

```text
Summary prefix
      ↓
More-specific route exists?
     / \
   Yes  No
    |    |
Forward Drop at Null0
```

This is useful for preventing routing loops or safely terminating traffic that matches an aggregate but not an actual reachable subnet. Cisco demonstrates Null0 static routing specifically for loop prevention. fileciteturn26file15

---

## Equal-Cost Static Routes

Multiple static routes to the same prefix can provide equal-cost paths when their route preference is equivalent and the platform supports installing multiple paths.

Example:

```cisco
ip route 10.20.20.0 255.255.255.0 192.0.2.2
ip route 10.20.20.0 255.255.255.0 198.51.100.2
```

The forwarding behavior then depends on the platform's ECMP implementation and CEF load-sharing behavior.

---

## IPv6 Static Routing

The same static-routing principles apply to IPv6. Cisco IOS/IOS XE requires IPv6 forwarding to be enabled:

```cisco
ipv6 unicast-routing
```

Example:

```cisco
ipv6 route 2001:db8:20::/64 2001:db8:12::2
```

Default IPv6 route:

```cisco
ipv6 route ::/0 2001:db8:12::2
```

Host route:

```cisco
ipv6 route 2001:db8:20::50/128 2001:db8:12::2
```

If the next hop is an **IPv6 link-local address**, the exit interface must also be specified because a link-local address is meaningful only on its local link. Cisco explicitly requires a fully specified static route in this case. fileciteturn26file12

Example:

```cisco
ipv6 route 2001:db8:20::/64 GigabitEthernet0/0 fe80::2
```

---

## 3. Why and When It Is Used

Static routing is appropriate when routes are:

- **Simple and stable**
- Required to follow a **specific deterministic path**
- Used for a **default route** toward an upstream router or firewall
- Used as a **backup** to dynamic routing
- Used to reach **stub networks**
- Used to intentionally **discard traffic** with Null0
- Needed where running a dynamic routing protocol is unnecessary or undesirable

Static routes consume no bandwidth for routing-protocol advertisements because no routing updates are exchanged. Cisco also notes that they provide precise control but create increasing administrative burden as the network grows. fileciteturn26file0

Static routing is generally unsuitable as the primary routing method in large or frequently changing topologies because routes must be maintained manually and do not inherently react to remote topology changes.

---

## 4. Key Configuration, Parameters, or CLI

### Cisco IOS / IOS XE — IPv4

General syntax:

```cisco
ip route <destination> <mask> <next-hop | exit-interface> [administrative-distance]
```

### Recursive

```cisco
ip route 10.20.20.0 255.255.255.0 192.0.2.2
```

### Fully Specified

```cisco
ip route 10.20.20.0 255.255.255.0 GigabitEthernet0/0 192.0.2.2
```

### Default

```cisco
ip route 0.0.0.0 0.0.0.0 192.0.2.2
```

### Host

```cisco
ip route 10.20.20.50 255.255.255.255 192.0.2.2
```

### Floating

```cisco
ip route 10.20.20.0 255.255.255.0 192.0.2.6 200
```

### Null0

```cisco
ip route 10.20.0.0 255.255.0.0 Null0
```

---

### Cisco IOS / IOS XE — IPv6

```cisco
ipv6 unicast-routing

ipv6 route 2001:db8:20::/64 2001:db8:12::2
ipv6 route ::/0 2001:db8:12::2
```

Using a link-local next hop:

```cisco
ipv6 route 2001:db8:20::/64 GigabitEthernet0/0 fe80::2
```

---

### Verification

```cisco
show ip route
show ip route static
show ip route <destination>
show ipv6 route
show ipv6 route static
show ip cef <destination>
ping <destination>
traceroute <destination>
```

Static routes appear with route code:

```text
S = Static
```

A useful troubleshooting sequence is:

```text
1. Is the static route configured correctly?
        ↓
2. Is the next hop reachable / exit interface up?
        ↓
3. Is the route installed in the RIB?
        ↓
4. Is a more-specific route winning?
        ↓
5. Is CEF forwarding toward the expected adjacency?
        ↓
6. Is there a valid return route?
```

---

## 5. Common Gotchas and Misconceptions

### Static Routes Always Override Dynamic Routes

**Incorrect.**

The router first compares **prefix length**.

```text
More-specific route wins
before AD is considered
```

Administrative distance is relevant when competing routes describe the same prefix.

A static route with the default AD of `1` normally beats OSPF (`110`) or EIGRP internal (`90`) for the **same prefix**.

---

### A Configured Static Route Must Appear in the Routing Table

**Incorrect.**

The route may fail to install if its next hop cannot be resolved or the required interface is unavailable.

Verify:

```cisco
show running-config | include ^ip route
show ip route <destination>
show ip route <next-hop>
```

---

### An Installed Static Route Proves the Destination Is Reachable

**Incorrect.**

A static route can remain installed as long as its local next-hop dependency remains valid, even if a failure exists farther downstream.

```text
Router → next hop works
          ↓
Remote path beyond next hop fails
          ↓
Static route may remain installed
```

For failover that depends on end-to-end reachability, use a platform-supported tracking mechanism such as object tracking/IP SLA when appropriate.

---

### Exit-Interface-Only Routes Are Fine on Any Ethernet Link

**Incorrect or risky.**

On point-to-point links, an exit-interface-only route is unambiguous. On multiaccess Ethernet, it can cause undesirable ARP behavior because multiple possible neighbors share the segment.

Prefer a **next-hop IP** or **fully specified route** on Ethernet unless the topology specifically justifies otherwise.

---

### Floating Static Means the Route Is Always Installed

**Incorrect.**

The route is configured, but normally remains out of the active routing table while a better route with lower AD exists.

---

### One-Way Static Routing Is Enough

**Incorrect.**

Forward and return paths both need valid routing.

```text
A → B works
B → A missing route
      ↓
Session fails
```

---

### Default Route Overrides Specific Routes

**Incorrect.**

```text
0.0.0.0/0
```

is the **least specific** IPv4 route. Any matching more-specific prefix is preferred.

---

## 6. Trade-Offs

### Best Practice

- Use static routes where the topology is simple, stable, and intentionally deterministic.
- Prefer next-hop or fully specified routes on Ethernet multiaccess links.
- Use floating static routes with an AD deliberately higher than the primary route.
- Verify both forward and return paths.
- Use Null0 routes intentionally and only for prefixes that should be discarded when no more-specific route exists.

---

### Context-Dependent Trade-Off

**Recursive vs Fully Specified**

```text
Recursive
+ Simpler
+ Adapts to changes in how the next hop is reached
- Requires recursive resolution

Fully specified
+ Explicit next hop and exit interface
+ Useful on multiaccess links
- More tightly coupled to interface topology
```

**Static vs Dynamic Routing**

```text
Static
+ Deterministic
+ No routing-protocol overhead
+ Simple at small scale
- Manual maintenance
- Limited failure awareness

Dynamic
+ Learns topology
+ Adapts to failures
+ Better scale
- Adds protocol complexity and control-plane state
```

**Default Route**

Excellent for stub/edge designs, but it intentionally hides detailed knowledge of external destinations. Whether that is desirable depends on the topology.

---

### Incorrect or Unsafe

- Using an exit-interface-only static route on a multiaccess Ethernet segment without understanding the ARP consequences
- Configuring a floating static route with an AD lower than the intended primary dynamic route
- Installing broad static/default routes without validating where unmatched traffic will go
- Assuming static routing automatically detects failures beyond the configured next-hop dependency
- Changing route AD in production without evaluating interactions with dynamic routing and redistribution

---

## Quick Reference

```text
Static Route
= Manually configured route

Default IPv4 AD
= 1

Recursive
= Destination + next-hop IP

Directly Attached
= Destination + exit interface
  Best suited to point-to-point links

Fully Specified
= Destination + exit interface + next hop

Default Route
= 0.0.0.0/0
= ::/0

Host Route
= /32 IPv4
= /128 IPv6

Floating Static
= Backup static route with higher AD

Null0 Static
= Intentional discard / loop prevention

IPv6 Link-Local Next Hop
= Must include exit interface

Route Selection
= Longest prefix first
  Then route-source preference / AD

Static Routing
= Deterministic but manually maintained
```

## CCNA Configuration

**IOS-XE — IPv4 Static Network Routes**

| Command | Description |
|---|---|
| **Configure next-hop route:**<br>`(config)#ip route <network> <subnet-mask> <next-hop-ip>` | Creates a recursive IPv4 static network route. |
| **Configure exit-interface route:**<br>`(config)#ip route <network> <subnet-mask> <outgoing-interface>` | Creates a directly attached IPv4 static route. |
| **Remove next-hop route:**<br>`(config)#no ip route <network> <subnet-mask> <next-hop-ip>` | Removes the specified recursive IPv4 static route. |
| **Remove exit-interface route:**<br>`(config)#no ip route <network> <subnet-mask> <outgoing-interface>` | Removes the specified directly attached static route. |

**IOS-XE — IPv4 Default Route**

| Command | Description |
|---|---|
| **Configure default route:**<br>`(config)#ip route 0.0.0.0 0.0.0.0 <next-hop-ip>` | Creates an IPv4 static default route. |
| **Configure interface default route:**<br>`(config)#ip route 0.0.0.0 0.0.0.0 <outgoing-interface>` | Creates an interface-based IPv4 default route. |
| **Remove default route:**<br>`(config)#no ip route 0.0.0.0 0.0.0.0 <next-hop-ip>` | Removes the specified IPv4 static default route. |

**IOS-XE — IPv4 Host Route**

| Command | Description |
|---|---|
| **Configure host route:**<br>`(config)#ip route <host-ip> 255.255.255.255 <next-hop-ip>` | Creates an IPv4 static route to one host. |
| **Configure interface host route:**<br>`(config)#ip route <host-ip> 255.255.255.255 <outgoing-interface>` | Creates an interface-based IPv4 host route. |

**IOS-XE — IPv4 Floating Static Route**

| Command | Description |
|---|---|
| **Configure floating route:**<br>`(config)#ip route <network> <subnet-mask> <next-hop-ip> <administrative-distance>` | Creates a static route with modified administrative distance. |
| **Configure interface floating route:**<br>`(config)#ip route <network> <subnet-mask> <outgoing-interface> <administrative-distance>` | Creates an interface static route with modified distance. |
| **Configure floating default:**<br>`(config)#ip route 0.0.0.0 0.0.0.0 <next-hop-ip> <administrative-distance>` | Creates a floating IPv4 static default route. |

**IOS-XE — IPv4 Static Route Verification**

| Command | Description |
|---|---|
| **Show static routes:**<br>`#show ip route static` | Displays installed IPv4 static routes. |
| **Show specific route:**<br>`#show ip route <destination>` | Displays detailed routing information for the destination. |
| **Show routing table:**<br>`#show ip route` | Displays the complete IPv4 routing table. |

**IOS-XE — IPv6 Static Network Routes**

| Command | Description |
|---|---|
| **Configure GUA next-hop route:**<br>`(config)#ipv6 route <prefix>/<prefix-length> <next-hop-gua>` | Creates an IPv6 static route using global next hop. |
| **Configure fully specified GUA route:**<br>`(config)#ipv6 route <prefix>/<prefix-length> <outgoing-interface> <next-hop-gua>` | Creates an IPv6 route using interface and global next hop. |
| **Configure link-local next-hop route:**<br>`(config)#ipv6 route <prefix>/<prefix-length> <outgoing-interface> <next-hop-lla>` | Creates an IPv6 route using a link-local next hop. |
| **Remove GUA next-hop route:**<br>`(config)#no ipv6 route <prefix>/<prefix-length> <next-hop-gua>` | Removes the specified IPv6 static route. |

**IOS-XE — IPv6 Default Route**

| Command | Description |
|---|---|
| **Configure GUA default route:**<br>`(config)#ipv6 route ::/0 <next-hop-gua>` | Creates an IPv6 default route using global next hop. |
| **Configure link-local default route:**<br>`(config)#ipv6 route ::/0 <outgoing-interface> <next-hop-lla>` | Creates an IPv6 default route using link-local next hop. |

**IOS-XE — IPv6 Host Route**

| Command | Description |
|---|---|
| **Configure host route:**<br>`(config)#ipv6 route <host-ipv6>/128 <next-hop-gua>` | Creates an IPv6 static route to one host. |
| **Configure link-local host route:**<br>`(config)#ipv6 route <host-ipv6>/128 <outgoing-interface> <next-hop-lla>` | Creates an IPv6 host route through link-local next hop. |

**IOS-XE — IPv6 Floating Static Route**

| Command | Description |
|---|---|
| **Configure floating route:**<br>`(config)#ipv6 route <prefix>/<prefix-length> <next-hop-gua> <administrative-distance>` | Creates an IPv6 static route with modified distance. |
| **Configure interface floating route:**<br>`(config)#ipv6 route <prefix>/<prefix-length> <outgoing-interface> <administrative-distance>` | Creates an interface IPv6 route with modified distance. |
| **Configure floating default:**<br>`(config)#ipv6 route ::/0 <next-hop-gua> <administrative-distance>` | Creates a floating IPv6 static default route. |

**IOS-XE — IPv6 Static Route Verification**

| Command | Description |
|---|---|
| **Show static routes:**<br>`#show ipv6 route static` | Displays installed IPv6 static routes. |
| **Show specific prefix:**<br>`#show ipv6 route <prefix>/<prefix-length>` | Displays detailed routing information for one IPv6 prefix. |
| **Show destination lookup:**<br>`#show ipv6 route <ipv6-address>` | Displays the route used for the specified destination. |
| **Show static route details:**<br>`#show ipv6 static detail` | Displays configured IPv6 static routes and installation state. |
| **Show routing table:**<br>`#show ipv6 route` | Displays the complete IPv6 routing table. |

## CCNP Configuration

**CCNP Enterprise — IOS-XE — Fully Specified IPv4 Static Routes**

| Command | Description |
|---|---|
| **Configure fully specified route:**<br>`(config)#ip route <network> <subnet-mask> <outgoing-interface> <next-hop-ip>` | Creates a route specifying interface and next-hop address. |
| **Configure fully specified default:**<br>`(config)#ip route 0.0.0.0 0.0.0.0 <outgoing-interface> <next-hop-ip>` | Creates a fully specified IPv4 default route. |
| **Remove fully specified route:**<br>`(config)#no ip route <network> <subnet-mask> <outgoing-interface> <next-hop-ip>` | Removes the specified fully specified static route. |

**CCNP Enterprise — IOS-XE — Static Null Routes and Route Tags**

| Command | Description |
|---|---|
| **Configure Null0 route:**<br>`(config)#ip route <network> <subnet-mask> Null0` | Creates a discard static route through Null0. |
| **Configure tagged static route:**<br>`(config)#ip route <network> <subnet-mask> <next-hop-ip> tag <tag-value>` | Assigns a route tag to the static route. |
| **Configure permanent route:**<br>`(config)#ip route <network> <subnet-mask> <next-hop-ip> permanent` | Keeps the static route installed despite next-hop resolution loss. |

**CCNP Enterprise — IOS-XE — IPv4 VRF Static Routing**

| Command | Description |
|---|---|
| **Configure VRF static route:**<br>`(config)#ip route vrf <vrf-name> <network> <subnet-mask> <next-hop-ip>` | Creates an IPv4 static route inside the specified VRF. |
| **Remove VRF static route:**<br>`(config)#no ip route vrf <vrf-name> <network> <subnet-mask> <next-hop-ip>` | Removes the specified IPv4 VRF static route. |
| **Show VRF routing table:**<br>`#show ip route vrf <vrf-name>` | Displays the IPv4 routing table for one VRF. |

**CCNP Enterprise — IOS-XE — IPv6 VRF Static Routing**

| Command | Description |
|---|---|
| **Configure VRF IPv6 route:**<br>`(config)#ipv6 route vrf <vrf-name> <prefix>/<prefix-length> <next-hop-ipv6>` | Creates an IPv6 static route inside the specified VRF. |
| **Configure tracked VRF IPv6 route:**<br>`(config)#ipv6 route vrf <vrf-name> <prefix>/<prefix-length> <outgoing-interface> <next-hop-ipv6> track <object-id>` | Associates an IPv6 VRF static route with tracking. |
| **Show VRF IPv6 routes:**<br>`#show ipv6 route vrf <vrf-name>` | Displays the IPv6 routing table for one VRF. |

**CCNP Enterprise — IOS-XE — IP SLA Tracked Static Route**

| Command | Description |
|---|---|
| **Create IP SLA operation:**<br>`(config)#ip sla <operation-number>` | Enters configuration mode for one IP SLA operation. |
| **Configure ICMP echo probe:**<br>`(config-ip-sla)#icmp-echo <destination-ip> [source-interface <interface-id>]` | Configures an ICMP echo reachability probe. |
| **Set probe frequency:**<br>`(config-ip-sla-echo)#frequency <seconds>` | Sets the interval between IP SLA probes. |
| **Schedule SLA continuously:**<br>`(config)#ip sla schedule <operation-number> life forever start-time now` | Starts the IP SLA operation immediately and continuously. |
| **Create tracking object:**<br>`(config)#track <object-id> ip sla <operation-number> reachability` | Tracks reachability reported by the IP SLA operation. |
| **Delay track transitions:**<br>`(config-track)#delay up <seconds> down <seconds>` | Delays tracked-object up and down state notifications. |
| **Track static route:**<br>`(config)#ip route <network> <subnet-mask> <next-hop-ip> track <object-id>` | Installs the static route while the tracking object is up. |
| **Show SLA configuration:**<br>`#show ip sla configuration` | Displays configured IP SLA operations and parameters. |
| **Show SLA statistics:**<br>`#show ip sla statistics [<operation-number>]` | Displays current operational status and SLA statistics. |
| **Show tracking state:**<br>`#show track [<object-id>]` | Displays tracked-object state and associated SLA status. |

**CCNP Security — ASA 9.x — IPv4 Static Routing**

| Command | Description |
|---|---|
| **Configure static route:**<br>`(config)#route <interface-name> <destination-ip> <netmask> <gateway-ip>` | Creates an ASA IPv4 static route. |
| **Set administrative distance:**<br>`(config)#route <interface-name> <destination-ip> <netmask> <gateway-ip> <distance>` | Creates an ASA static route with custom distance. |
| **Configure default route:**<br>`(config)#route <interface-name> 0.0.0.0 0.0.0.0 <gateway-ip>` | Creates an ASA IPv4 static default route. |
| **Configure floating default:**<br>`(config)#route <interface-name> 0.0.0.0 0.0.0.0 <gateway-ip> <distance>` | Creates an ASA floating static default route. |
| **Remove static route:**<br>`(config)#no route <interface-name> <destination-ip> <netmask> <gateway-ip>` | Removes the specified ASA IPv4 static route. |
| **Show routing table:**<br>`#show route` | Displays the ASA routing table. |

**CCNP Security — ASA 9.x — IPv6 Static Routing**

| Command | Description |
|---|---|
| **Configure IPv6 static route:**<br>`(config)#ipv6 route <interface-name> <prefix>/<prefix-length> <gateway-ipv6>` | Creates an ASA IPv6 static route. |
| **Set IPv6 route distance:**<br>`(config)#ipv6 route <interface-name> <prefix>/<prefix-length> <gateway-ipv6> <distance>` | Creates an ASA IPv6 route with custom distance. |
| **Configure IPv6 default route:**<br>`(config)#ipv6 route <interface-name> ::/0 <gateway-ipv6>` | Creates an ASA IPv6 static default route. |
| **Configure floating IPv6 default:**<br>`(config)#ipv6 route <interface-name> ::/0 <gateway-ipv6> <distance>` | Creates an ASA floating IPv6 default route. |

**CCNP Security — ASA 9.x — Tracked Static Routing**

| Command | Description |
|---|---|
| **Create SLA monitor:**<br>`(config)#sla monitor <sla-id>` | Enters ASA SLA monitoring configuration mode. |
| **Configure ICMP echo:**<br>`(config-sla-monitor)#type echo protocol ipicmpecho <target-ip> interface <interface-name>` | Configures ICMP reachability monitoring through the selected interface. |
| **Set probe frequency:**<br>`(config-sla-monitor-echo)#frequency <seconds>` | Sets the interval between ASA SLA probes. |
| **Schedule SLA monitor:**<br>`(config)#sla monitor schedule <sla-id> life forever start-time now` | Starts the ASA SLA monitor immediately and continuously. |
| **Create tracking object:**<br>`(config)#track <track-id> rtr <sla-id> reachability` | Associates an ASA tracking object with SLA reachability. |
| **Track static route:**<br>`(config)#route <interface-name> <destination-ip> <netmask> <gateway-ip> [<distance>] track <track-id>` | Associates the ASA static route with the tracking object. |
| **Configure backup route:**<br>`(config)#route <backup-interface> <destination-ip> <netmask> <backup-gateway-ip> <higher-distance>` | Creates a higher-distance backup static route. |
| **Show SLA configuration:**<br>`#show sla monitor configuration [<sla-id>]` | Displays ASA SLA monitor configuration and defaults. |
| **Show SLA state:**<br>`#show sla monitor operational-state [<sla-id>]` | Displays ASA SLA monitor operational statistics. |
| **Show tracking state:**<br>`#show track [<track-id>]` | Displays ASA tracked-object reachability state. |

