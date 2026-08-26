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
