<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# Border Gateway Protocol (BGP)

> **Core idea:** BGP is a **path-vector routing protocol** used to exchange reachability between autonomous systems and to apply routing policy at scale. It selects paths using attributes such as **LOCAL_PREF, AS_PATH, MED, and NEXT_HOP**, then installs the best eligible route into the routing table.

---

## 1. What It Is

**BGP (Border Gateway Protocol)** is a policy-driven path-vector routing protocol used for interdomain routing and large-scale route exchange. It runs over **TCP port 179**, exchanges prefixes plus path attributes, and supports both **eBGP** between different autonomous systems and **iBGP** within the same autonomous system.

```text
eBGP
= Different autonomous systems

iBGP
= Same autonomous system
```

Cisco IOS/IOS XE default administrative distances:

```text
eBGP = 20
iBGP = 200
```

> BGP is designed for **policy, scale, and control**, not simply for finding the numerically shortest path.

---

## 2. How It Works

## BGP Peering

BGP peers are explicitly configured.

```text
R1 AS 65001 ---------------- R2 AS 65002
             eBGP
```

The peers first need IP reachability to each other's configured neighbor addresses.

BGP then establishes a TCP session:

```text
Source TCP port = ephemeral
Destination TCP = 179
```

Once TCP is established, BGP negotiates the session and exchanges routing information.

---

## BGP Finite State Machine

The main BGP states are:

```text
Idle
  ↓
Connect
  ↓
Active
  ↓
OpenSent
  ↓
OpenConfirm
  ↓
Established
```

### Established

```text
Established
= BGP session is operational
= UPDATE messages can be exchanged
```

> In BGP, **Active is not the desired steady state**. A peer repeatedly stuck in Active normally indicates a TCP reachability, neighbor configuration, authentication, or transport problem.

---

## BGP Message Types

### OPEN

Starts the BGP session and exchanges parameters such as:

```text
BGP version
Autonomous system
Hold time
BGP router ID
Capabilities
```

---

### KEEPALIVE

Maintains the session when no UPDATE traffic is being sent.

Common Cisco defaults:

```text
Keepalive = 60 seconds
Hold time = 180 seconds
```

The negotiated hold time is based on the values exchanged by the peers.

---

### UPDATE

Advertises or withdraws reachable prefixes.

An UPDATE can carry:

```text
NLRI / prefixes
Path attributes
Withdrawn routes
```

---

### NOTIFICATION

Reports a BGP error and closes the session.

---

### ROUTE-REFRESH

Requests that a peer resend routes so inbound policy can be reevaluated without tearing down the BGP session, when the capability is supported and negotiated.

---

# BGP Route Information

A BGP route is more than a prefix.

Conceptually:

```text
Prefix
+
NEXT_HOP
+
AS_PATH
+
LOCAL_PREF
+
MED
+
ORIGIN
+
Communities
+
Other attributes
```

BGP uses these attributes to apply policy and select the best path.

---

## NEXT_HOP

The NEXT_HOP attribute identifies the Layer 3 next hop used to reach the advertised prefix.

### eBGP

When advertising a route to an eBGP peer, a router normally changes NEXT_HOP to itself.

```text
AS 65001                 AS 65002

R1 ---------------------- R2

R1 advertises:
203.0.113.0/24
NEXT_HOP = R1
```

---

### iBGP

When advertising an eBGP-learned route to an iBGP peer, BGP normally **preserves the original NEXT_HOP**.

```text
ISP
 |
R1 -------- iBGP -------- R2
```

R2 may learn the route but fail to use it if it cannot reach the advertised next hop.

A common solution is:

```cisco
neighbor <peer> next-hop-self
```

> **BGP reachability depends on NEXT_HOP reachability.** A prefix can exist in the BGP table but remain unusable if its next hop cannot be resolved.

---

## AS_PATH

AS_PATH records the autonomous systems a route has traversed.

Example:

```text
65010 65020 65030
```

When a route crosses an eBGP boundary, the advertising router normally prepends its own AS number.

```text
Prefix originates in AS 65030

Advertised to AS 65020:
65030

Advertised to AS 65010:
65020 65030
```

### Loop Prevention

A router normally rejects an eBGP route if its own AS already appears in the AS_PATH.

```text
Local AS found in AS_PATH
        ↓
Reject route
```

This is a fundamental BGP loop-prevention mechanism.

---

## LOCAL_PREF

LOCAL_PREF tells routers **inside one AS** which exit path is preferred.

```text
Higher LOCAL_PREF
= More preferred
```

Example:

```text
ISP-A
  |
R1 ───────── Enterprise AS ───────── R2
                                   |
                                  ISP-B
```

If routes learned through ISP-A have:

```text
LOCAL_PREF 200
```

and ISP-B:

```text
LOCAL_PREF 100
```

the AS normally prefers ISP-A for outbound traffic.

LOCAL_PREF is propagated through iBGP inside the AS and is not normally advertised to external eBGP neighbors.

---

## MED

**MED (Multi-Exit Discriminator)** suggests which entry point another AS should prefer when multiple links exist between the same autonomous systems.

```text
Lower MED
= More preferred
```

Example:

```text
        AS 65001
       /        \
     R1          R2
      \          /
       \        /
        AS 65002
```

AS 65001 can advertise different MED values to influence which link AS 65002 prefers for traffic entering AS 65001.

> MED behavior is policy-dependent, and by default Cisco commonly compares MED only under specific neighboring-AS conditions. Do not treat MED as a universal Internet-wide preference mechanism.

---

## WEIGHT — Cisco-Specific

Cisco IOS/IOS XE supports a local **Weight** attribute.

```text
Higher Weight
= More preferred
```

Weight:

```text
Exists only on the local Cisco router
Is not advertised to BGP peers
```

It is useful for local path preference but does not communicate policy to the rest of the AS.

---

## Communities

BGP communities are tags attached to routes so policy can be applied without matching individual prefixes repeatedly.

Conceptually:

```text
Prefix
+
Community 65001:100
        ↓
Route-map matches community
        ↓
Apply policy
```

Communities are widely used for:

```text
Routing policy
Provider signaling
Traffic engineering
Route classification
```

Their meaning is defined by the network or provider policy.

---

# BGP Best-Path Selection

BGP compares candidate paths for the **same prefix** and chooses one best path unless multipath is deliberately configured.

A simplified Cisco decision sequence includes:

```text
1. Highest Weight
2. Highest LOCAL_PREF
3. Prefer locally originated route
4. Shortest AS_PATH
5. Lowest ORIGIN type
6. Lowest MED when comparable
7. Prefer eBGP over iBGP
8. Lowest IGP metric to NEXT_HOP
9. Additional tie-breakers
```

Important notes:

- **Weight is Cisco-specific.**
- MED comparison has conditions and can be changed by configuration.
- Cisco uses additional tie-breakers after the items shown above.
- Multipath behavior requires specific configuration and path compatibility.

> BGP best-path selection and IP forwarding are different decisions. After routes are installed in the RIB/FIB, normal forwarding still uses **longest-prefix match**.

---

# eBGP vs iBGP

| Behavior | eBGP | iBGP |
|---|---|---|
| Peer AS | Different | Same |
| Default AD on IOS/IOS XE | 20 | 200 |
| AS_PATH | Local AS is prepended | Normally unchanged |
| NEXT_HOP | Normally changed to self | Normally preserved |
| Default direct-connect expectation | Yes | No |
| Route propagation rule | Advertise according to policy | iBGP-learned routes are not normally advertised to other iBGP peers |

---

## eBGP Direct-Connect Behavior

A normal eBGP session expects the peer to be directly connected.

The default IP TTL for eBGP is normally:

```text
1
```

If the peers use loopbacks or are separated by intermediate routers, additional configuration is required, such as:

```cisco
neighbor <peer> ebgp-multihop <ttl>
```

and usually:

```cisco
neighbor <peer> update-source Loopback0
```

The underlay must route between the loopback addresses.

---

## iBGP Split-Horizon Rule

A route learned from one iBGP peer is **not normally advertised to another iBGP peer**.

```text
R1 -------- R2 -------- R3
     iBGP        iBGP

Route learned by R2 from R1
        X
Not automatically sent to R3
```

This prevents routing loops inside the AS because iBGP does not add the local AS to AS_PATH.

---

## iBGP Scaling

### Full Mesh

Without another mechanism, every iBGP router that must exchange BGP routes directly requires a full mesh.

For `n` routers:

```text
Sessions = n(n-1) / 2
```

This does not scale well.

---

### Route Reflectors

A **Route Reflector (RR)** relaxes the normal iBGP advertisement rule.

```text
         RR
       /    \
 Client1   Client2
```

The RR can reflect iBGP-learned routes between clients according to route-reflector rules.

This reduces the number of required iBGP sessions.

> Route reflection improves scale but can hide alternate paths and create path-selection behavior that differs from a full mesh. The RR topology must be designed deliberately.

---

# Route Advertisement

## `network` Statement

On Cisco IOS/IOS XE, the BGP `network` statement does **not create a route**.

It originates a prefix into BGP only when a matching route exists in the local routing table.

Example:

```cisco
network 203.0.113.0 mask 255.255.255.0
```

requires the RIB to contain the matching prefix:

```text
203.0.113.0/24
```

> The BGP `network` command means **originate this existing route into BGP**, not "enable BGP on this interface."

---

## Redistribution

Routes from another source can also be redistributed into BGP.

This must be controlled carefully because broad redistribution can inject large or unintended route sets.

Use:

```text
Prefix filters
Route maps
Explicit policy
```

rather than uncontrolled redistribution.

---

# Route Policy

BGP is fundamentally policy-driven.

Common Cisco policy tools include:

```text
Prefix lists
AS-path access lists
Community lists
Route maps
Neighbor route policies
```

Typical policy actions:

```text
Permit / deny prefixes
Set LOCAL_PREF
Set MED
Set communities
Prepend AS_PATH
Set Weight
Control advertisements
```

---

## Prefix Filtering

Example IOS/IOS XE prefix list:

```cisco
ip prefix-list CUSTOMER-IN seq 10 permit 203.0.113.0/24
```

Apply inbound:

```cisco
router bgp 65001
 address-family ipv4 unicast
  neighbor 192.0.2.2 prefix-list CUSTOMER-IN in
```

Filtering should explicitly define what prefixes a peer is allowed to advertise.

---

## Maximum Prefix Protection

A BGP peer can be limited to an expected maximum number of prefixes.

Example:

```cisco
neighbor 192.0.2.2 maximum-prefix 1000
```

Exact syntax/options and shutdown/restart behavior vary by IOS/IOS XE release.

This helps protect against accidental route leaks or unexpectedly large advertisements.

---

# BGP and the Routing Table

BGP maintains its own table of candidate paths.

```text
Neighbor UPDATE
      ↓
Inbound policy
      ↓
BGP table
      ↓
Best-path selection
      ↓
Best BGP route
      ↓
RIB comparison
      ↓
FIB / forwarding
```

A route can therefore appear in:

```cisco
show ip bgp
```

but not in:

```cisco
show ip route
```

Possible reasons include:

```text
Unreachable NEXT_HOP
Another route source is preferred
The BGP path is not best
Policy prevents installation/use
```

---

## 3. Why and When It Is Used

BGP is appropriate when routing decisions require **policy, scale, or autonomous-system boundaries**.

Typical use cases:

```text
Internet routing
Dual-ISP / multihoming
Enterprise edge routing
Service-provider networks
Large data-center fabrics
MPLS VPN control planes
EVPN control planes
Large SD-WAN/cloud interconnect designs
```

Use BGP when:

- Multiple external routing policies must be controlled.
- An organization owns or uses an autonomous system.
- Internet prefixes must be advertised or received.
- Multiple WAN exits require deterministic preference.
- Very large route sets must be exchanged.
- Route policy is more important than simple shortest-path behavior.

BGP is usually unnecessary for a small internal campus where an IGP such as OSPF, IS-IS, or EIGRP already provides simpler and faster internal reachability.

---

## 4. Key Configuration, Parameters, or CLI

> **Platform:** Cisco IOS / IOS XE.  
> Address-family behavior and supported features vary by platform and software release.

---

## Basic eBGP

R1:

```text
AS 65001
192.0.2.1
```

R2:

```text
AS 65002
192.0.2.2
```

R1:

```cisco
router bgp 65001
 bgp log-neighbor-changes
 neighbor 192.0.2.2 remote-as 65002
 address-family ipv4 unicast
  neighbor 192.0.2.2 activate
  network 203.0.113.0 mask 255.255.255.0
 exit-address-family
```

R2:

```cisco
router bgp 65002
 bgp log-neighbor-changes
 neighbor 192.0.2.1 remote-as 65001
 address-family ipv4 unicast
  neighbor 192.0.2.1 activate
 exit-address-family
```

The `203.0.113.0/24` route must already exist in R1's routing table for the `network` statement to originate it.

---

## Basic iBGP

```cisco
router bgp 65001
 neighbor 10.255.255.2 remote-as 65001
 neighbor 10.255.255.2 update-source Loopback0
 address-family ipv4 unicast
  neighbor 10.255.255.2 activate
  neighbor 10.255.255.2 next-hop-self
 exit-address-family
```

The routers need IGP/static reachability to each other's loopbacks.

---

## eBGP Using Loopbacks

```cisco
router bgp 65001
 neighbor 10.255.255.2 remote-as 65002
 neighbor 10.255.255.2 update-source Loopback0
 neighbor 10.255.255.2 ebgp-multihop 2
```

The correct multihop value depends on the actual routed distance between peers.

---

## Route Reflector

On the RR:

```cisco
router bgp 65001
 address-family ipv4 unicast
  neighbor 10.255.255.2 route-reflector-client
  neighbor 10.255.255.3 route-reflector-client
 exit-address-family
```

Only configure route reflection when the iBGP topology is intentionally designed for it.

---

## Verification

Primary commands:

```cisco
show ip bgp summary
show ip bgp
show ip bgp <prefix>
show ip bgp neighbors
show ip route bgp
show ip route <prefix>
```

Useful operational checks:

```text
Neighbor state
Remote AS
Prefixes received
Prefixes advertised
NEXT_HOP
AS_PATH
LOCAL_PREF
MED
Weight
Best-path marker
Route installation
```

---

## Practical Troubleshooting Sequence

```text
1. Can the neighbor IP be reached?
        ↓
2. Is TCP/179 reachable?
        ↓
3. Do remote-as values match the design?
        ↓
4. Is authentication consistent?
        ↓
5. Does the session reach Established?
        ↓
6. Are prefixes received?
        ↓
7. Does inbound policy permit them?
        ↓
8. Is NEXT_HOP reachable?
        ↓
9. Is the path best in the BGP table?
        ↓
10. Is the route installed in the RIB?
        ↓
11. Is outbound policy advertising the expected route?
        ↓
12. Is the return path valid?
```

Start with:

```cisco
show ip bgp summary
```

then inspect the specific prefix:

```cisco
show ip bgp <prefix>
show ip route <prefix>
```

---

## 5. Common Gotchas and Misconceptions

### BGP Active Means the Session Is Working

**Incorrect.**

```text
Established
= Working session

Active
= BGP is attempting to establish TCP connectivity
```

A peer stuck in Active requires investigation.

---

### The BGP `network` Command Creates a Route

**Incorrect.**

The matching prefix must already exist in the local RIB.

```text
Route in RIB
+
network statement
=
Originate into BGP
```

---

### A Route in `show ip bgp` Must Be in the Routing Table

**Incorrect.**

The route may:

```text
Not be the BGP best path
Have an unreachable NEXT_HOP
Lose the RIB comparison
Be blocked by policy
```

Check both the BGP table and RIB.

---

### iBGP Automatically Changes NEXT_HOP

**Incorrect.**

iBGP normally preserves the next hop.

Use:

```cisco
next-hop-self
```

where the design requires the iBGP router to advertise itself as the reachable next hop.

---

### iBGP Peers Automatically Re-advertise iBGP Routes to Each Other

**Incorrect.**

Routes learned through iBGP are not normally advertised to another iBGP peer.

Use:

```text
Full mesh
Route reflectors
Confederations
```

as appropriate.

---

### Shortest AS_PATH Always Wins

**Incorrect.**

AS_PATH length is only one part of the best-path process.

Attributes such as:

```text
Weight
LOCAL_PREF
Local origination
```

are considered before AS_PATH in the common Cisco selection process.

---

### BGP Selects the Shortest Physical Path

**Incorrect.**

BGP is policy-driven.

```text
Preferred BGP path
≠
Lowest latency
≠
Fewest physical hops
≠
Highest bandwidth
```

Its decision is based on path attributes and policy.

---

### eBGP Must Always Be Directly Connected

**Not always.**

Directly connected peering is the normal default model, but multihop eBGP is supported when explicitly configured and the underlay provides reachability.

---

### BGP Automatically Protects Against Bad Route Advertisements

**Incorrect or Unsafe.**

A correctly formed BGP session can still advertise:

```text
Incorrect prefixes
Too many prefixes
Default route unintentionally
Overly specific routes
Routes that create leaks
```

Use explicit inbound/outbound policy and maximum-prefix controls.

---

### BGP Authentication Encrypts Routing Updates

**Incorrect.**

Neighbor authentication protects the TCP/BGP peering relationship from unauthorized session establishment or spoofing according to the configured mechanism.

It does not provide payload confidentiality like IPsec.

---

## 6. Trade-Offs

### Best Practice

- Apply explicit **inbound and outbound prefix policy** on external peers.
- Use `maximum-prefix` or equivalent controls where route-count expectations are known.
- Use loopbacks for resilient iBGP peering where the IGP provides multiple paths.
- Ensure every advertised NEXT_HOP is reachable.
- Use LOCAL_PREF for internal exit preference and AS_PATH policy for external influence where appropriate.
- Use route reflectors deliberately at scale rather than building uncontrolled iBGP meshes.
- Monitor session state, prefix counts, route changes, and policy violations.

---

### Context-Dependent Trade-Off — Full Routes vs Default/Partial Routes

**Full Internet Table**

```text
+ Maximum path-selection visibility
+ More granular multihoming policy
- High memory/CPU/resource requirements
- More operational complexity
```

**Default / Partial Routes**

```text
+ Lower resource use
+ Simpler operation
- Less path-control granularity
```

The correct model depends on hardware capacity, multihoming goals, traffic-engineering needs, and failure behavior.

---

### Context-Dependent Trade-Off — Route Reflectors

```text
Route Reflector
+ Reduces iBGP session count
+ Scales large AS designs
- Can hide alternate paths
- RR placement affects best-path visibility
```

Use enough topology awareness and redundancy to avoid creating a control-plane bottleneck.

---

### Context-Dependent Trade-Off — Faster BGP Convergence

Aggressive timers and rapid failure detection can improve convergence:

```text
+ Faster response to failure
- Higher sensitivity to transient loss/CPU events
```

BFD or platform-supported fast-failure mechanisms may be more appropriate than simply reducing BGP timers.

---

### Incorrect or Unsafe

- Accepting unrestricted routes from an external peer.
- Advertising unrestricted internal routes to an external peer.
- Using broad redistribution into BGP without filters.
- Changing LOCAL_PREF, MED, AS_PATH prepending, or route-reflector design without evaluating traffic-engineering effects.
- Treating an Established session as proof that the correct prefixes are being exchanged.
- Clearing production BGP sessions unnecessarily instead of using route refresh or soft policy reevaluation where supported.

---

## Quick Reference

```text
BGP
= Border Gateway Protocol

Type
= Path-vector routing protocol

Transport
= TCP 179

eBGP
= Between different ASes

iBGP
= Within the same AS

Default Cisco AD
= eBGP 20
= iBGP 200

Working State
= Established

Main Messages
= OPEN
= UPDATE
= KEEPALIVE
= NOTIFICATION
= ROUTE-REFRESH

NEXT_HOP
= Must be reachable

AS_PATH
= Records AS traversal
= Loop-prevention mechanism

LOCAL_PREF
= Higher is preferred
= Internal AS exit policy

MED
= Lower is preferred when compared
= Suggests preferred inbound entry

Weight
= Cisco-specific
= Local router only
= Higher is preferred

Communities
= Route-policy tags

iBGP Rule
= iBGP-learned routes are not normally advertised to other iBGP peers

Route Reflector
= Scales iBGP by reflecting routes

network Statement
= Originates an existing RIB prefix into BGP

BGP Best Path
≠ Longest-prefix forwarding decision

Core Troubleshooting
= Reachability → TCP 179 → Established → Policy → NEXT_HOP → Best Path → RIB → Advertisement
```

</div>
