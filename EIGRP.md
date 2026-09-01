# Enhanced Interior Gateway Routing Protocol (EIGRP)

> **Core idea:** EIGRP is an advanced distance-vector IGP that uses **DUAL** to select loop-free best paths and, when available, precomputed loop-free backup paths.

---

## 1. What It Is

**EIGRP (Enhanced Interior Gateway Routing Protocol)** is an interior gateway routing protocol used to exchange routes within an administrative routing domain. It uses the **Diffusing Update Algorithm (DUAL)**, a topology table, and a composite metric to select and maintain loop-free paths.

EIGRP uses **IP protocol 88**. For IPv4, it normally uses multicast address **224.0.0.10** for neighbor discovery and protocol communication when multicast is appropriate.

Default Cisco administrative distances:

| Route Type | Administrative Distance |
|---|---:|
| EIGRP summary | 5 |
| Internal EIGRP | 90 |
| External EIGRP | 170 |

> An EIGRP autonomous-system number identifies an EIGRP routing domain. It is **not the same concept as a BGP autonomous system**, even though both use the term *AS*.

---

## 2. How It Works

### Neighbor Formation

EIGRP-enabled routers discover each other with **Hello** packets and build a neighbor table.

Neighbors must use compatible EIGRP parameters, including:

- The same EIGRP autonomous system
- Compatible metric/K-value settings
- Compatible authentication when authentication is configured
- Valid Layer 3 connectivity on the shared link

Unlike OSPF, **EIGRP Hello and Hold timers do not have to match**. A router advertises its Hold time to its neighbor.

Typical defaults on common high-speed links are:

```text
Hello interval: 5 seconds
Hold time:      15 seconds
```

The Hold timer determines how long a neighbor is considered reachable without receiving another Hello.

---

### Route Exchange

When an adjacency first forms, neighbors exchange routing information. After convergence, EIGRP normally sends **partial, triggered updates** instead of periodically sending the entire routing table.

EIGRP uses five packet types:

| Packet | Purpose |
|---|---|
| **Hello** | Discover and maintain neighbors |
| **Update** | Advertise routing information |
| **Query** | Ask neighbors for an alternate path |
| **Reply** | Respond to a Query |
| **Request** | Request specific routing information |

Updates, Queries, and Replies use reliable delivery and acknowledgments when required.

---

### EIGRP Tables

EIGRP maintains three operational views:

```text
Neighbor Table
     ↓
Tracks EIGRP adjacencies

Topology Table
     ↓
Stores learned paths and their metrics

Routing Table
     ↓
Contains the EIGRP paths selected for forwarding
```

The topology table is central to DUAL because it retains alternate paths rather than keeping only the installed route.

---

### DUAL Path Selection

The key DUAL terms are:

| Term | Meaning |
|---|---|
| **Successor** | Next-hop router on the best path |
| **Successor Route** | Lowest-metric path to the destination |
| **Feasible Distance (FD)** | Local metric for the best path |
| **Reported Distance (RD)** | Metric a neighbor reports for its own path to the destination |
| **Feasible Successor (FS)** | Loop-free backup path that satisfies the Feasibility Condition |

The **Feasibility Condition** is:

```text
Neighbor RD < Local FD
```

If true, the alternate path is guaranteed loop-free from DUAL's perspective and can be retained as a **Feasible Successor**.

Example:

```text
Current Successor FD = 3000
Alternate Neighbor RD = 2000

2000 < 3000
```

The alternate path satisfies the Feasibility Condition.

> A valid alternate route does not automatically become a Feasible Successor. It must satisfy the Feasibility Condition.

---

### Passive and Active States

An EIGRP prefix is normally:

```text
Passive = Stable; no route computation is in progress
```

**Passive is the normal state.**

If the Successor fails:

1. If a Feasible Successor exists, DUAL can immediately promote it to Successor.
2. If no Feasible Successor exists, the prefix becomes **Active**.
3. The router sends Queries to appropriate EIGRP neighbors.
4. Neighbors return Replies or propagate the Query when necessary.
5. After all required Replies are received, DUAL selects the new path and returns the prefix to **Passive**.

```text
Successor fails
      ↓
Feasible Successor available?
     / \
   Yes  No
    |    |
Promote  Route becomes Active
    |    |
Update   Send Queries
         |
         v
      Replies
         |
         v
     Run DUAL
         |
         v
      Passive
```

A route that remains Active because required responses are not received can lead to a **Stuck-in-Active (SIA)** condition. Large query domains, poor connectivity, overloaded routers, or unstable links can contribute to this problem.

---

### EIGRP Metric

With the default classic metric K-values:

```text
K1 = 1
K2 = 0
K3 = 1
K4 = 0
K5 = 0
```

EIGRP uses:

- **Minimum bandwidth** along the path
- **Cumulative delay** along the path

The classic default metric can be represented as:

```text
Metric =
256 × [(10^7 / Minimum Bandwidth in kbps)
       + (Total Delay in microseconds / 10)]
```

Bandwidth is based on the **lowest-bandwidth link** in the path; delay is accumulated across the path.

Load, reliability, MTU, and hop count may appear in EIGRP route information, but **load and reliability are not used in the default metric**, and **MTU and hop count are not components of the default composite metric**.

Modern EIGRP also supports **wide metrics**, which provide greater precision for high-speed interfaces.

---

### Equal- and Unequal-Cost Paths

EIGRP supports normal equal-cost multipath routing.

It can also install eligible **unequal-cost** paths using `variance`.

For an alternate path to be installed with variance, it must:

1. Be loop-free and satisfy the Feasibility Condition.
2. Have a metric within the configured variance threshold.

Conceptually:

```text
Alternate Metric < Successor Metric × Variance
```

`variance` does **not** make a non-feasible path eligible.

---

### Query Control and Scaling

EIGRP convergence becomes more predictable when Query propagation is bounded.

Two important mechanisms are:

**EIGRP Stub**

A stub router tells its neighbors not to use it as a transit path for arbitrary EIGRP Queries. This is particularly useful for branch, spoke, and leaf routers.

**Route Summarization**

Summarization reduces routing-table size and can create a Query boundary, reducing the scope of DUAL recomputation.

---

## 3. Why and When It Is Used

EIGRP is appropriate when the routing domain requires:

- Fast convergence
- Efficient triggered updates
- Precomputed loop-free backup paths
- Simple operation in Cisco-centric enterprise networks
- Equal- or unequal-cost multipath routing
- Effective scaling of hub-and-spoke or branch topologies using Stub routing and summarization

EIGRP is **not an Internet interdomain routing protocol**; BGP is used for that role.

Do not assume EIGRP interoperability in a multivendor environment simply because its protocol specification is publicly documented. Confirm implementation support on every participating platform.

---

## 4. Key Configuration, Parameters, or CLI

### Cisco IOS / IOS XE — Classic EIGRP

Basic IPv4 configuration:

```cisco
router eigrp 100
 eigrp router-id 1.1.1.1
 network 10.0.0.0 0.255.255.255
 passive-interface default
 no passive-interface GigabitEthernet0/0
```

`network` enables EIGRP on matching interfaces and advertises the connected prefixes associated with those interfaces.

Using `passive-interface default` prevents unintended neighbor formation, while `no passive-interface` explicitly enables EIGRP adjacency formation on required transit interfaces.

---

### Cisco IOS XE — Named EIGRP

Named mode organizes IPv4/IPv6 and interface-specific EIGRP configuration under address-family hierarchy.

```cisco
router eigrp CORE
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 1.1.1.1
  network 10.0.0.0 0.255.255.255
  af-interface default
   passive-interface
  exit-af-interface
  af-interface GigabitEthernet0/0
   no passive-interface
  exit-af-interface
 exit-address-family
```

Named-mode syntax and supported features can vary by IOS XE release and platform. Verify against the applicable Cisco IOS XE configuration and command reference before production deployment.

---

### EIGRP Stub — IOS / IOS XE Classic Mode

For a branch or leaf router:

```cisco
router eigrp 100
 eigrp stub
```

The default Stub behavior advertises connected and summary routes. Additional Stub options should be configured only when the topology requires them.

---

### Unequal-Cost Multipath — IOS / IOS XE Classic Mode

```cisco
router eigrp 100
 variance 2
```

A higher variance does not bypass the Feasibility Condition.

---

### Verification — IOS / IOS XE

```cisco
show ip eigrp neighbors
show ip eigrp interfaces
show ip eigrp topology
show ip route eigrp
show ip protocols
```

Use them in this order when troubleshooting:

```text
EIGRP enabled on interface?
          ↓
Neighbor established?
          ↓
Prefix present in topology table?
          ↓
Successor / Feasible Successor correct?
          ↓
Route installed in the RIB?
```

---

## 5. Common Gotchas and Misconceptions

### Hello and Hold Timers Must Match

**Incorrect.** EIGRP neighbors can form with different Hello/Hold timers because the Hold time is advertised to the neighbor.

---

### Passive Means the Route Is Down

**Incorrect.**

```text
Passive = Normal/stable
Active  = DUAL is actively searching for a path
```

An unexpected long-running Active state is the condition to investigate.

---

### Every Backup Path Is a Feasible Successor

**Incorrect.** The alternate route must satisfy:

```text
RD < FD
```

A valid route that fails this test can still be discovered through a DUAL Query process after the Successor fails, but it is not a precomputed Feasible Successor.

---

### `variance` Allows Any Unequal-Cost Route

**Incorrect.** The path must satisfy both the Feasibility Condition and the configured variance threshold.

---

### EIGRP Uses Bandwidth Alone

**Incorrect.** By default, EIGRP uses **minimum bandwidth and cumulative delay**.

---

### Changing Interface Bandwidth Is Always the Best Way to Tune EIGRP

**Incorrect or Unsafe.** The interface `bandwidth` value may also influence other routing protocols, QoS features, and management calculations.

When the goal is specifically to influence EIGRP path selection, adjusting **interface delay** is often the cleaner choice because it avoids changing the bandwidth value consumed by other features.

---

### K-Values Can Be Changed Independently

**Incorrect or Unsafe.** Neighbors require compatible metric K-values. Inconsistent K-values can prevent adjacency formation.

Changing the default metric weights should be treated as a routing-domain-wide design decision, not a local tuning shortcut.

---

## 6. Trade-Offs

### Best Practice

- Use `passive-interface default` and explicitly enable EIGRP only on intended neighbor-facing links.
- Use **EIGRP Stub** on true branch/spoke/leaf routers that should not provide transit.
- Use summarization where the addressing design supports it to reduce route scale and Query scope.
- Prefer **delay** over artificial bandwidth changes when tuning only EIGRP path preference.
- Verify the topology table before changing metrics or enabling `variance`.

---

### Context-Dependent Trade-Off

**Unequal-cost load balancing (`variance`)**

Useful when multiple loop-free WAN paths should carry traffic, but it introduces asymmetric utilization and requires capacity and application-flow consideration.

**Route summarization**

Improves scale and limits Queries, but hides specific prefixes. Poorly placed summaries can create suboptimal routing or black holes if reachability underneath the summary is not understood.

**Authentication**

Recommended where neighbor trust must be enforced, especially on less-trusted shared segments. Exact authentication algorithms and syntax depend on EIGRP mode, IOS/IOS XE release, and platform.

---

### Incorrect or Unsafe

- Changing K-values on only part of an EIGRP routing domain
- Using `variance` without validating the Feasibility Condition
- Disabling split horizon without a topology-specific requirement
- Making a router an EIGRP Stub when it must provide transit connectivity
- Changing administrative distance or metrics without evaluating redistribution, failover, and loop behavior

---

## Quick Reference

```text
EIGRP
= Advanced distance-vector IGP

IP Protocol
= 88

IPv4 Multicast
= 224.0.0.10

Algorithm
= DUAL

Successor
= Best next hop

Feasible Distance
= Local best-path metric

Reported Distance
= Neighbor's advertised metric

Feasible Successor
= Precomputed loop-free backup

Feasibility Condition
= RD < FD

Passive
= Stable

Active
= Searching for a new path

Default Metric Inputs
= Minimum bandwidth + cumulative delay

Internal AD
= 90

External AD
= 170

Stub + Summarization
= Reduce Query scope and improve scalability

Variance
= Unequal-cost multipath for eligible feasible paths
```

## CCNA Configuration

EIGRP configuration is outside current CCNA 200-301 v1.1 scope.

## CCNP Configuration

**CCNP Enterprise — ENARSI 300-410 v1.1 — IOS-XE Classic EIGRP IPv4**

| Command | Description |
|---|---|
| **Start EIGRP process:**<br>`(config)#router eigrp <as-number>` | Enters the classic IPv4 EIGRP routing process. |
| **Set router ID:**<br>`(config-router)#eigrp router-id <ipv4-address>` | Sets the EIGRP router identifier. |
| **Enable EIGRP on interfaces:**<br>`(config-router)#network <ipv4-network> <wildcard-mask>` | Matches IPv4 interfaces for EIGRP participation. |
| **Match one interface address:**<br>`(config-router)#network <interface-ip> 0.0.0.0` | Enables EIGRP only on the matching interface address. |
| **Make all interfaces passive:**<br>`(config-router)#passive-interface default` | Suppresses EIGRP adjacencies on all matched interfaces. |
| **Enable one active interface:**<br>`(config-router)#no passive-interface <interface-id>` | Re-enables EIGRP adjacency formation on one interface. |
| **Configure EIGRP stub:**<br>`(config-router)#eigrp stub connected summary` | Advertises connected and summary routes as an EIGRP stub. |
| **Configure receive-only stub:**<br>`(config-router)#eigrp stub receive-only` | Configures the EIGRP stub to receive routes only. |

**CCNP Enterprise — ENARSI 300-410 v1.1 — IOS-XE Classic EIGRP Interface Tuning**

| Command | Description |
|---|---|
| **Set hello interval:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip hello-interval eigrp <as-number> <seconds>` | Sets the EIGRP hello interval on the interface. |
| **Set hold time:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip hold-time eigrp <as-number> <seconds>` | Sets the EIGRP neighbor hold time. |
| **Set EIGRP bandwidth percentage:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip bandwidth-percent eigrp <as-number> <percentage>` | Limits EIGRP traffic relative to interface bandwidth. |
| **Disable split horizon:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no ip split-horizon eigrp <as-number>` | Disables EIGRP split horizon on the interface. |
| **Disable next-hop self:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no ip next-hop-self eigrp <as-number>` | Preserves the received EIGRP next-hop address. |
| **Set metric bandwidth:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#bandwidth <kilobits>` | Sets interface bandwidth used by EIGRP metric calculations. |
| **Set metric delay:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#delay <tens-of-microseconds>` | Sets interface delay used by EIGRP metric calculations. |

**CCNP Enterprise — ENARSI 300-410 v1.1 — IOS-XE Classic EIGRP Load Balancing and Metrics**

| Command | Description |
|---|---|
| **Set maximum paths:**<br>`(config-router)#maximum-paths <number-of-paths>` | Sets maximum equal or unequal EIGRP paths installed. |
| **Enable unequal-cost paths:**<br>`(config-router)#variance <multiplier>` | Sets the EIGRP unequal-cost load-balancing variance. |
| **Use balanced traffic sharing:**<br>`(config-router)#traffic-share balanced` | Distributes traffic proportionally across installed EIGRP paths. |
| **Set classic K-values:**<br>`(config-router)#metric weights 0 <k1> <k2> <k3> <k4> <k5>` | Configures classic EIGRP metric weighting constants. |
| **Add inbound metric offset:**<br>`(config-router)#offset-list <acl-number-or-name> in <offset> [<interface-id>]` | Adds an offset to matching inbound EIGRP metrics. |
| **Add outbound metric offset:**<br>`(config-router)#offset-list <acl-number-or-name> out <offset> [<interface-id>]` | Adds an offset to matching outbound EIGRP metrics. |

**CCNP Enterprise — ENARSI 300-410 v1.1 — IOS-XE Classic EIGRP Summarization**

| Command | Description |
|---|---|
| **Configure interface summary:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip summary-address eigrp <as-number> <summary-address> <subnet-mask>` | Advertises an IPv4 EIGRP summary from the interface. |
| **Configure summary leak-map:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip summary-address eigrp <as-number> <summary-address> <subnet-mask> leak-map <route-map-name>` | Leaks selected component routes through the interface summary. |

**CCNP Enterprise — ENARSI 300-410 v1.1 — IOS-XE Named EIGRP IPv4**

| Command | Description |
|---|---|
| **Create named EIGRP process:**<br>`(config)#router eigrp <instance-name>` | Enters named EIGRP configuration mode. |
| **Create IPv4 address family:**<br>`(config-router)#address-family ipv4 unicast autonomous-system <as-number>` | Creates the global IPv4 EIGRP address family. |
| **Create IPv4 VRF address family:**<br>`(config-router)#address-family ipv4 unicast vrf <vrf-name> autonomous-system <as-number>` | Creates an IPv4 EIGRP address family for a VRF. |
| **Start address family:**<br>`(config-router-af)#no shutdown` | Starts the named EIGRP address-family process. |
| **Set router ID:**<br>`(config-router-af)#eigrp router-id <ipv4-address>` | Sets the named EIGRP router identifier. |
| **Enable EIGRP on interfaces:**<br>`(config-router-af)#network <ipv4-network> <wildcard-mask>` | Matches IPv4 interfaces for named EIGRP. |
| **Configure EIGRP stub:**<br>`(config-router-af)#eigrp stub connected summary` | Advertises connected and summary routes as a stub. |
| **Set named K-values:**<br>`(config-router-af)#metric weights 0 <k1> <k2> <k3> <k4> <k5> <k6>` | Configures wide-metric EIGRP weighting constants. |

**CCNP Enterprise — ENARSI 300-410 v1.1 — IOS-XE Named EIGRP Interface Tuning**

| Command | Description |
|---|---|
| **Set default passive state:**<br>`(config-router-af)#af-interface default`<br>&nbsp;&nbsp;○ `(config-router-af-interface)#passive-interface` | Makes all address-family interfaces passive by default. |
| **Enable adjacency on interface:**<br>`(config-router-af)#af-interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-router-af-interface)#no passive-interface` | Enables neighbor formation on the selected interface. |
| **Set hello interval:**<br>`(config-router-af)#af-interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-router-af-interface)#hello-interval <seconds>` | Sets the named EIGRP hello interval. |
| **Set hold time:**<br>`(config-router-af)#af-interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-router-af-interface)#hold-time <seconds>` | Sets the named EIGRP neighbor hold time. |
| **Set bandwidth percentage:**<br>`(config-router-af)#af-interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-router-af-interface)#bandwidth-percent <percentage>` | Limits EIGRP traffic relative to interface bandwidth. |
| **Disable split horizon:**<br>`(config-router-af)#af-interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-router-af-interface)#no split-horizon` | Disables EIGRP split horizon on the selected interface. |
| **Disable next-hop self:**<br>`(config-router-af)#af-interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-router-af-interface)#no next-hop-self` | Preserves the received EIGRP next-hop address. |
| **Disable next-hop self for ECMP:**<br>`(config-router-af)#af-interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-router-af-interface)#no next-hop-self no-ecmp-mode` | Preserves next hops while evaluating equal-cost paths. |
| **Configure named summary:**<br>`(config-router-af)#af-interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-router-af-interface)#summary-address <summary-address> <subnet-mask>` | Advertises an EIGRP summary from the selected interface. |
| **Set summary distance:**<br>`(config-router-af-interface)#summary-address <summary-address> <subnet-mask> <administrative-distance>` | Sets administrative distance for the named summary route. |
| **Configure summary leak-map:**<br>`(config-router-af-interface)#summary-address <summary-address> <subnet-mask> <administrative-distance> leak-map <route-map-name>` | Leaks selected component routes through the named summary. |

**CCNP Enterprise — ENARSI 300-410 v1.1 — IOS-XE Named EIGRP Topology Tuning**

| Command | Description |
|---|---|
| **Enter base topology:**<br>`(config-router-af)#topology base` | Enters named EIGRP base-topology configuration mode. |
| **Set maximum paths:**<br>`(config-router-af-topology)#maximum-paths <number-of-paths>` | Sets maximum EIGRP paths installed for the topology. |
| **Enable unequal-cost paths:**<br>`(config-router-af-topology)#variance <multiplier>` | Sets unequal-cost load-balancing variance for the topology. |
| **Use balanced traffic sharing:**<br>`(config-router-af-topology)#traffic-share balanced` | Distributes traffic proportionally across installed EIGRP paths. |
| **Add inbound metric offset:**<br>`(config-router-af-topology)#offset-list <acl-number-or-name> in <offset> [<interface-id>]` | Adds an offset to matching inbound EIGRP metrics. |
| **Add outbound metric offset:**<br>`(config-router-af-topology)#offset-list <acl-number-or-name> out <offset> [<interface-id>]` | Adds an offset to matching outbound EIGRP metrics. |

**CCNP Enterprise — ENARSI 300-410 v1.1 — IOS-XE Classic EIGRP IPv6**

| Command | Description |
|---|---|
| **Enable IPv6 routing:**<br>`(config)#ipv6 unicast-routing` | Enables IPv6 packet forwarding globally. |
| **Start IPv6 EIGRP process:**<br>`(config)#ipv6 router eigrp <as-number>` | Enters classic IPv6 EIGRP router configuration mode. |
| **Set router ID:**<br>`(config-router)#eigrp router-id <ipv4-address>` | Sets the IPv6 EIGRP router identifier. |
| **Start IPv6 EIGRP:**<br>`(config-router)#no shutdown` | Starts the classic IPv6 EIGRP process. |
| **Enable EIGRP on interface:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ipv6 eigrp <as-number>` | Enables classic IPv6 EIGRP on the interface. |
| **Set IPv6 hello interval:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ipv6 hello-interval eigrp <as-number> <seconds>` | Sets the IPv6 EIGRP hello interval. |
| **Set IPv6 hold time:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ipv6 hold-time eigrp <as-number> <seconds>` | Sets the IPv6 EIGRP neighbor hold time. |
| **Disable IPv6 split horizon:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no ipv6 split-horizon eigrp <as-number>` | Disables IPv6 EIGRP split horizon. |
| **Disable IPv6 next-hop self:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no ipv6 next-hop-self eigrp <as-number>` | Preserves the received IPv6 EIGRP next hop. |
| **Configure IPv6 EIGRP stub:**<br>`(config-router)#eigrp stub connected summary` | Advertises connected and summary routes as an IPv6 stub. |

**CCNP Enterprise — ENARSI 300-410 v1.1 — IOS-XE Named EIGRP IPv6**

| Command | Description |
|---|---|
| **Enable IPv6 routing:**<br>`(config)#ipv6 unicast-routing` | Enables IPv6 packet forwarding globally. |
| **Create named EIGRP process:**<br>`(config)#router eigrp <instance-name>` | Enters named EIGRP configuration mode. |
| **Create IPv6 address family:**<br>`(config-router)#address-family ipv6 unicast autonomous-system <as-number>` | Creates the global IPv6 EIGRP address family. |
| **Create IPv6 VRF address family:**<br>`(config-router)#address-family ipv6 unicast vrf <vrf-name> autonomous-system <as-number>` | Creates an IPv6 EIGRP address family for a VRF. |
| **Start address family:**<br>`(config-router-af)#no shutdown` | Starts the named IPv6 EIGRP address family. |
| **Set router ID:**<br>`(config-router-af)#eigrp router-id <ipv4-address>` | Sets the named IPv6 EIGRP router identifier. |
| **Disable all AF interfaces:**<br>`(config-router-af)#af-interface default`<br>&nbsp;&nbsp;○ `(config-router-af-interface)#shutdown` | Disables EIGRP on all IPv6 interfaces by default. |
| **Enable one AF interface:**<br>`(config-router-af)#af-interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-router-af-interface)#no shutdown` | Enables named IPv6 EIGRP on one interface. |
| **Configure IPv6 EIGRP stub:**<br>`(config-router-af)#eigrp stub connected summary` | Advertises connected and summary routes as an IPv6 stub. |

**CCNP Enterprise / Security — ENARSI 300-410 v1.1 / SCOR 350-701 v1.1 — IOS-XE EIGRP Authentication**

| Command | Description |
|---|---|
| **Create key chain:**<br>`(config)#key chain <key-chain-name>` | Creates an EIGRP authentication key chain. |
| **Create key:**<br>`(config-keychain)#key <key-id>`<br>&nbsp;&nbsp;○ `(config-keychain-key)#key-string <key-string>` | Creates a key and assigns its secret string. |
| **Set key acceptance lifetime:**<br>`(config-keychain-key)#accept-lifetime <start-time> infinite` | Sets when the authentication key is accepted. |
| **Set key sending lifetime:**<br>`(config-keychain-key)#send-lifetime <start-time> infinite` | Sets when the authentication key is transmitted. |
| **Enable classic MD5:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip authentication mode eigrp <as-number> md5` | Enables MD5 authentication for classic IPv4 EIGRP. |
| **Apply classic key chain:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip authentication key-chain eigrp <as-number> <key-chain-name>` | Applies the key chain to classic IPv4 EIGRP. |
| **Enable named MD5:**<br>`(config-router-af)#af-interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-router-af-interface)#authentication mode md5` | Enables MD5 authentication for named EIGRP. |
| **Apply named key chain:**<br>`(config-router-af)#af-interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-router-af-interface)#authentication key-chain <key-chain-name>` | Applies the key chain to named EIGRP. |
| **Enable named HMAC-SHA-256:**<br>`(config-router-af)#af-interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-router-af-interface)#authentication mode hmac-sha-256 <encryption-type> <password>` | Enables HMAC-SHA-256 authentication for named EIGRP. |

**CCNP Enterprise — ENARSI 300-410 v1.1 — IOS-XE Classic EIGRP Verification**

| Command | Description |
|---|---|
| **Show neighbors:**<br>`#show ip eigrp neighbors` | Displays classic IPv4 EIGRP neighbor adjacencies. |
| **Show EIGRP interfaces:**<br>`#show ip eigrp interfaces` | Displays interfaces participating in IPv4 EIGRP. |
| **Show detailed interfaces:**<br>`#show ip eigrp interfaces detail` | Displays detailed IPv4 EIGRP interface parameters. |
| **Show topology:**<br>`#show ip eigrp topology` | Displays the IPv4 EIGRP topology table. |
| **Show active routes:**<br>`#show ip eigrp topology active` | Displays EIGRP routes currently in active state. |
| **Show all topology paths:**<br>`#show ip eigrp topology all-links` | Displays successor and non-successor topology paths. |
| **Show EIGRP traffic:**<br>`#show ip eigrp traffic` | Displays IPv4 EIGRP packet counters. |
| **Show EIGRP routes:**<br>`#show ip route eigrp` | Displays EIGRP-installed IPv4 routing-table entries. |
| **Show routing protocol state:**<br>`#show ip protocols` | Displays IPv4 EIGRP process and routing parameters. |

**CCNP Enterprise — ENARSI 300-410 v1.1 — IOS-XE Named EIGRP Verification**

| Command | Description |
|---|---|
| **Show IPv4 neighbors:**<br>`#show eigrp address-family ipv4 <as-number> neighbors` | Displays named IPv4 EIGRP neighbors. |
| **Show IPv4 VRF neighbors:**<br>`#show eigrp address-family ipv4 vrf <vrf-name> <as-number> neighbors` | Displays named IPv4 EIGRP neighbors in a VRF. |
| **Show IPv4 interfaces:**<br>`#show eigrp address-family ipv4 <as-number> interfaces` | Displays named IPv4 EIGRP interfaces. |
| **Show IPv4 topology:**<br>`#show eigrp address-family ipv4 <as-number> topology` | Displays named IPv4 EIGRP topology entries. |
| **Show IPv4 active routes:**<br>`#show eigrp address-family ipv4 <as-number> topology active` | Displays named IPv4 routes currently active. |
| **Show IPv4 traffic:**<br>`#show eigrp address-family ipv4 <as-number> traffic` | Displays named IPv4 EIGRP packet counters. |
| **Show IPv6 neighbors:**<br>`#show eigrp address-family ipv6 <as-number> neighbors` | Displays named IPv6 EIGRP neighbors. |
| **Show IPv6 VRF neighbors:**<br>`#show eigrp address-family ipv6 vrf <vrf-name> <as-number> neighbors` | Displays named IPv6 EIGRP neighbors in a VRF. |
| **Show IPv6 interfaces:**<br>`#show eigrp address-family ipv6 <as-number> interfaces` | Displays named IPv6 EIGRP interfaces. |
| **Show IPv6 topology:**<br>`#show eigrp address-family ipv6 <as-number> topology` | Displays named IPv6 EIGRP topology entries. |
| **Show IPv6 traffic:**<br>`#show eigrp address-family ipv6 <as-number> traffic` | Displays named IPv6 EIGRP packet counters. |
| **Show named processes:**<br>`#show eigrp protocols` | Displays configured named EIGRP protocol instances. |

**CCNP Enterprise — ENARSI 300-410 v1.1 — IOS-XE Classic EIGRP IPv6 Verification**

| Command | Description |
|---|---|
| **Show IPv6 neighbors:**<br>`#show ipv6 eigrp <as-number> neighbors` | Displays classic IPv6 EIGRP neighbor adjacencies. |
| **Show IPv6 interfaces:**<br>`#show ipv6 eigrp <as-number> interfaces` | Displays interfaces participating in classic IPv6 EIGRP. |
| **Show IPv6 interface detail:**<br>`#show ipv6 eigrp <as-number> interfaces detail` | Displays detailed classic IPv6 EIGRP interface parameters. |
| **Show IPv6 topology:**<br>`#show ipv6 eigrp <as-number> topology` | Displays the classic IPv6 EIGRP topology table. |
| **Show IPv6 active routes:**<br>`#show ipv6 eigrp <as-number> topology active` | Displays classic IPv6 routes currently active. |
| **Show IPv6 traffic:**<br>`#show ipv6 eigrp <as-number> traffic` | Displays classic IPv6 EIGRP packet counters. |
| **Show IPv6 routes:**<br>`#show ipv6 route eigrp` | Displays EIGRP-installed IPv6 routing-table entries. |

