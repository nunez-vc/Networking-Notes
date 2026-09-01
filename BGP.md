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

**CCNA Configuration:** Not applicable — BGP configuration is outside current CCNA 200-301 v2.0 scope.

## CCNP Configuration

**IOS-XE — CCNP Enterprise (ENCOR v1.2) — Direct eBGP**

| Command | Description |
|---|---|
| **Create BGP process:**<br>`(config)#router bgp <local-asn>` | Creates the local BGP routing process. |
| **Set BGP router ID:**<br>`(config-router)#bgp router-id <ipv4-address>` | Sets the BGP router identifier. |
| **Configure directly connected eBGP peer:**<br>`(config-router)#neighbor <peer-ip> remote-as <remote-asn>` | Defines the directly connected external BGP peer. |
| **Enter IPv4 unicast AF:**<br>`(config-router)#address-family ipv4 [unicast]` | Enters the IPv4 unicast address family. |
| **Activate IPv4 peer:**<br>`(config-router-af)#neighbor <peer-ip> activate` | Activates the peer for IPv4 unicast. |
| **Advertise IPv4 prefix:**<br>`(config-router-af)#network <network> mask <subnet-mask>` | Injects an exact RIB prefix into BGP. |
| **Verify eBGP summary:**<br>`#show bgp ipv4 unicast summary` | Displays IPv4 BGP peer state and prefix counts. |
| **Verify eBGP peer:**<br>`#show bgp ipv4 unicast neighbors <peer-ip>` | Displays detailed BGP neighbor state and capabilities. |
| **Verify BGP table:**<br>`#show bgp ipv4 unicast` | Displays the IPv4 BGP Loc-RIB. |
| **Verify best path:**<br>`#show bgp ipv4 unicast <prefix> best-path-reason` | Displays path-selection reasons for the specified prefix. |

**IOS-XE — CCNP Enterprise (ENARSI v1.1) — Neighbor Sessions**

| Command | Description |
|---|---|
| **Configure iBGP peer:**<br>`(config-router)#neighbor <peer-ip> remote-as <local-asn>` | Defines an internal BGP peer. |
| **Set neighbor source interface:**<br>`(config-router)#neighbor <peer-ip> update-source <interface-id>` | Sources the BGP session from the specified interface. |
| **Set iBGP next hop to self:**<br>`(config-router)#address-family ipv4 unicast`<br>&nbsp;&nbsp;○ `(config-router-af)#neighbor <peer-ip> next-hop-self` | Advertises the local router as BGP next hop. |
| **Enable multihop eBGP:**<br>`(config-router)#neighbor <peer-ip> ebgp-multihop [<ttl>]` | Permits eBGP peering beyond directly connected neighbors. |
| **Configure neighbor authentication:**<br>`(config-router)#neighbor <peer-ip> password [0\|7] <password>` | Configures MD5 authentication for the BGP TCP session. |
| **Set neighbor timers:**<br>`(config-router)#neighbor <peer-ip> timers <keepalive-seconds> <holdtime-seconds>` | Sets per-neighbor BGP keepalive and hold timers. |
| **Set global BGP timers:**<br>`(config-router)#timers bgp <keepalive-seconds> <holdtime-seconds>` | Sets default BGP timers for all neighbors. |
| **Log neighbor changes:**<br>`(config-router)#bgp log-neighbor-changes` | Logs BGP neighbor up and down events. |
| **Administratively disable peer:**<br>`(config-router)#neighbor <peer-ip> shutdown` | Administratively shuts down the BGP neighbor session. |
| **Re-enable peer:**<br>`(config-router)#no neighbor <peer-ip> shutdown` | Removes the administrative neighbor shutdown. |
| **Limit received prefixes:**<br>`(config-router-af)#neighbor <peer-ip> maximum-prefix <maximum> [<threshold>] [restart <minutes>] [warning-only]` | Configures a maximum received-prefix threshold. |

**IOS-XE — CCNP Enterprise (ENARSI v1.1) — Four-Byte ASNs**

| Command | Description |
|---|---|
| **Use asdot display format:**<br>`(config-router)#bgp asnotation dot` | Displays four-byte ASNs using asdot notation. |
| **Restore asplain display format:**<br>`(config-router)#no bgp asnotation dot` | Restores four-byte ASN output to asplain notation. |
| **Match AS path by regex:**<br>`#show ip bgp regexp <regular-expression>` | Displays BGP routes matching an AS-path expression. |

**IOS-XE — CCNP Enterprise (ENARSI v1.1) — MP-BGP Address Families**

| Command | Description |
|---|---|
| **Disable automatic IPv4 activation:**<br>`(config-router)#no bgp default ipv4-unicast` | Requires explicit address-family activation for configured peers. |
| **Enter IPv4 unicast AF:**<br>`(config-router)#address-family ipv4 [unicast]` | Enters the IPv4 unicast address family. |
| **Activate IPv4 peer:**<br>`(config-router-af)#neighbor <peer-ip> activate` | Activates the peer for IPv4 unicast. |
| **Enter IPv6 unicast AF:**<br>`(config-router)#address-family ipv6 [unicast]` | Enters the IPv6 unicast address family. |
| **Activate IPv6 peer:**<br>`(config-router-af)#neighbor <peer-ipv6> activate` | Activates the peer for IPv6 unicast. |
| **Advertise IPv4 prefix:**<br>`(config-router-af)#network <network> mask <subnet-mask> [route-map <route-map-name>]` | Advertises an exact IPv4 RIB prefix. |
| **Advertise IPv6 prefix:**<br>`(config-router-af)#network <ipv6-prefix>/<prefix-length> [route-map <route-map-name>]` | Advertises an exact IPv6 RIB prefix. |

**IOS-XE — CCNP Enterprise (ENARSI v1.1) — VRF-Lite BGP**

| Command | Description |
|---|---|
| **Enter IPv4 VRF address family:**<br>`(config-router)#address-family ipv4 vrf <vrf-name>` | Enters the BGP IPv4 address family for a VRF. |
| **Configure VRF BGP peer:**<br>`(config-router)#address-family ipv4 vrf <vrf-name>`<br>&nbsp;&nbsp;○ `(config-router-af)#neighbor <peer-ip> remote-as <remote-asn>`<br>&nbsp;&nbsp;○ `(config-router-af)#neighbor <peer-ip> activate` | Defines and activates a BGP peer inside the VRF. |
| **Advertise VRF IPv4 prefix:**<br>`(config-router-af)#network <network> mask <subnet-mask>` | Advertises a VRF RIB prefix into BGP. |

**IOS-XE — CCNP Enterprise (ENARSI v1.1) — Route Origination**

| Command | Description |
|---|---|
| **Advertise IPv4 network:**<br>`(config-router-af)#network <network> mask <subnet-mask> [route-map <route-map-name>]` | Originates an exact IPv4 RIB prefix. |
| **Advertise IPv6 network:**<br>`(config-router-af)#network <ipv6-prefix>/<prefix-length> [route-map <route-map-name>]` | Originates an exact IPv6 RIB prefix. |
| **Redistribute connected routes:**<br>`(config-router-af)#redistribute connected [route-map <route-map-name>]` | Redistributes connected routes into the address family. |
| **Redistribute static routes:**<br>`(config-router-af)#redistribute static [route-map <route-map-name>]` | Redistributes static routes into the address family. |
| **Redistribute OSPF routes:**<br>`(config-router-af)#redistribute ospf <process-id> [route-map <route-map-name>]` | Redistributes OSPF routes into the address family. |
| **Originate default toward peer:**<br>`(config-router-af)#neighbor <peer-ip> default-originate [route-map <route-map-name>]` | Advertises a default route to the specified peer. |

**IOS-XE — CCNP Enterprise (ENARSI v1.1) — BGP Aggregation**

| Command | Description |
|---|---|
| **Create IPv4 aggregate:**<br>`(config-router-af)#aggregate-address <network> <subnet-mask>` | Creates an IPv4 BGP aggregate from component routes. |
| **Suppress IPv4 component routes:**<br>`(config-router-af)#aggregate-address <network> <subnet-mask> summary-only` | Advertises only the IPv4 aggregate route. |
| **Preserve IPv4 AS path set:**<br>`(config-router-af)#aggregate-address <network> <subnet-mask> as-set [summary-only]` | Preserves component AS information in the aggregate. |
| **Create IPv6 aggregate:**<br>`(config-router-af)#aggregate-address <ipv6-prefix>/<prefix-length>` | Creates an IPv6 BGP aggregate from component routes. |
| **Suppress IPv6 component routes:**<br>`(config-router-af)#aggregate-address <ipv6-prefix>/<prefix-length> summary-only` | Advertises only the IPv6 aggregate route. |
| **Preserve IPv6 AS path set:**<br>`(config-router-af)#aggregate-address <ipv6-prefix>/<prefix-length> as-set [summary-only]` | Preserves component AS information in the IPv6 aggregate. |

**IOS-XE — CCNP Enterprise (ENARSI v1.1) — Peer Groups**

| Command | Description |
|---|---|
| **Create peer group:**<br>`(config-router)#neighbor <peer-group-name> peer-group` | Creates a reusable BGP peer group. |
| **Set peer-group remote AS:**<br>`(config-router)#neighbor <peer-group-name> remote-as <remote-asn>` | Assigns the remote ASN to the peer group. |
| **Assign neighbor to peer group:**<br>`(config-router)#neighbor <peer-ip> peer-group <peer-group-name>` | Adds the BGP neighbor to the peer group. |
| **Activate peer group:**<br>`(config-router-af)#neighbor <peer-group-name> activate` | Activates the peer group for the address family. |

**IOS-XE — CCNP Enterprise (ENARSI v1.1) — Route Reflector**

| Command | Description |
|---|---|
| **Configure route-reflector client:**<br>`(config-router)#address-family ipv4 unicast`<br>&nbsp;&nbsp;○ `(config-router-af)#neighbor <peer-ip> route-reflector-client` | Marks the iBGP neighbor as a route-reflector client. |
| **Set route-reflector cluster ID:**<br>`(config-router)#bgp cluster-id <cluster-id>` | Configures the BGP route-reflector cluster identifier. |

**IOS-XE — CCNP Enterprise (ENARSI v1.1) — Prefix Filtering**

| Command | Description |
|---|---|
| **Create IPv4 prefix list:**<br>`(config)#ip prefix-list <list-name> [seq <sequence>] {permit\|deny} <prefix>/<length> [ge <length>] [le <length>]` | Creates an IPv4 prefix-filter entry. |
| **Create IPv6 prefix list:**<br>`(config)#ipv6 prefix-list <list-name> [seq <sequence>] {permit\|deny} <prefix>/<length> [ge <length>] [le <length>]` | Creates an IPv6 prefix-filter entry. |
| **Apply prefix list to neighbor:**<br>`(config-router-af)#neighbor <peer-ip> prefix-list <list-name> {in\|out}` | Filters BGP prefixes for the specified neighbor. |
| **Apply distribute list:**<br>`(config-router-af)#neighbor <peer-ip> distribute-list {<acl-number>\|<acl-name>} {in\|out}` | Filters neighbor routes using an access list. |
| **Verify IPv4 prefix list:**<br>`#show ip prefix-list [<list-name>]` | Displays IPv4 prefix-list entries and counters. |
| **Verify IPv6 prefix list:**<br>`#show ipv6 prefix-list [<list-name>]` | Displays IPv6 prefix-list entries and counters. |

**IOS-XE — CCNP Enterprise (ENARSI v1.1) — AS-Path Filtering**

| Command | Description |
|---|---|
| **Create AS-path ACL:**<br>`(config)#ip as-path access-list <acl-number> {permit\|deny} <regular-expression>` | Creates an AS-path regular-expression filter. |
| **Apply AS-path filter:**<br>`(config-router-af)#neighbor <peer-ip> filter-list <acl-number> {in\|out}` | Applies the AS-path filter to neighbor updates. |
| **Verify AS-path matches:**<br>`#show ip bgp regexp <regular-expression>` | Displays BGP routes matching the AS-path expression. |

**IOS-XE — CCNP Enterprise (ENARSI v1.1) — Route Maps**

| Command | Description |
|---|---|
| **Create route-map sequence:**<br>`(config)#route-map <route-map-name> {permit\|deny} [<sequence>]` | Creates a route-map sequence. |
| **Match IPv4 prefix list:**<br>`(config-route-map)#match ip address prefix-list <list-name>` | Matches routes permitted by an IPv4 prefix list. |
| **Match AS path:**<br>`(config-route-map)#match as-path <acl-number>` | Matches routes permitted by an AS-path ACL. |
| **Match BGP community:**<br>`(config-route-map)#match community <community-list> [exact]` | Matches routes by BGP community values. |
| **Apply inbound route map:**<br>`(config-router-af)#neighbor <peer-ip> route-map <route-map-name> in` | Applies a route map to received BGP updates. |
| **Apply outbound route map:**<br>`(config-router-af)#neighbor <peer-ip> route-map <route-map-name> out` | Applies a route map to advertised BGP updates. |
| **Verify route map:**<br>`#show route-map [<route-map-name>]` | Displays route-map configuration and match counters. |

**IOS-XE — CCNP Enterprise (ENARSI v1.1) — Path Attribute Manipulation**

| Command | Description |
|---|---|
| **Set BGP weight:**<br>`(config-route-map)#set weight <0-65535>` | Sets Cisco-local BGP weight for matched routes. |
| **Set local preference:**<br>`(config-route-map)#set local-preference <value>` | Sets BGP local preference for matched routes. |
| **Prepend AS path:**<br>`(config-route-map)#set as-path prepend <asn> [<asn> ...]` | Prepends AS numbers to matched route advertisements. |
| **Set MED:**<br>`(config-route-map)#set metric <value>` | Sets BGP MED for matched routes. |
| **Set origin attribute:**<br>`(config-route-map)#set origin {igp\|egp <asn>\|incomplete}` | Sets the BGP origin attribute. |
| **Set BGP community:**<br>`(config-route-map)#set community <community> [additive]` | Sets or appends a BGP community value. |

**IOS-XE — CCNP Enterprise (ENARSI v1.1) — BGP Communities**

| Command | Description |
|---|---|
| **Use new community format:**<br>`(config)#ip bgp-community new-format` | Displays standard communities in ASN:value format. |
| **Send standard communities:**<br>`(config-router-af)#neighbor <peer-ip> send-community [standard]` | Sends standard BGP communities to the peer. |
| **Send extended communities:**<br>`(config-router-af)#neighbor <peer-ip> send-community extended` | Sends extended BGP communities to the peer. |
| **Send both community types:**<br>`(config-router-af)#neighbor <peer-ip> send-community both` | Sends standard and extended communities. |
| **Create standard community list:**<br>`(config)#ip community-list standard <list-name> {permit\|deny} <community>` | Creates a named standard BGP community list. |
| **Create expanded community list:**<br>`(config)#ip community-list expanded <list-name> {permit\|deny} <regular-expression>` | Creates a named regex-based community list. |
| **Match community in route map:**<br>`(config-route-map)#match community <community-list> [exact]` | Matches routes using the configured community list. |
| **Set community in route map:**<br>`(config-route-map)#set community <community> [additive]` | Sets or appends communities on matched routes. |
| **Verify community list:**<br>`#show ip community-list [<community-list>]` | Displays configured BGP community lists. |
| **Verify routes by community:**<br>`#show bgp ipv4 unicast community <community>` | Displays IPv4 BGP routes containing the community. |

**IOS-XE — CCNP Enterprise (ENARSI v1.1) — Private AS Handling**

| Command | Description |
|---|---|
| **Remove private ASNs:**<br>`(config-router-af)#neighbor <peer-ip> remove-private-as` | Removes eligible private ASNs from outbound eBGP updates. |
| **Remove all private ASNs:**<br>`(config-router-af)#neighbor <peer-ip> remove-private-as all` | Removes all private ASNs from outbound eBGP updates. |
| **Replace removed private ASNs:**<br>`(config-router-af)#neighbor <peer-ip> remove-private-as all replace-as` | Replaces removed private ASNs with the local ASN. |

**IOS-XE — CCNP Enterprise (ENARSI v1.1) — Route Refresh and Session Reset**

| Command | Description |
|---|---|
| **Soft-refresh IPv4 inbound routes:**<br>`#clear bgp ipv4 unicast <peer-ip> soft in` | Reprocesses inbound IPv4 routes without resetting the session. |
| **Soft-refresh IPv4 outbound routes:**<br>`#clear bgp ipv4 unicast <peer-ip> soft out` | Reprocesses outbound IPv4 routes without resetting the session. |
| **Soft-refresh IPv6 inbound routes:**<br>`#clear bgp ipv6 unicast <peer-ipv6> soft in` | Reprocesses inbound IPv6 routes without resetting the session. |
| **Soft-refresh IPv6 outbound routes:**<br>`#clear bgp ipv6 unicast <peer-ipv6> soft out` | Reprocesses outbound IPv6 routes without resetting the session. |
| **Hard-reset one IPv4 peer:**<br>`#clear ip bgp <peer-ip>` | Resets the selected BGP session and relearns routes. |
| **Hard-reset all IPv4 peers:**<br>`#clear ip bgp *` | Resets all IPv4 BGP sessions. |

**IOS-XE — CCNP Enterprise (ENCOR v1.2 / ENARSI v1.1) — Verification**

| Command | Description |
|---|---|
| **Show IPv4 BGP summary:**<br>`#show bgp ipv4 unicast summary` | Displays IPv4 peer state and received-prefix counts. |
| **Show IPv6 BGP summary:**<br>`#show bgp ipv6 unicast summary` | Displays IPv6 peer state and received-prefix counts. |
| **Show IPv4 neighbor details:**<br>`#show bgp ipv4 unicast neighbors <peer-ip>` | Displays IPv4 neighbor state, timers, and capabilities. |
| **Show IPv6 neighbor details:**<br>`#show bgp ipv6 unicast neighbors <peer-ipv6>` | Displays IPv6 neighbor state, timers, and capabilities. |
| **Show IPv4 BGP table:**<br>`#show bgp ipv4 unicast` | Displays all IPv4 BGP Loc-RIB entries. |
| **Show IPv6 BGP table:**<br>`#show bgp ipv6 unicast` | Displays all IPv6 BGP Loc-RIB entries. |
| **Show one IPv4 prefix:**<br>`#show bgp ipv4 unicast <prefix>` | Displays all BGP paths for one IPv4 prefix. |
| **Show selected best path:**<br>`#show bgp ipv4 unicast <prefix> bestpath` | Displays only the selected best BGP path. |
| **Show best-path reason:**<br>`#show bgp ipv4 unicast <prefix> best-path-reason` | Displays selection reasons for all candidate paths. |
| **Show advertised routes:**<br>`#show bgp ipv4 unicast neighbors <peer-ip> advertised-routes` | Displays routes advertised to the specified IPv4 peer. |
| **Show BGP IPv4 RIB routes:**<br>`#show ip route bgp` | Displays IPv4 routes installed from BGP. |
| **Show BGP IPv6 RIB routes:**<br>`#show ipv6 route bgp` | Displays IPv6 routes installed from BGP. |
| **Show BGP configuration:**<br>`#show running-config \| section router bgp` | Displays the configured BGP routing process. |


</div>
