<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# Quality of Service (QoS)

> **Core idea:** QoS classifies traffic and gives different traffic classes different forwarding treatment when network resources are constrained. It does **not create bandwidth**; it controls how bandwidth, queue space, delay, and packet loss are allocated during contention.

---

## 1. What It Is

**Quality of Service (QoS)** is a set of mechanisms used by routers and switches to classify traffic and apply differentiated forwarding treatment based on business or application requirements.

QoS primarily manages four delivery characteristics:

```text
Bandwidth
Delay (latency)
Jitter (delay variation)
Packet loss
```

Real-time traffic such as voice and interactive video is especially sensitive to delay, jitter, and loss, while many data applications tolerate delay better.

---

## 2. How It Works

## The QoS Processing Model

A typical QoS policy follows this logic:

```text
Packet arrives
     ↓
Classify traffic
     ↓
Trust or remark QoS marking
     ↓
Apply rate control if required
     ↓
Forward toward egress
     ↓
Queue during congestion
     ↓
Schedule traffic for transmission
     ↓
Drop selectively if congestion persists
```

The major QoS mechanisms are:

```text
Classification
Marking
Policing
Shaping
Queuing / Scheduling
Congestion Avoidance
```

---

## Classification

**Classification** identifies traffic that should receive a particular treatment.

Traffic can be classified using fields such as:

```text
Source / destination IP
Protocol
TCP / UDP ports
DSCP
CoS / PCP
ACL match
Application identification on supported platforms
```

Example:

```text
UDP voice traffic
        ↓
Classify as VOICE
        ↓
Apply low-latency treatment
```

Classification itself does not improve packet delivery; it determines which subsequent QoS action applies.

---

## Marking

**Marking** writes a QoS value into a packet/frame so downstream devices can classify it efficiently.

### Layer 3 — DSCP

IPv4 and IPv6 use a **6-bit Differentiated Services Code Point (DSCP)** field:

```text
DSCP range = 0-63
```

Common examples:

| Marking | Decimal | Typical Meaning |
|---|---:|---|
| **DSCP 0 / Default** | 0 | Best effort |
| **EF** | 46 | Low-delay traffic, commonly voice payload |
| **AFxx** | Varies | Assured Forwarding classes |
| **CSx** | Multiples of 8 | Class Selector values |

DSCP is useful because the IP header normally remains with the packet across routed hops.

---

### Layer 2 — CoS / PCP

An 802.1Q Ethernet tag contains a 3-bit **Priority Code Point (PCP)** field, commonly called **CoS** on Cisco platforms.

```text
CoS / PCP range = 0-7
```

CoS exists only when the frame carries the relevant 802.1Q information.

```text
DSCP
= Layer 3
= More persistent across routed networks

CoS / PCP
= Layer 2
= Relevant on tagged Ethernet links
```

---

## Trust Boundary

A **QoS trust boundary** is the point where the network begins trusting packet markings.

Example:

```text
Untrusted PC
     ↓
Access Switch
Classify + Remark
     ↓
Trust Boundary
     ↓
Distribution / Core / WAN
Trust DSCP
```

Do not blindly trust markings from uncontrolled endpoints.

Otherwise, an endpoint could mark ordinary traffic as high priority:

```text
Bulk data marked EF
        ↓
Network trusts it
        ↓
Bulk data enters priority treatment
```

A common design is to classify and mark traffic as close to the source as practical, then use those trusted markings deeper in the network.

---

## DiffServ

The most common scalable QoS model is **Differentiated Services (DiffServ)**.

```text
Edge:
Perform detailed classification and marking

Core:
Read DSCP
Apply per-hop forwarding behavior
```

This avoids maintaining per-flow reservation state on every router.

Important principle:

> **DSCP is a marking, not a guarantee.** Each device must have a policy that maps the marking to the intended forwarding treatment.

---

## Queuing and Congestion Management

Queuing matters when an interface cannot immediately transmit every packet waiting for it.

```text
Packets arrive faster than interface can transmit
                 ↓
              Congestion
                 ↓
               Queues
```

Without differentiated queuing, packets are generally treated as best effort.

---

### Class-Based Weighted Fair Queuing (CBWFQ)

CBWFQ creates separate queues for traffic classes and assigns bandwidth to those classes during congestion.

Conceptually:

```text
VOICE      → Queue 1
VIDEO      → Queue 2
BUSINESS   → Queue 3
DEFAULT    → Queue 4
```

The scheduler determines which queue transmits next.

---

### Low-Latency Queuing (LLQ)

LLQ adds a **strict-priority queue** to class-based queuing.

Typical use:

```text
Voice RTP
        ↓
Priority Queue
        ↓
Transmit ahead of non-priority queues during congestion
```

Priority traffic must be bounded so it cannot starve every other class.

On IOS/IOS XE, the MQC `priority` action provides LLQ behavior and enforces the configured priority allowance during congestion.

> Priority treatment should be reserved for traffic that genuinely requires very low delay and jitter.

---

## Policing

**Policing** enforces a traffic rate by measuring traffic against a configured rate.

Traffic that exceeds the allowed rate can be:

```text
Dropped
or
Remarked
```

Conceptually:

```text
Traffic rate
     ↓
Within allowed rate?
   /         \
 Yes         No
  |           |
Forward    Drop / Remark
```

Policing does not normally buffer excess traffic.

Result:

```text
+ Controls rate immediately
+ Protects network resources
- Can introduce packet loss
```

Common use:

```text
Ingress rate enforcement
Provider/customer traffic contracts
Scavenger or non-critical traffic limits
```

---

## Shaping

**Shaping** also controls traffic rate, but excess traffic is buffered and transmitted later.

```text
Traffic burst
     ↓
Exceeds shaping rate
     ↓
Queue excess packets
     ↓
Transmit them later
```

Result:

```text
+ Smooths bursts
+ Reduces downstream drops
- Adds delay
- Requires queue memory
```

Shaping is typically applied on egress.

---

## Policing vs Shaping

| Policing | Shaping |
|---|---|
| Drops or remarks excess traffic | Buffers excess traffic |
| Does not smooth traffic | Smooths traffic over time |
| Can increase packet loss | Can increase delay |
| Common ingress enforcement use | Common egress WAN use |
| Immediate rate enforcement | Delayed transmission |

Memory:

```text
Policing
= Excess traffic is punished

Shaping
= Excess traffic waits
```

---

## Congestion Avoidance

If queues fill completely, **tail drop** discards newly arriving packets.

```text
Queue full
     ↓
New packet arrives
     ↓
Drop
```

**Weighted Random Early Detection (WRED)** can begin probabilistically dropping selected traffic before the queue is full.

This can:

```text
Signal TCP senders to slow down earlier
Reduce global TCP synchronization
Preferentially retain higher-priority marked traffic
```

WRED is primarily useful for responsive traffic such as TCP.

It is not a replacement for LLQ for real-time voice traffic.

---

## QoS Is Per-Hop

QoS treatment occurs independently at each device:

```text
Access Switch
     ↓
Distribution
     ↓
WAN Edge
     ↓
Provider
     ↓
Remote Edge
```

End-to-end QoS therefore requires consistent classification, marking, queuing, and provider treatment across the entire path.

```text
Correct marking on one router
≠
End-to-end QoS
```

Provider networks may remark or map customer DSCP values according to the service contract.

---

## QoS and Congestion

QoS is most visible when traffic competes for a constrained resource.

Example:

```text
1-Gbps LAN
      ↓
100-Mbps WAN

Traffic offered = 180 Mbps
WAN capacity    = 100 Mbps
```

The WAN egress is the congestion point.

QoS can determine:

```text
Which traffic transmits first
Which traffic receives bandwidth
Which traffic waits
Which traffic is dropped
```

QoS cannot make the 100-Mbps link physically transmit 180 Mbps.

---

## 3. Why and When It Is Used

QoS is appropriate where traffic classes have different delivery requirements and may compete for limited resources.

Typical use cases:

```text
Voice
Interactive video
Video conferencing
Business-critical applications
WAN links
Internet edges
Wireless networks
Oversubscribed campus links
Provider handoffs
```

QoS is especially useful when:

```text
Bandwidth is constrained
Traffic is bursty
Real-time applications share links with bulk traffic
Different applications require different service levels
```

QoS may add little value on a path that is consistently uncongested and has ample capacity, although classification/marking can still be useful for downstream policy.

QoS should not be used to conceal chronic under-capacity when adding bandwidth is the correct fix.

---

## 4. Key Configuration, Parameters, or CLI

> **Platform:** Cisco IOS / IOS XE using Modular QoS CLI (MQC).  
> QoS capabilities and hardware implementation vary significantly by platform, interface type, ASIC, and software release. Validate feature support on the exact device before production deployment.

---

## MQC Structure

Cisco MQC uses three main components:

```text
class-map
= Identify traffic

policy-map
= Define treatment

service-policy
= Apply policy to an interface
```

---

### Classify Traffic

```cisco
class-map match-any VOICE
 match dscp ef
```

---

### Mark Traffic

```cisco
policy-map MARK-IN
 class VOICE
  set dscp ef
 class class-default
  set dscp default
```

Apply inbound:

```cisco
interface GigabitEthernet0/0
 service-policy input MARK-IN
```

---

### LLQ and CBWFQ

Example egress policy:

```cisco
policy-map WAN-QOS
 class VOICE
  priority percent 10
 class BUSINESS
  bandwidth percent 30
 class class-default
```

Apply:

```cisco
interface GigabitEthernet0/1
 service-policy output WAN-QOS
```

Meaning:

```text
VOICE
= Strict-priority treatment during congestion

BUSINESS
= Class bandwidth allocation during congestion

class-default
= Everything not matched elsewhere
```

Exact scheduler behavior depends on platform and release.

---

### Policing

Example:

```cisco
policy-map POLICE-IN
 class BULK
  police 10000000
```

The supported police syntax and optional conform/exceed actions vary by IOS/IOS XE release and platform.

Verify with the platform command reference before production use.

---

### Shaping

Example:

```cisco
policy-map SHAPE-WAN
 class class-default
  shape average 100000000
```

This shapes the class to approximately:

```text
100 Mbps
```

On more complex WAN designs, hierarchical QoS can place child queuing policies under a shaped parent policy.

---

## Verification

Primary command:

```cisco
show policy-map interface
```

It provides evidence such as:

```text
Packets matched
Bytes matched
Marking actions
Queue statistics
Drops
Policer statistics
Shaping state
Offered rate
Drop rate
```

Additional useful commands:

```cisco
show class-map
show policy-map
show running-config interface <interface>
show interfaces <interface>
```

For platform-specific hardware QoS:

```text
Use the exact Catalyst / router platform QoS verification commands
documented for that hardware and IOS XE release.
```

Do not assume all Catalyst switches expose identical MQC behavior or counters.

---

## Practical Troubleshooting Sequence

```text
1. Where is the actual congestion point?
        ↓
2. Is the policy applied to the correct interface and direction?
        ↓
3. Does the class-map match the intended traffic?
        ↓
4. Are DSCP/CoS markings what the policy expects?
        ↓
5. Are class counters increasing?
        ↓
6. Is the queue actually congested?
        ↓
7. Are drops occurring in the expected class?
        ↓
8. Is policing dropping traffic unexpectedly?
        ↓
9. Is shaping configured to the real downstream rate?
        ↓
10. Does the next network/provider preserve or remark the QoS markings?
```

Start with:

```cisco
show policy-map interface
```

because configuration without matching traffic has no operational effect.

---

## 5. Common Gotchas and Misconceptions

### QoS Creates More Bandwidth

**Incorrect.**

QoS decides how existing bandwidth is used.

```text
100 Mbps link
+ QoS
= Still 100 Mbps
```

If sustained demand exceeds capacity, packets must eventually wait or be dropped.

---

### DSCP EF Automatically Gives Priority

**Incorrect.**

DSCP is only a marking.

```text
DSCP EF
+
No QoS policy
=
No guaranteed priority treatment
```

Every relevant hop must map the marking to the desired behavior.

---

### QoS Matters Only for Voice

**Incorrect.**

QoS can protect:

```text
Voice
Video
Business-critical applications
Routing/control traffic
Transactional applications
```

and constrain low-priority traffic.

---

### CoS and DSCP Are the Same Field

**Incorrect.**

```text
CoS / PCP
= 3 bits in 802.1Q Layer 2 information

DSCP
= 6 bits in IPv4/IPv6 Layer 3 header
```

They can be mapped to each other, but they are different markings.

---

### Trusting Endpoint Markings Is Always Safe

**Incorrect or Unsafe.**

Uncontrolled endpoints can mark traffic with high-priority DSCP values.

Establish a deliberate trust boundary and remark traffic where necessary.

---

### Shaping and Policing Are the Same

**Incorrect.**

```text
Policing
= Drop/remark excess

Shaping
= Buffer excess and send later
```

Their effect on packet loss and delay is fundamentally different.

---

### Priority Queue Means Unlimited Bandwidth

**Incorrect.**

A priority class must be bounded.

Otherwise, sustained priority traffic could starve other queues.

---

### QoS Must Be Applied Everywhere Identically

**Incorrect.**

QoS is **per-hop** and should reflect the constraints at each hop.

Example:

```text
Access switch
= Classify / mark

WAN edge
= Shape / queue

Provider
= Map DSCP to contracted service class
```

The policy should be consistent in intent, not necessarily identical in configuration.

---

### No QoS Drops Means QoS Is Working

**Not necessarily.**

The interface may simply not be congested.

QoS queuing behavior often becomes visible only when offered traffic exceeds available service capacity.

---

### WRED Is Ideal for Voice

**Incorrect.**

Real-time voice is generally better protected with an LLQ/priority design.

Random early dropping is mainly useful for congestion-responsive traffic such as TCP.

---

## 6. Trade-Offs

### Best Practice

- Identify the actual congestion points before configuring QoS.
- Classify and mark close to the trusted network edge.
- Prefer DSCP for Layer 3 end-to-end marking.
- Reserve strict priority for genuinely delay-sensitive traffic.
- Shape to a known downstream/provider rate when the physical interface speed exceeds the contracted service rate.
- Verify QoS with real counters under representative load.
- Keep the number of traffic classes operationally manageable.

---

### Context-Dependent Trade-Off — More QoS Classes

```text
More classes
+ Finer application differentiation
+ More precise service levels
- More configuration complexity
- Harder troubleshooting
- Harder provider mapping
```

Use only as many classes as the network can operate consistently end to end.

---

### Context-Dependent Trade-Off — Policing vs Shaping

```text
Policing
+ Immediate rate enforcement
+ No buffering delay
- Drops excess traffic

Shaping
+ Smooths bursts
+ Can reduce downstream loss
- Adds buffering and delay
```

Choose based on where rate enforcement occurs and whether excess traffic should be dropped or delayed.

---

### Context-Dependent Trade-Off — Priority Bandwidth

Allocating more bandwidth to the priority queue can improve real-time performance under congestion.

However:

```text
More priority bandwidth
=
Less guaranteed capacity available to other classes
```

Size priority traffic from measured or engineered demand, not guesswork.

---

### Incorrect or Unsafe

- Marking broad categories of traffic as EF to "make them faster."
- Trusting arbitrary endpoint DSCP markings.
- Giving the priority queue enough sustained traffic to starve other classes.
- Shaping or policing without confirming the real link/provider rate.
- Changing production QoS without baseline traffic measurements and post-change counter validation.
- Treating QoS as a substitute for adding bandwidth when the link is chronically undersized.

---

## Quick Reference

```text
QoS
= Differentiated forwarding treatment

Primary Problems
= Bandwidth
= Delay
= Jitter
= Packet loss

Classification
= Identify traffic

Marking
= Label traffic for downstream treatment

DSCP
= Layer 3
= 6 bits
= Values 0-63

EF
= DSCP 46
= Common voice-payload marking

CoS / PCP
= Layer 2 802.1Q
= 3 bits
= Values 0-7

Trust Boundary
= Point where markings become trusted

DiffServ
= Scalable class-based per-hop QoS model

CBWFQ
= Bandwidth allocation by class

LLQ
= Strict-priority queue for low-latency traffic

Policing
= Drop / remark excess traffic

Shaping
= Buffer excess traffic and send later

WRED
= Early selective dropping for congestion avoidance

MQC
= class-map
→ policy-map
→ service-policy

Primary Verification
= show policy-map interface

Core Rule
= QoS does not create bandwidth
= QoS controls who gets service when contention occurs
```

## CCNA Configuration

Current CCNA 200-301 v1.1 QoS scope is explanation-only; no CLI configuration commands are required.

## CCNP Configuration

**CCNP Enterprise — IOS-XE — MQC Classification**

| Command | Description |
|---|---|
| **Create match-all class map:**<br>`(config)#class-map match-all <class-map-name>` | Enters class-map mode using logical AND matching. |
| **Create match-any class map:**<br>`(config)#class-map match-any <class-map-name>` | Enters class-map mode using logical OR matching. |
| **Match named ACL:**<br>`(config-cmap)#match access-group name <acl-name>` | Classifies traffic matched by the named ACL. |
| **Match numbered ACL:**<br>`(config-cmap)#match access-group <acl-number>` | Classifies traffic matched by the numbered ACL. |
| **Match all traffic:**<br>`(config-cmap)#match any` | Matches every packet entering the class map. |
| **Match CoS:**<br>`(config-cmap)#match cos <cos-value-list>` | Matches specified Layer 2 CoS values. |
| **Match DSCP:**<br>`(config-cmap)#match dscp <dscp-value-list>` | Matches specified DSCP values for IPv4 and IPv6. |
| **Match IPv4 DSCP:**<br>`(config-cmap)#match ip dscp <dscp-value-list>` | Matches specified DSCP values for IPv4 traffic. |
| **Match precedence:**<br>`(config-cmap)#match precedence <precedence-value-list>` | Matches specified precedence values for IPv4 and IPv6. |
| **Match IPv4 precedence:**<br>`(config-cmap)#match ip precedence <precedence-value-list>` | Matches specified IP precedence values for IPv4. |
| **Match RTP ports:**<br>`(config-cmap)#match ip rtp <starting-port> <port-range>` | Matches RTP traffic within the specified UDP port range. |
| **Match NBAR protocol:**<br>`(config-cmap)#match protocol <protocol-name>` | Matches an NBAR protocol; platform support varies. |
| **Match QoS group:**<br>`(config-cmap)#match qos-group <qos-group-value>` | Matches the specified internal QoS group value. |

**CCNP Enterprise — IOS-XE — MQC Policy Construction**

| Command | Description |
|---|---|
| **Create policy map:**<br>`(config)#policy-map <policy-map-name>` | Enters policy-map configuration mode. |
| **Attach class map:**<br>`(config-pmap)#class <class-map-name>` | Enters policy class configuration for classified traffic. |
| **Enter default class:**<br>`(config-pmap)#class class-default` | Enters the implicit class for otherwise unmatched traffic. |
| **Apply inbound policy:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#service-policy input <policy-map-name>` | Applies the policy map to interface ingress traffic. |
| **Apply outbound policy:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#service-policy output <policy-map-name>` | Applies the policy map to interface egress traffic. |
| **Remove inbound policy:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no service-policy input <policy-map-name>` | Removes the inbound service policy from the interface. |
| **Remove outbound policy:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no service-policy output <policy-map-name>` | Removes the outbound service policy from the interface. |

**CCNP Enterprise — IOS-XE — Class-Based Marking**

| Command | Description |
|---|---|
| **Set DSCP:**<br>`(config-pmap-c)#set dscp <dscp-value>` | Marks DSCP for classified IPv4 and IPv6 packets. |
| **Set IPv4 DSCP:**<br>`(config-pmap-c)#set ip dscp <dscp-value>` | Marks DSCP for classified IPv4 packets. |
| **Set CoS:**<br>`(config-pmap-c)#set cos <cos-value>` | Marks the Layer 2 CoS value. |
| **Set precedence:**<br>`(config-pmap-c)#set precedence <precedence-value>` | Marks precedence for classified IPv4 and IPv6 packets. |
| **Set IPv4 precedence:**<br>`(config-pmap-c)#set ip precedence <precedence-value>` | Marks IP precedence for classified IPv4 packets. |
| **Set QoS group:**<br>`(config-pmap-c)#set qos-group <qos-group-id>` | Assigns an internal QoS group identifier. |

**CCNP Enterprise — IOS-XE — Class-Based Policing**

| Command | Description |
|---|---|
| **Configure basic policer:**<br>`(config-pmap-c)#police <cir-bps>` | Polices traffic to the specified committed information rate. |
| **Police and drop excess:**<br>`(config-pmap-c)#police <cir-bps> conform-action transmit exceed-action drop` | Transmits conforming traffic and drops excess traffic. |
| **Set single-rate bursts:**<br>`(config-pmap-c)#police cir <cir-bps> bc <bc-bytes> be <be-bytes>` | Configures CIR, committed burst, and excess burst sizes. |
| **Remark exceeding traffic:**<br>`(config-pmap-c)#police <cir-bps> conform-action transmit exceed-action set-dscp-transmit <dscp-value>` | Remarks and transmits traffic exceeding the CIR. |
| **Configure three-color policer:**<br>`(config-pmap-c)#police <cir-bps> conform-action set-dscp-transmit <conform-dscp> exceed-action set-dscp-transmit <exceed-dscp> violate-action drop` | Applies distinct conform, exceed, and violate actions. |
| **Configure two-rate policer:**<br>`(config-pmap-c)#police cir <cir-bps> pir <pir-bps> conform-action transmit exceed-action set-dscp-transmit <dscp-value> violate-action drop` | Configures CIR/PIR policing with three traffic actions. |

**CCNP Enterprise — IOS-XE — LLQ and CBWFQ**

| Command | Description |
|---|---|
| **Configure strict priority rate:**<br>`(config-pmap-c)#priority <kbps>` | Creates a strict-priority queue with conditional policing. |
| **Configure strict priority percent:**<br>`(config-pmap-c)#priority percent <percentage>` | Allocates strict-priority bandwidth as an interface percentage. |
| **Configure level-1 priority:**<br>`(config-pmap-c)#priority level 1` | Creates an unconstrained level-1 strict-priority queue. |
| **Configure level-2 priority:**<br>`(config-pmap-c)#priority level 2` | Creates an unconstrained level-2 strict-priority queue. |
| **Configure level-1 percent:**<br>`(config-pmap-c)#priority level 1 percent <percentage>` | Configures level-1 priority with percentage-based conditional policing. |
| **Configure level-2 percent:**<br>`(config-pmap-c)#priority level 2 percent <percentage>` | Configures level-2 priority with percentage-based conditional policing. |
| **Guarantee bandwidth:**<br>`(config-pmap-c)#bandwidth <kbps>` | Guarantees minimum class bandwidth in kilobits per second. |
| **Guarantee bandwidth percent:**<br>`(config-pmap-c)#bandwidth percent <percentage>` | Guarantees minimum bandwidth as an absolute percentage. |
| **Allocate remaining percent:**<br>`(config-pmap-c)#bandwidth remaining percent <percentage>` | Allocates a percentage of bandwidth remaining after priority traffic. |
| **Allocate remaining ratio:**<br>`(config-pmap-c)#bandwidth remaining ratio <ratio>` | Allocates remaining bandwidth using relative class ratios. |
| **Enable flow fair queuing:**<br>`(config-pmap-c)#fair-queue` | Enables flow-based fair queuing within the class. |

**CCNP Enterprise — IOS-XE — Congestion Avoidance**

| Command | Description |
|---|---|
| **Enable WRED:**<br>`(config-pmap-c)#random-detect` | Enables precedence-based weighted random early detection. |
| **Enable DSCP WRED:**<br>`(config-pmap-c)#random-detect dscp-based` | Enables WRED using DSCP-based thresholds. |
| **Enable precedence WRED:**<br>`(config-pmap-c)#random-detect precedence-based` | Enables WRED using precedence-based thresholds. |
| **Enable CoS WRED:**<br>`(config-pmap-c)#random-detect cos-based` | Enables WRED using CoS-based thresholds. |
| **Set queue limit:**<br>`(config-pmap-c)#queue-limit <packets>` | Sets the class queue tail-drop limit. |

**CCNP Enterprise — IOS-XE — Class-Based Shaping and HQoS**

| Command | Description |
|---|---|
| **Configure average shaping:**<br>`(config-pmap-c)#shape average <mean-rate-bps>` | Shapes egress traffic to the configured average rate. |
| **Set shaping bursts:**<br>`(config-pmap-c)#shape average <mean-rate-bps> <bc-bytes> <be-bytes>` | Configures average shaping with explicit burst values. |
| **Configure peak shaping:**<br>`(config-pmap-c)#shape peak <mean-rate-bps> <bc-bytes> <be-bytes>` | Configures peak-rate shaping with explicit burst values. |
| **Create hierarchical parent:**<br>`(config)#policy-map <parent-policy-name>`<br>&nbsp;&nbsp;○ `(config-pmap)#class class-default`<br>&nbsp;&nbsp;○ `(config-pmap-c)#shape average <mean-rate-bps>`<br>&nbsp;&nbsp;○ `(config-pmap-c)#service-policy <child-policy-name>` | Shapes parent traffic and invokes the child QoS policy. |
| **Apply hierarchical policy:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#service-policy output <parent-policy-name>` | Applies the hierarchical shaping policy outbound. |

**CCNP Enterprise — IOS-XE — QoS Verification**

| Command | Description |
|---|---|
| **Show all class maps:**<br>`#show class-map` | Displays configured class maps and match criteria. |
| **Show one class map:**<br>`#show class-map <class-map-name>` | Displays match criteria for the specified class map. |
| **Show all policy maps:**<br>`#show policy-map` | Displays configured QoS policy maps and actions. |
| **Show one policy map:**<br>`#show policy-map <policy-map-name>` | Displays actions and parameters for one policy map. |
| **Show interface policies:**<br>`#show policy-map interface` | Displays attached service policies and runtime counters. |
| **Show interface QoS counters:**<br>`#show policy-map interface <interface-id>` | Displays class counters, rates, drops, and QoS actions. |


</div>
