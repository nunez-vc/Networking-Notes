# Hot Standby Router Protocol (HSRP)

> **Core idea:** HSRP provides a resilient default gateway by allowing multiple routers or multilayer switches to share one **virtual IP address** and **virtual MAC address**. One device forwards traffic as **Active** while another is ready to take over as **Standby**.

---

## 1. What It Is

**HSRP (Hot Standby Router Protocol)** is a Cisco first-hop redundancy protocol that provides default-gateway redundancy for hosts on the same subnet or VLAN.

Hosts use an HSRP **Virtual IP (VIP)** as their default gateway instead of the physical IP address of a specific router.

```text
Hosts
  |
  | Default Gateway = 172.16.10.1
  v
+-----------------------------+
| HSRP Virtual Gateway        |
| VIP  = 172.16.10.1          |
| VMAC = HSRP virtual MAC     |
+-----------------------------+
       |               |
       v               v
   SW1 Active      SW2 Standby
   .2              .3
```

---

## 2. How It Works

### Virtual Gateway

All devices participating in the same HSRP group agree on a shared:

```text
Virtual IP address
Virtual MAC address
```

The **Active** device owns and forwards traffic for the virtual gateway.

```text
Host ARP table:

172.16.10.1 -> HSRP virtual MAC
```

The host does not need to know which physical router is currently Active.

---

### Active and Standby Roles

A normal HSRP group has:

```text
Active
= Forwards traffic sent to the virtual gateway

Standby
= Ready to become Active if the current Active device fails
```

HSRP can contain additional routers, but only one router is Active and one is the selected Standby for a group at a given time.

HSRP devices exchange multicast UDP-based control messages to maintain group state and detect failure.

Default timers commonly used on Cisco IOS/IOS XE are:

```text
Hello = 3 seconds
Hold  = 10 seconds
```

If the Standby stops hearing from the Active router for the Hold interval, it can assume the Active role.

---

### Failover

Normal operation:

```text
Hosts
  |
  v
VIP / VMAC
  |
  v
SW1 = Active
SW2 = Standby
```

If SW1 fails:

```text
SW1 fails
   ↓
SW2 detects loss of Active router
   ↓
SW2 becomes Active
   ↓
SW2 assumes the VIP and virtual MAC
   ↓
Hosts continue using the same default gateway
```

The host's configured default gateway does not change.

The important principle is:

> **The virtual IP and virtual MAC move logically with the Active HSRP role.**

---

### Election and Priority

HSRP chooses the preferred Active router using **priority**.

```text
Higher priority = More preferred
Default priority = 100
```

Example:

```text
SW1 priority = 150
SW2 priority = 100

SW1 preferred
```

If priorities are equal, the router with the **higher interface IP address on the HSRP segment** wins the election.

---

### Preemption

A higher-priority router does **not automatically reclaim the Active role after it recovers**.

Preemption is disabled by default.

Without preemption:

```text
SW1 priority 150 = Active
        ↓
SW1 fails
        ↓
SW2 priority 100 = Active
        ↓
SW1 recovers
        ↓
SW2 remains Active
```

With preemption enabled on SW1:

```text
SW1 recovers
     ↓
SW1 has higher priority
     ↓
SW1 preempts SW2
     ↓
SW1 becomes Active again
```

> **Priority determines preference. Preemption determines whether a better router can take the Active role from an already-Active router.**

---

### Object Tracking

Basic HSRP detects failure of the HSRP-facing device or interface, but that alone may not detect loss of upstream connectivity.

Example:

```text
                    WAN
                     X
                     |
Hosts ---- SW1 Active
             |
             +---- HSRP interface still up
```

SW1 may remain Active even though its upstream path has failed.

HSRP can track an interface, route, or other tracked object and reduce its priority when that object fails.

```text
SW1 priority = 150
Track decrement = 60

Normal:
150

Tracked object fails:
150 - 60 = 90

SW2 priority = 100
```

If the design uses preemption appropriately, SW2 can then become Active.

```text
Upstream failure
      ↓
Tracked object goes Down
      ↓
HSRP priority decreases
      ↓
Standby becomes more preferred
      ↓
HSRP role changes
```

> Tracking should monitor the failure condition that actually determines whether the gateway can still provide useful forwarding—not merely an unrelated interface state.

---

### HSRP Versions

Cisco supports **HSRP Version 1 and Version 2**.

| Feature | HSRPv1 | HSRPv2 |
|---|---|---|
| Group range | `0-255` | `0-4095` |
| IPv4 multicast | `224.0.0.2` | `224.0.0.102` |
| Virtual MAC format | `0000.0C07.ACxx` | `0000.0C9F.Fxxx` |
| Millisecond timers | No | Yes |
| IPv6 support | No | Yes |

Routers participating in the same HSRP group must use a compatible HSRP version.

For modern deployments that require the larger group range, subsecond timers, or IPv6 HSRP, **HSRPv2** is required.

---

## 3. Why and When It Is Used

Without an FHRP, hosts normally point to one physical router as their default gateway:

```text
Host
Default Gateway = 172.16.10.2
                     |
                     v
                    SW1
```

If SW1 fails, the host still points to `172.16.10.2`, so off-subnet connectivity fails.

HSRP removes that single gateway dependency:

```text
Host
Default Gateway = 172.16.10.1 VIP
                  /          \
                 /            \
          SW1 Active       SW2 Standby
```

HSRP is appropriate when:

- Hosts require a redundant IPv4 or IPv6 default gateway.
- Two or more Cisco Layer 3 devices serve the same VLAN/subnet.
- Gateway failover should occur without changing endpoint configuration.
- The design requires deterministic Active/Standby gateway placement.

HSRP is unnecessary when the first hop is already provided by another redundancy mechanism or when only one Layer 3 gateway exists and redundancy is not required.

---

## 4. Key Configuration, Parameters, or CLI

### Cisco IOS / IOS XE — Basic IPv4 HSRP

Example for VLAN 10:

```text
VIP: 172.16.10.1
SW1: 172.16.10.2
SW2: 172.16.10.3
HSRP Group: 10
```

### SW1 — Preferred Active

```cisco
interface Vlan10
 ip address 172.16.10.2 255.255.255.0
 standby 10 ip 172.16.10.1
 standby 10 priority 150
 standby 10 preempt
```

### SW2 — Standby

```cisco
interface Vlan10
 ip address 172.16.10.3 255.255.255.0
 standby 10 ip 172.16.10.1
 standby 10 priority 100
 standby 10 preempt
```

---

### HSRPv2 — IOS / IOS XE

The HSRP version is configured per interface:

```cisco
interface Vlan10
 standby version 2
 standby 10 ip 172.16.10.1
```

Both peers should use the same version.

---

### Object Tracking — IOS / IOS XE

Track an upstream interface:

```cisco
track 10 interface GigabitEthernet1/0/48 line-protocol
```

Tie the tracked object to HSRP:

```cisco
interface Vlan10
 standby 10 priority 150
 standby 10 preempt
 standby 10 track 10 decrement 60
```

The decrement must be large enough to make the router less preferred than the intended backup when the tracked object fails.

---

### Verification — IOS / IOS XE

Quick status:

```cisco
show standby brief
```

Detailed state:

```cisco
show standby
```

Verify tracking:

```cisco
show track
```

Useful checks:

```text
HSRP version
HSRP group
Virtual IP
Active router
Standby router
Priority
Preemption
Hello / Hold timers
Virtual MAC
Tracked-object state
```

A practical troubleshooting sequence is:

```text
SVI/interface Up?
      ↓
Peers in same subnet/VLAN?
      ↓
Same HSRP group and version?
      ↓
Same VIP?
      ↓
Active/Standby roles correct?
      ↓
Priority/preemption correct?
      ↓
Tracking state correct?
      ↓
Host ARP points to virtual MAC?
```

---

## 5. Common Gotchas and Misconceptions

### Higher Priority Always Becomes Active

**Incorrect.**

A higher-priority router that returns after another router has already become Active will not automatically take over unless **preemption is enabled**.

```text
Priority = Preference
Preempt  = Permission to reclaim Active role
```

---

### HSRP Automatically Detects Every Upstream Failure

**Incorrect.**

HSRP Hello messages primarily verify HSRP peer availability on the local segment.

An Active gateway can remain alive while its upstream path is broken.

Use meaningful **object tracking** when upstream reachability should influence HSRP ownership.

---

### Tracking an Interface Always Proves Internet/WAN Reachability

**Incorrect.**

```text
Local interface = Up
Remote path      = Failed
```

Interface tracking cannot detect failures beyond that interface.

Where the design requires end-to-end or next-hop reachability detection, use an appropriate route/reachability tracking mechanism supported by the platform.

---

### Tracking Alone Guarantees Failover

**Incorrect.**

Tracking normally reduces HSRP priority. The resulting priorities and preemption behavior must actually cause another router to become preferred.

Example:

```text
Active priority      = 150
Track decrement      = 20
New priority         = 130
Standby priority     = 100

130 > 100
```

No role change occurs because the Active router is still more preferred.

---

### HSRP Load Balances Traffic Within One Group

**Incorrect.**

One HSRP group is fundamentally **Active/Standby**.

Load distribution can be achieved by making different routers Active for different VLANs or HSRP groups:

```text
VLAN 10 → SW1 Active
VLAN 20 → SW2 Active
```

This is load distribution across groups, not simultaneous forwarding by multiple routers for one HSRP virtual gateway.

---

### HSRP Replaces Routing

**Incorrect.**

HSRP protects the **first-hop default gateway**. Each Active router still needs valid routing toward destination networks.

```text
HSRP working
+
No upstream route
=
Traffic still fails
```

---

### HSRP Automatically Makes the Entire Network Redundant

**Incorrect.**

HSRP protects only the gateway function represented by the virtual IP.

End-to-end availability still depends on:

```text
Layer 2 path
Routing
WAN/uplink availability
Firewalls
NAT
Applications
Other upstream dependencies
```

---

### HSRPv1 and HSRPv2 Can Be Mixed in the Same Group

**Incorrect.**

Peers using incompatible HSRP versions do not form the expected HSRP relationship.

Verify:

```cisco
show standby
```

---

## 6. Trade-Offs

### Best Practice

- Use a **virtual IP** as the endpoint default gateway.
- Configure deterministic priorities so the intended device is Active.
- Use preemption when the preferred topology should be restored after recovery.
- Track meaningful upstream reachability when gateway usefulness depends on resources beyond the local VLAN.
- Make the priority decrement large enough to produce the intended failover.
- Align HSRP Active placement with the Layer 2 forwarding topology when that reduces unnecessary traffic paths.

---

### Context-Dependent Trade-Off — Preemption

**Enabled**

```text
+ Restores the preferred gateway after recovery
+ Maintains deterministic traffic paths
- Can introduce additional role changes during unstable recovery
```

**Disabled**

```text
+ Avoids unnecessary failback
+ Can provide greater role stability
- A less-preferred router may remain Active after the preferred router recovers
```

Where recovery is slow or dependencies need time to stabilize, delayed preemption may be appropriate if supported by the platform/release.

---

### Context-Dependent Trade-Off — Timers

Shorter timers can improve failure detection:

```text
Short timers
= Faster failover
```

but overly aggressive timers can increase sensitivity to CPU load, congestion, packet loss, or transient control-plane delays.

Do not reduce timers simply for a lower convergence number without validating the platform and production conditions.

---

### Incorrect or Unsafe

- Enabling preemption without considering gateway churn during unstable recovery.
- Using tracking that does not represent actual upstream forwarding capability.
- Configuring a decrement that never makes the backup router preferred.
- Running inconsistent HSRP versions or VIPs within the same intended group.
- Treating HSRP as a replacement for dynamic routing, firewall HA, or end-to-end path redundancy.

---

## Quick Reference

```text
HSRP
= Cisco first-hop redundancy protocol

Model
= Active / Standby

Host Default Gateway
= Virtual IP (VIP)

Active Router
= Owns virtual gateway and forwards traffic

Standby Router
= Takes over after Active failure

Default Priority
= 100

Higher Priority
= More preferred

Preemption
= Disabled by default

Preempt
= Allows a better router to reclaim Active role

Default Timers
= Hello 3 sec
= Hold 10 sec

Tracking
= Adjusts priority based on another object's state

HSRPv1
= Groups 0-255
= 224.0.0.2
= 0000.0C07.ACxx

HSRPv2
= Groups 0-4095
= 224.0.0.102
= 0000.0C9F.Fxxx
= Millisecond timers
= IPv6 support

HSRP
≠ Routing
≠ Load balancing within one group
≠ Complete end-to-end redundancy

```

## CCNA Configuration

HSRP configuration is CCNP Enterprise scope; current CCNA covers FHRP operation only.

## CCNP Configuration

**CCNP Enterprise — IOS-XE 17.x — HSRP Base Configuration**

| Command | Description |
|---|---|
| **Configure HSRP version:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby version <1|2>` | Selects HSRP version for all groups on the interface. |
| **Configure virtual IPv4 address:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby <group-id> ip <virtual-ip>` | Activates the HSRP group with a virtual IPv4 address. |
| **Add secondary virtual address:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby <group-id> ip <virtual-ip> secondary` | Adds a secondary virtual IPv4 address to the group. |
| **Assign group name:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby <group-id> name <group-name>` | Assigns an administrative name to the HSRP group. |
| **Set virtual MAC address:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby <group-id> mac-address <mac-address>` | Manually assigns the HSRP virtual MAC address. |

**CCNP Enterprise — IOS-XE 17.x — HSRP Priority and Preemption**

| Command | Description |
|---|---|
| **Set HSRP priority:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby <group-id> priority <priority>` | Sets the local HSRP election priority. |
| **Enable preemption:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby <group-id> preempt` | Enables the higher-priority router to reclaim active state. |
| **Set preemption delay:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby <group-id> preempt delay [minimum <seconds>] [reload <seconds>] [sync <seconds>]` | Configures minimum, reload, or synchronization preemption delays. |
| **Disable preemption:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no standby <group-id> preempt` | Disables preemption for the selected HSRP group. |

**CCNP Enterprise — IOS-XE 17.x — HSRP Timers and Initialization**

| Command | Description |
|---|---|
| **Set second-based timers:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby <group-id> timers <hello-seconds> <hold-seconds>` | Sets HSRP hello and hold timers in seconds. |
| **Set millisecond timers:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby <group-id> timers msec <hello-ms> msec <hold-ms>` | Sets HSRP hello and hold timers in milliseconds. |
| **Delay HSRP initialization:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby delay minimum <min-seconds> reload <reload-seconds>` | Delays HSRP initialization after link-up and device reload. |
| **Verify initialization delay:**<br>`#show standby delay [<interface-type> <interface-number>]` | Displays configured HSRP initialization delay values. |

**CCNP Enterprise — IOS-XE 17.x — HSRP Object Tracking**

| Command | Description |
|---|---|
| **Track interface line protocol:**<br>`(config)#track <object-id> interface <interface-id> line-protocol` | Tracks the selected interface line-protocol state. |
| **Track IPv4 route reachability:**<br>`(config)#track <object-id> ip route <prefix>/<prefix-length> reachability` | Tracks reachability of the selected IPv4 route. |
| **Track with priority decrement:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby <group-id> track <object-id> decrement <decrement-value>` | Decrements HSRP priority when the tracked object fails. |
| **Track with group shutdown:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby <group-id> track <object-id> shutdown` | Shuts the HSRP group when the tracked object fails. |
| **Remove tracked object:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no standby <group-id> track <object-id>` | Removes object tracking from the HSRP group. |
| **Show tracked objects:**<br>`#show track [<object-id>]` | Displays tracked-object state and status. |

**CCNP Enterprise — IOS-XE 17.x — HSRP Text Authentication**

| Command | Description |
|---|---|
| **Configure text authentication:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby <group-id> authentication text <string>` | Configures plaintext HSRP authentication for the group. |

**CCNP Enterprise — IOS-XE 17.x — HSRP MD5 Key-String Authentication**

| Command | Description |
|---|---|
| **Configure MD5 key string:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby <group-id> authentication md5 key-string [0|7] <key> [timeout <seconds>]` | Configures HSRP MD5 authentication using a direct key string. |

**CCNP Enterprise — IOS-XE 17.x — HSRP MD5 Key-Chain Authentication**

| Command | Description |
|---|---|
| **Create key chain:**<br>`(config)#key chain <key-chain-name>` | Creates a key chain for HSRP authentication. |
| **Create key:**<br>`(config-keychain)#key <key-id>`<br>&nbsp;&nbsp;○ `(config-keychain-key)#key-string <key-string>` | Creates a key and assigns its authentication string. |
| **Apply MD5 key chain:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby <group-id> authentication md5 key-chain <key-chain-name>` | Applies the configured MD5 key chain to HSRP. |
| **Show key chains:**<br>`#show key chain` | Displays configured key-chain and key information. |

**CCNP Enterprise — IOS-XE 17.x — HSRP for IPv6**

| Command | Description |
|---|---|
| **Enable IPv6 forwarding:**<br>`(config)#ipv6 unicast-routing` | Enables IPv6 unicast routing required for HSRP IPv6. |
| **Enable HSRPv2:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby version 2` | Enables HSRPv2 required for IPv6 HSRP operation. |
| **Autoconfigure virtual link-local address:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby <group-id> ipv6 autoconfig` | Activates IPv6 HSRP with an autogenerated link-local address. |
| **Set virtual link-local address:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby <group-id> ipv6 <link-local-address>` | Activates IPv6 HSRP with a specified link-local address. |
| **Set IPv6 HSRP priority:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby <group-id> priority <priority>` | Sets the IPv6 HSRP election priority. |
| **Enable IPv6 HSRP preemption:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#standby <group-id> preempt` | Enables preemption for the IPv6 HSRP group. |
| **Verify IPv6 interface:**<br>`#show ipv6 interface <interface-type> <interface-number>` | Displays IPv6 interface addressing and operational state. |

**CCNP Enterprise — IOS-XE 17.x — HSRP Verification**

| Command | Description |
|---|---|
| **Show HSRP state:**<br>`#show standby` | Displays detailed state for configured HSRP groups. |
| **Show HSRP summary:**<br>`#show standby brief` | Displays summarized HSRP group and role information. |
| **Show all HSRP groups:**<br>`#show standby all` | Displays configured and learned HSRP groups. |
| **Show interface HSRP state:**<br>`#show standby <interface-type> <interface-number>` | Displays HSRP information for the selected interface. |
| **Show specific HSRP group:**<br>`#show standby <interface-type> <interface-number> <group-id>` | Displays HSRP information for one interface group. |
| **Show specific group summary:**<br>`#show standby <interface-type> <interface-number> <group-id> brief` | Displays summarized state for one HSRP group. |


