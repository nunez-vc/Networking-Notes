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
