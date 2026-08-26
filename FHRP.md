<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# First-Hop Redundancy Protocol (FHRP)

> **Core idea:** FHRP is a **class of gateway-redundancy protocols** that lets multiple Layer 3 devices present a single virtual default gateway to hosts. If the forwarding gateway fails, another device assumes the virtual gateway without requiring hosts to change their default-gateway configuration.

---

## 1. What It Is

**First-Hop Redundancy Protocol (FHRP)** is a family of protocols that provides resilient default-gateway service on a shared subnet or VLAN by using a **virtual IP address (VIP)** and associated virtual MAC behavior.

The main FHRPs commonly encountered in Cisco enterprise networks are:

| Protocol | Type | Forwarding Model |
|---|---|---|
| **HSRP** | Cisco | Active / Standby |
| **VRRP** | Standards-based | Master / Backup |
| **GLBP** | Cisco | One AVG with multiple active forwarders |

> **FHRP protects the first-hop gateway function. It does not replace routing or provide end-to-end redundancy by itself.**

---

## 2. How It Works

### Virtual Default Gateway

Without FHRP, a host normally points to one physical router:

```text
Host
Default Gateway = 172.16.10.2
                     |
                     v
                   Router
```

If that router fails, the host still points to the failed address.

With FHRP:

```text
Host
Default Gateway = 172.16.10.1
                     |
                     v
          +---------------------+
          | Virtual Gateway     |
          | VIP 172.16.10.1     |
          +---------------------+
              /           \
             /             \
         Router A       Router B
```

The hosts keep using the same VIP while the participating routers determine which device currently owns or forwards for the virtual gateway.

---

### Control Plane

FHRP peers exchange protocol messages on the local subnet to:

```text
Discover peers
     ↓
Elect forwarding roles
     ↓
Monitor peer availability
     ↓
React to failures
     ↓
Transfer virtual-gateway responsibility
```

Each FHRP uses its own election rules, terminology, timers, and multicast/control messages.

---

### Data Plane

Hosts resolve the virtual gateway through ARP for IPv4 or the corresponding IPv6 neighbor mechanism when supported.

For HSRP and VRRP:

```text
Host
  |
  | Frame to Virtual MAC
  v
Active / Master Router
  |
  v
Routes packet normally
```

If that router fails, another device assumes the virtual gateway role.

From the endpoint perspective:

```text
Default Gateway IP = unchanged
Gateway identity    = unchanged logically
Physical forwarder  = changed
```

---

### Failover

Generic FHRP failover:

```text
Preferred gateway forwarding
          ↓
Failure detected
          ↓
Backup becomes forwarding gateway
          ↓
Virtual gateway remains available
          ↓
Hosts continue using the same default gateway
```

The failover time depends on:

- Protocol timers
- Failure-detection method
- Object tracking
- Platform/software behavior
- Layer 2 convergence
- Routing convergence beyond the FHRP gateway

---

### Priority and Preemption

Most FHRPs use a priority value to determine which device is preferred.

Conceptually:

```text
Higher Priority
     ↓
More preferred gateway
```

However, **priority and preemption are separate concepts**.

```text
Priority
= Which router is preferred

Preemption
= Whether a better router can take the role from the current forwarding router
```

Examples:

- **HSRP:** preemption is disabled by default.
- **VRRP:** preemption is enabled by default.
- **GLBP:** preemption can be configured for the AVG role.

---

### Object Tracking

Basic FHRP peer detection may confirm that the gateway device is alive while failing to detect that its **upstream path is unusable**.

Example:

```text
Hosts
  |
  v
SW1 = FHRP Forwarder
  |
  | Local SVI still Up
  |
  X Upstream WAN failure
```

Object tracking allows an FHRP device to reduce its priority when a meaningful dependency fails.

Common tracked objects include:

```text
Interface line protocol
Route reachability
Other platform-supported tracked objects
```

Generic behavior:

```text
Tracked object fails
        ↓
Gateway priority decreases
        ↓
Peer becomes more preferred
        ↓
FHRP role changes
```

The priority decrement must be large enough to actually make the intended backup device more preferred.

---

# FHRP Protocols

## HSRP

**Hot Standby Router Protocol (HSRP)** uses an **Active / Standby** model.

```text
Active
= Forwards traffic for the virtual gateway

Standby
= Ready to become Active
```

Key behavior:

```text
Default priority = 100
Higher priority  = preferred
Preemption       = disabled by default
```

HSRP supports:

```text
HSRPv1
HSRPv2
```

HSRPv2 provides a larger group range, millisecond timer support, and IPv6 support compared with HSRPv1.

---

## VRRP

**Virtual Router Redundancy Protocol (VRRP)** provides similar gateway redundancy using:

```text
Master
= Current forwarding gateway

Backup
= Ready to become Master
```

Important differences from HSRP:

```text
Preemption = enabled by default
IPv4 multicast = 224.0.0.18
Virtual MAC = 0000.5E00.01xx
```

Versions:

```text
VRRPv2 → IPv4
VRRPv3 → IPv4 and IPv6
```

VRRP is the standards-based choice when multivendor FHRP interoperability is required, subject to platform implementation support.

---

## GLBP

**Gateway Load Balancing Protocol (GLBP)** provides gateway redundancy while allowing multiple routers to actively forward traffic for the same virtual gateway.

GLBP uses two roles:

### Active Virtual Gateway (AVG)

The AVG:

```text
Owns responsibility for the VIP
Responds to ARP requests for the VIP
Assigns hosts to virtual forwarders
```

### Active Virtual Forwarder (AVF)

An AVF:

```text
Owns a virtual MAC
Receives traffic from assigned hosts
Routes that traffic normally
```

Conceptually:

```text
                  VIP
                   |
                   v
              GLBP AVG
             /        \
        VMAC-1        VMAC-2
          |              |
          v              v
        AVF-1          AVF-2
```

The AVG can return different virtual MAC addresses to different hosts, allowing multiple routers to forward traffic simultaneously.

GLBP supports these load-balancing methods:

```text
Round-robin
Weighted
Host-dependent
```

Default:

```text
Round-robin
```

---

## Protocol Comparison

| Feature | HSRP | VRRP | GLBP |
|---|---|---|---|
| **Gateway role** | Active / Standby | Master / Backup | AVG + AVFs |
| **Multiple active forwarders per group** | No | No | Yes |
| **Preemption default** | Disabled | Enabled | Configurable |
| **Standards-based** | No | Yes | No |
| **Gateway load sharing** | Across separate groups/VLANs | Across separate groups/VLANs | Within one GLBP group |
| **Object tracking** | Yes | Yes | Yes |

With HSRP or VRRP, traffic can still be distributed across two distribution switches by making different devices preferred for different VLANs:

```text
VLAN 10 → SW1 preferred
VLAN 20 → SW2 preferred
```

That is **load distribution across FHRP groups**, not simultaneous forwarding for one virtual gateway.

---

## 3. Why and When It Is Used

FHRP solves the single-default-gateway failure problem.

Use FHRP when:

- Two or more Layer 3 devices provide gateway service for the same VLAN/subnet.
- Hosts need gateway redundancy without changing their IP configuration.
- Gateway failover must be transparent to endpoints.
- A campus or data-center design retains Layer 2 adjacency to redundant gateway devices.
- Upstream failures should influence gateway ownership through object tracking.

FHRP is generally unnecessary when:

- Only one gateway exists and redundancy is not required.
- The access layer itself performs routing and uses routed links upstream.
- Another architecture already provides distributed or anycast gateway redundancy.

---

## 4. Key Configuration, Parameters, or CLI

> The examples below are **Cisco IOS / IOS XE** syntax. Feature availability and exact syntax can vary by platform and release. Verify support in the applicable Cisco configuration guide before production deployment.

---

### Cisco IOS / IOS XE — HSRP

```cisco
interface Vlan10
 ip address 172.16.10.2 255.255.255.0
 standby 10 ip 172.16.10.1
 standby 10 priority 110
 standby 10 preempt
```

Verify:

```cisco
show standby brief
show standby
```

---

### Cisco IOS XE — VRRPv3

VRRPv3 hierarchical configuration:

```cisco
fhrp version vrrp v3

interface Vlan20
 ip address 172.16.20.2 255.255.255.0
 vrrp 20 address-family ipv4
  address 172.16.20.1
  priority 110
```

Verify:

```cisco
show vrrp brief
show vrrp
```

VRRPv2 and VRRPv3 are not directly interchangeable; use the correct version for the platform and design.

---

### Cisco IOS / IOS XE — GLBP

```cisco
interface Vlan30
 ip address 172.16.30.2 255.255.255.0
 glbp 30 ip 172.16.30.1
 glbp 30 priority 110
 glbp 30 preempt
```

Verify:

```cisco
show glbp brief
show glbp
```

Optional GLBP load-balancing method:

```cisco
interface Vlan30
 glbp 30 load-balancing round-robin
```

Other supported methods:

```text
weighted
host-dependent
```

---

### Object Tracking — IOS / IOS XE

Track an interface:

```cisco
track 10 interface GigabitEthernet1/0/48 line-protocol
```

Track route reachability:

```cisco
track 20 ip route 192.0.2.0/24 reachability
```

Verify:

```cisco
show track
```

Example HSRP integration:

```cisco
interface Vlan10
 standby 10 track 10 decrement 20
```

The same principle can be applied to supported VRRP and GLBP tracking configurations.

---

### Practical Verification Sequence

```text
1. Is the SVI/interface up?
        ↓
2. Are peers in the same VLAN/subnet?
        ↓
3. Do group/VIP/version settings match?
        ↓
4. Which device currently owns the forwarding role?
        ↓
5. Are priority and preemption behaving as intended?
        ↓
6. Are tracked objects Up?
        ↓
7. Does the host resolve the virtual gateway correctly?
        ↓
8. Does the active gateway have valid upstream routing?
```

---

## 5. Common Gotchas and Misconceptions

### FHRP Is a Routing Protocol

**Incorrect.**

FHRP selects the **first-hop gateway**. It does not advertise network prefixes or replace OSPF, EIGRP, BGP, or static routing.

```text
FHRP working
+
No route upstream
=
Traffic still fails
```

---

### FHRP Makes the Entire Network Highly Available

**Incorrect.**

FHRP protects only the gateway function.

End-to-end availability still depends on:

```text
Layer 2 topology
Routing
WAN/uplinks
Firewalls
NAT
Applications
Other dependencies
```

---

### Priority Alone Guarantees the Preferred Router Will Take Over

**Incorrect.**

Preemption behavior matters.

For example:

```text
HSRP
Higher priority + no preempt
= Recovered router may remain Standby
```

VRRP behaves differently because preemption is enabled by default.

---

### Tracking Any Interface Is Enough

**Incorrect.**

Tracking should represent whether the gateway can still provide useful forwarding.

```text
Local interface Up
≠
Remote destination reachable
```

Track the dependency that reflects the real failure domain.

---

### A Priority Decrement Automatically Causes Failover

**Incorrect.**

Example:

```text
Current priority = 150
Decrement        = 20
New priority     = 130
Peer priority    = 100
```

The same router remains preferred.

---

### HSRP and VRRP Load-Balance a Single Gateway Group

**Incorrect.**

HSRP and VRRP normally select one forwarding gateway per group.

To distribute traffic, use different preferred gateways across VLANs/groups.

GLBP is specifically designed to support multiple active forwarders within one group.

---

### Faster FHRP Timers Always Mean a Better Design

**Incorrect.**

Aggressive timers reduce detection time but increase sensitivity to:

```text
CPU load
Packet loss
Congestion
Transient control-plane delay
```

Layer 2 and routing convergence must also support the target failover time.

---

## 6. Trade-Offs

### Best Practice

- Use a VIP as the endpoint default gateway.
- Configure deterministic priority so gateway ownership matches the intended topology.
- Use object tracking for meaningful upstream failures.
- Align FHRP gateway placement with the Layer 2 forwarding path to avoid unnecessary traffic detours.
- Validate the complete failover path, not only the FHRP state.

---

### Context-Dependent Trade-Off — HSRP vs VRRP

**HSRP**

```text
+ Common in Cisco-centric environments
+ Mature Cisco feature integration
- Cisco-specific
```

**VRRP**

```text
+ Standards-based
+ Better fit for multivendor environments
- Exact feature behavior/support varies by vendor implementation
```

Choose based on platform support, interoperability requirements, and operational standards.

---

### Context-Dependent Trade-Off — HSRP/VRRP vs GLBP

**HSRP / VRRP**

```text
+ Simpler active/standby forwarding model
+ Predictable gateway ownership
- Only one forwarding gateway per group
```

**GLBP**

```text
+ Multiple gateways can actively forward
+ Better first-hop uplink utilization
- More operational complexity
- Cisco-specific
- Platform/topology support must be verified
```

---

### Context-Dependent Trade-Off — Preemption

**Enabled**

```text
+ Restores preferred gateway placement
- Can create additional role changes during unstable recovery
```

**Disabled**

```text
+ Avoids unnecessary failback
- Less-preferred device may remain active after recovery
```

---

### Incorrect or Unsafe

- Treating FHRP as a replacement for routing or end-to-end HA.
- Using tracking that does not reflect actual forwarding capability.
- Configuring aggressive timers without validating control-plane and Layer 2 behavior.
- Assuming HSRP, VRRP, and GLBP use identical election, preemption, or version behavior.
- Deploying an FHRP without validating STP/L2 path alignment in a Layer 2 access design.

---

## Quick Reference

```text
FHRP
= First-hop gateway redundancy

Virtual IP (VIP)
= Default gateway configured on hosts

Virtual MAC
= Layer 2 identity associated with the virtual gateway

HSRP
= Cisco
= Active / Standby
= Preemption disabled by default

VRRP
= Standards-based
= Master / Backup
= Preemption enabled by default

GLBP
= Cisco
= AVG + multiple AVFs
= Gateway redundancy + load balancing

Object Tracking
= Changes gateway preference when a dependency fails

FHRP
≠ Routing protocol
≠ End-to-end redundancy
≠ Firewall HA
```

</div>
