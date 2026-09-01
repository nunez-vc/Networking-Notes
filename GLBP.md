<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# Gateway Load Balancing Protocol (GLBP)

> **Core idea:** GLBP is a Cisco first-hop redundancy protocol that provides **default-gateway redundancy and load sharing within the same subnet**. Hosts use one virtual IP address, while GLBP can distribute them across multiple active forwarding routers by returning different virtual MAC addresses.

---

## 1. What It Is

**GLBP (Gateway Load Balancing Protocol)** is a Cisco first-hop redundancy protocol that combines a resilient virtual default gateway with multiple active forwarding paths.

Unlike HSRP or VRRP, which normally forward through one active/master gateway per group, GLBP can use **multiple Active Virtual Forwarders (AVFs)** at the same time.

```text
                 Virtual IP
                172.16.30.1
                     |
            +--------+--------+
            |                 |
          SW1               SW2
        AVG + AVF            AVF
        VMAC-1              VMAC-2
```

---

## 2. How It Works

### Virtual Gateway

All hosts configure the same GLBP virtual IP as their default gateway:

```text
Default Gateway = 172.16.30.1
```

GLBP separates gateway control from packet forwarding using two roles:

| Role | Function |
|---|---|
| **AVG — Active Virtual Gateway** | Controls the virtual IP and answers ARP requests for it |
| **AVF — Active Virtual Forwarder** | Owns a virtual MAC address and forwards traffic from hosts assigned to that MAC |

A router can be both the **AVG and an AVF**.

GLBP supports **one AVG and up to four active AVFs per group**.

---

### ARP-Based Load Sharing

The AVG controls which AVF a host uses by choosing the virtual MAC address placed in the ARP reply.

Example:

```text
Host A:
ARP: Who has 172.16.30.1?
        ↓
AVG replies with VMAC-1
        ↓
Host A sends gateway traffic to SW1
```

Another host can receive a different answer:

```text
Host B:
ARP: Who has 172.16.30.1?
        ↓
AVG replies with VMAC-2
        ↓
Host B sends gateway traffic to SW2
```

Therefore:

```text
Same default-gateway IP
        ↓
Different virtual MAC addresses
        ↓
Different forwarding routers
```

The load distribution occurs primarily when hosts resolve the virtual gateway through ARP.

---

### Control Plane vs Data Plane

```text
AVG
= Controls the VIP and AVF assignment

AVFs
= Forward user traffic
```

This means GLBP is not simply "multiple active gateways."

There is still:

```text
One Active Virtual Gateway
+
Multiple Active Virtual Forwarders
```

The AVG may also be one of the AVFs.

---

### AVG Election

GLBP elects an AVG for each group.

```text
Higher GLBP priority
= More preferred AVG

Default priority
= 100
```

AVG preemption is not enabled by default. A recovered router with a better priority does not automatically reclaim the AVG role unless preemption is configured.

```text
Higher priority
= Preference

Preempt
= Permission to reclaim AVG role
```

---

### AVF Operation

Each AVF is assigned a unique GLBP virtual MAC address.

Example from group 30:

```text
AVF 1 → 0007.b400.1e01
AVF 2 → 0007.b400.1e02
```

Hosts send Ethernet frames to the virtual MAC they learned from the AVG.

```text
Host
  |
  | DST MAC = AVF virtual MAC
  v
AVF
  |
  v
Normal Layer 3 routing
```

---

### AVF Failure

If an AVF fails, another GLBP router can assume responsibility for the failed AVF's virtual MAC address.

```text
AVF-2 fails
    ↓
Another GLBP router takes over VMAC-2
    ↓
Existing hosts using VMAC-2 can continue forwarding
```

The AVG also stops assigning new hosts to an unavailable forwarder and uses the remaining available AVFs.

This allows GLBP to preserve gateway availability while also maintaining forwarding load distribution.

---

### GLBP Timers

Common default timers are:

```text
Hello = 3 seconds
Hold  = 10 seconds
```

GLBP uses multicast address:

```text
224.0.0.102
```

Timer tuning can reduce failure-detection time, but it should be validated against platform performance and Layer 2/routing convergence.

---

### Load-Balancing Methods

GLBP supports three load-balancing methods.

#### Round Robin — Default

The AVG cycles through available AVF virtual MAC addresses when answering ARP requests.

```text
Host A → AVF-1
Host B → AVF-2
Host C → AVF-1
Host D → AVF-2
```

---

#### Weighted

Routers receive traffic in proportion to configured GLBP weights.

Example:

```text
SW1 weight = 80
SW2 weight = 20

Approximate distribution:
SW1 → 80%
SW2 → 20%
```

Useful when forwarding devices have different capacities.

---

#### Host-Dependent

The AVG uses the host MAC address to consistently map a host to an AVF while the set of forwarders remains unchanged.

```text
Host A → AVF-1
Host A → AVF-1
Host A → AVF-1
```

Useful when greater per-host forwarding consistency is preferred.

---

### Priority vs Weighting

These values control different GLBP functions:

```text
Priority
= AVG election

Weighting
= AVF participation / weighted load distribution
```

Do not treat GLBP priority and weighting as interchangeable.

---

## 3. Why and When It Is Used

GLBP solves two first-hop problems at the same time:

```text
Default-gateway redundancy
+
Utilization of multiple forwarding routers
```

Use GLBP when:

- Multiple Cisco Layer 3 devices serve the same subnet or VLAN.
- Hosts require a resilient virtual default gateway.
- Multiple first-hop routers should actively forward traffic within the same GLBP group.
- Per-host gateway load sharing is useful.
- A Cisco-only FHRP is acceptable.

GLBP is generally unnecessary when:

- Only gateway redundancy is required and HSRP/VRRP is simpler.
- The design already distributes traffic effectively across VLANs using separate HSRP/VRRP groups.
- Routed access or an anycast/distributed gateway removes the need for a traditional FHRP.
- Multivendor FHRP interoperability is required.

---

## 4. Key Configuration, Parameters, or CLI

> **Platform:** Cisco IOS / IOS XE.  
> GLBP support is platform- and software-dependent. Confirm support in the applicable Cisco feature/configuration documentation before deployment.

---

### Basic GLBP

Topology:

```text
VIP: 172.16.30.1

SW1: 172.16.30.2
SW2: 172.16.30.3

GLBP Group: 30
```

### SW1

```cisco
interface Vlan30
 ip address 172.16.30.2 255.255.255.0
 glbp 30 ip 172.16.30.1
 glbp 30 priority 110
 glbp 30 preempt
```

### SW2

```cisco
interface Vlan30
 ip address 172.16.30.3 255.255.255.0
 glbp 30 ip 172.16.30.1
 glbp 30 priority 100
 glbp 30 preempt
```

The priority affects **AVG election**, not AVF load-sharing ratio.

---

### Load-Balancing Method

#### Round Robin

```cisco
interface Vlan30
 glbp 30 load-balancing round-robin
```

#### Weighted

```cisco
interface Vlan30
 glbp 30 load-balancing weighted
 glbp 30 weighting 80
```

Peer example:

```cisco
interface Vlan30
 glbp 30 load-balancing weighted
 glbp 30 weighting 20
```

#### Host-Dependent

```cisco
interface Vlan30
 glbp 30 load-balancing host-dependent
```

---

### Verification

Quick status:

```cisco
show glbp brief
```

Detailed status:

```cisco
show glbp
```

Verify:

```text
Virtual IP
AVG state
Standby AVG
AVG priority
Preemption
AVF states
AVF virtual MAC addresses
Hello / Hold timers
Weighting
Load-balancing method
```

A practical verification sequence is:

```text
SVI/interface Up?
      ↓
Peers in same VLAN/subnet?
      ↓
Same GLBP group and VIP?
      ↓
Correct AVG elected?
      ↓
Expected AVFs Active?
      ↓
Correct load-balancing method?
      ↓
Hosts learning different VMACs as expected?
      ↓
Each AVF has valid upstream routing?
```

---

## 5. Common Gotchas and Misconceptions

### GLBP Has Multiple Active Gateways

**Partly incorrect.**

GLBP has:

```text
One AVG
+
Multiple AVFs
```

Only one router controls the virtual gateway function, while several routers can actively forward user traffic.

---

### The AVG Must Forward All User Traffic

**Incorrect.**

The AVG controls ARP responses and AVF assignment. User traffic can be forwarded by any active AVF.

The AVG can also be an AVF, but the roles are logically separate.

---

### GLBP Load-Balances Every Packet

**Incorrect.**

GLBP primarily distributes **hosts** across AVFs through ARP replies.

A host normally continues using the virtual MAC already stored in its ARP cache.

```text
GLBP load sharing
≠ Per-packet load balancing
```

---

### Higher Priority Means More User Traffic

**Incorrect.**

```text
Priority
= AVG election

Weighting
= Weighted AVF load sharing
```

Raising GLBP priority does not make a router receive a larger percentage of forwarded host traffic.

---

### Preemption Is Automatically Enabled for the AVG

**Incorrect.**

AVG preemption must be explicitly enabled when the preferred router should reclaim the AVG role after recovery.

```cisco
glbp 30 preempt
```

---

### GLBP Replaces Routing

**Incorrect.**

An AVF still performs normal Layer 3 forwarding and therefore requires valid routes.

```text
GLBP healthy
+
No route upstream
=
Traffic fails
```

---

### GLBP Automatically Detects Every Upstream Failure

**Incorrect.**

GLBP peer health does not by itself prove that an AVF has useful upstream connectivity.

Where upstream failure should influence forwarding eligibility, use an appropriate platform-supported tracking and weighting design and validate the exact syntax for the IOS/IOS XE release in use.

---

### GLBP Is Standards-Based

**Incorrect.**

GLBP is Cisco-specific. For multivendor FHRP requirements, VRRP is generally the standards-based alternative, subject to feature compatibility across vendors.

---

## 6. Trade-Offs

### Best Practice

- Use GLBP only when active use of multiple first-hop gateways provides a real benefit.
- Keep AVG selection deterministic with explicit priority.
- Configure preemption only when restoring the preferred AVG after recovery is desirable.
- Validate that every AVF has equivalent or intentionally designed upstream reachability.
- Use `show glbp` to verify AVG and AVF roles separately.
- Align GLBP forwarding with the Layer 2 topology so traffic does not take unnecessary paths.

---

### Context-Dependent Trade-Off — GLBP vs HSRP / VRRP

**GLBP**

```text
+ Multiple active forwarders in one group
+ Better first-hop link/router utilization
+ Per-host load sharing
- More operational complexity
- Cisco-specific
```

**HSRP / VRRP**

```text
+ Simpler active/standby model
+ Easier troubleshooting
+ Predictable single forwarder per group
- Multiple routers are not active forwarders for the same group
```

Choose GLBP only when its active-forwarder model materially improves the design.

---

### Context-Dependent Trade-Off — Load-Balancing Method

**Round Robin**

```text
+ Simple
+ Default behavior
- Does not account for unequal router capacity
```

**Weighted**

```text
+ Can favor higher-capacity routers
- Requires intentional weight design
```

**Host-Dependent**

```text
+ Consistent AVF selection per host
- Distribution can be uneven if host traffic volumes differ significantly
```

---

### Incorrect or Unsafe

- Treating GLBP as per-packet load balancing.
- Assuming AVG priority controls AVF traffic percentage.
- Deploying GLBP without confirming platform/software support.
- Using aggressive timers without validating control-plane, Layer 2, and routing convergence.
- Assuming GLBP removes the need for redundant upstream routing.
- Using GLBP in a multivendor design without verifying actual protocol support.

---

## Quick Reference

```text
GLBP
= Cisco first-hop redundancy + gateway load sharing

Virtual IP
= One default-gateway IP used by all hosts

AVG
= Active Virtual Gateway
= Controls VIP and ARP replies

AVF
= Active Virtual Forwarder
= Owns a VMAC and forwards host traffic

Per Group
= 1 AVG
= Up to 4 active AVFs

Default Priority
= 100

Priority
= AVG election

Preemption
= Disabled for AVG by default

Default Load Balancing
= Round robin

Other Methods
= Weighted
= Host-dependent

Default Timers
= Hello 3 sec
= Hold 10 sec

IPv4 Multicast
= 224.0.0.102

GLBP Load Sharing
= Primarily per host through ARP

GLBP
≠ Per-packet load balancing
≠ Routing protocol
≠ End-to-end redundancy
≠ Standards-based FHRP
```

GLBP configuration is CCNP Enterprise scope; current CCNA 200-301 v2.0 omits GLBP configuration.

## CCNP Configuration

**CCNP Enterprise — IOS-XE 17.x — GLBP Base Configuration**

| Command | Description |
|---|---|
| **Configure virtual gateway:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#glbp <group-id> ip <virtual-ip>` | Creates the GLBP group and primary virtual IPv4 address. |
| **Add secondary virtual address:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#glbp <group-id> ip <virtual-ip> secondary` | Adds a secondary virtual IPv4 address to the group. |
| **Set gateway priority:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#glbp <group-id> priority <priority>` | Sets the local AVG election priority. |
| **Enable AVG preemption:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#glbp <group-id> preempt` | Enables higher-priority AVG preemption. |
| **Delay AVG preemption:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#glbp <group-id> preempt delay minimum <seconds>` | Delays AVG preemption by the configured interval. |
| **Assign redundancy name:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#glbp <group-id> name <redundancy-name>` | Assigns a redundancy-client name to the GLBP group. |

**CCNP Enterprise — IOS-XE 17.x — GLBP Timers**

| Command | Description |
|---|---|
| **Set hello and hold timers:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#glbp <group-id> timers <hello-seconds> <hold-seconds>` | Sets GLBP hello and hold timers in seconds. |
| **Set millisecond timers:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#glbp <group-id> timers msec <hello-ms> msec <hold-ms>` | Sets GLBP hello and hold timers in milliseconds. |
| **Set redirect timers:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#glbp <group-id> timers redirect <redirect-seconds> <timeout-seconds>` | Sets client redirection and forwarder timeout intervals. |

**CCNP Enterprise — IOS-XE 17.x — GLBP Load Balancing**

| Command | Description |
|---|---|
| **Use round-robin balancing:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#glbp <group-id> load-balancing round-robin` | Selects round-robin GLBP host assignment. |
| **Use weighted balancing:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#glbp <group-id> load-balancing weighted` | Selects weighting-based GLBP host assignment. |
| **Use host-dependent balancing:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#glbp <group-id> load-balancing host-dependent` | Selects host-dependent GLBP host assignment. |
| **Set gateway weighting:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#glbp <group-id> weighting <maximum>` | Sets the initial GLBP gateway weighting value. |
| **Set weighting thresholds:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#glbp <group-id> weighting <maximum> lower <lower> upper <upper>` | Sets initial, lower, and upper forwarding weights. |

**CCNP Enterprise — IOS-XE 17.x — GLBP Object Tracking**

| Command | Description |
|---|---|
| **Track interface line protocol:**<br>`(config)#track <object-id> interface <interface-id> line-protocol` | Tracks the selected interface line-protocol state. |
| **Track interface IP routing:**<br>`(config)#track <object-id> interface <interface-id> ip routing` | Tracks IP-routing readiness on the selected interface. |
| **Apply weighting tracking:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#glbp <group-id> weighting track <object-id> decrement <value>` | Decrements GLBP weighting when the tracked object fails. |
| **Set AVF preemption delay:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#glbp <group-id> forwarder preempt delay minimum <seconds>` | Sets the minimum AVF preemption delay. |
| **Disable AVF preemption:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no glbp <group-id> forwarder preempt` | Disables AVF preemption for the GLBP group. |
| **Verify tracked object:**<br>`#show track <object-id>` | Displays state and status for the tracked object. |

**CCNP Enterprise — IOS-XE 17.x — GLBP Authentication**

| Command | Description |
|---|---|
| **Configure text authentication:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#glbp <group-id> authentication text <string>` | Configures plaintext GLBP group authentication. |
| **Configure MD5 key string:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#glbp <group-id> authentication md5 key-string <key>` | Configures MD5 authentication with a direct key string. |
| **Create authentication key chain:**<br>`(config)#key chain <key-chain-name>`<br>&nbsp;&nbsp;○ `(config-keychain)#key <key-id>`<br>&nbsp;&nbsp;○ `(config-keychain-key)#key-string <key-string>` | Creates a key chain for GLBP MD5 authentication. |
| **Apply MD5 key chain:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#glbp <group-id> authentication md5 key-chain <key-chain-name>` | Applies the configured MD5 key chain to GLBP. |
| **Verify key chain:**<br>`#show key chain` | Displays configured key-chain information. |

**CCNP Enterprise — IOS-XE 17.x — GLBP Verification**

| Command | Description |
|---|---|
| **Show GLBP state:**<br>`#show glbp` | Displays detailed GLBP AVG and AVF state. |
| **Show GLBP summary:**<br>`#show glbp brief` | Displays summarized GLBP groups, AVG, and AVF roles. |
| **Show specific group:**<br>`#show glbp <group-id>` | Displays detailed information for one GLBP group. |


</div>
