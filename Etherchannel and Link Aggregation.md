<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# EtherChannel and Link Aggregation

> **Core idea:** Link aggregation combines multiple physical Ethernet links into one logical interface. On Cisco platforms this logical bundle is commonly called an **EtherChannel** or **Port-Channel**, providing higher aggregate bandwidth, link redundancy, and a single logical interface for Layer 2 or Layer 3 forwarding.

---

## 1. What It Is

**Link aggregation** is the technique of bundling multiple physical Ethernet interfaces so they operate as one logical link. **EtherChannel** is Cisco's implementation/terminology for that bundle, represented by a **Port-Channel** interface.

The bundle can operate as:

```text
Layer 2 EtherChannel
= One logical switchport/trunk

Layer 3 EtherChannel
= One logical routed interface
```

> The physical links remain separate transmission paths, but protocols such as STP and routing normally treat the Port-Channel as one logical interface.

---

## 2. How It Works

## Logical Bundle

Assume four physical links connect two switches:

```text
SW1                              SW2

Gi1/0/1 ---------------------- Gi1/0/1
Gi1/0/2 ---------------------- Gi1/0/2
Gi1/0/3 ---------------------- Gi1/0/3
Gi1/0/4 ---------------------- Gi1/0/4
      \                          /
       \                        /
        +---- Port-Channel1 ----+
```

Instead of operating as four independent Layer 2 links, the interfaces become members of one logical Port-Channel:

```text
Po1
= Gi1/0/1
+ Gi1/0/2
+ Gi1/0/3
+ Gi1/0/4
```

The logical interface carries the configuration that applies to the bundle.

---

## Negotiation Methods

EtherChannel can be formed using:

```text
LACP
PAgP
Static mode
```

---

## LACP

**LACP (Link Aggregation Control Protocol)** is the standards-based protocol defined by IEEE 802.1AX.

Cisco LACP modes:

```text
active
= Actively sends LACP packets

passive
= Responds to LACP packets
```

Valid combinations:

```text
active  ↔ active   = Forms
active  ↔ passive  = Forms
passive ↔ passive  = Does not form
```

Memory:

```text
At least one side must be active.
```

LACP verifies that candidate links are compatible and belong to the same logical aggregation.

---

## PAgP

**PAgP (Port Aggregation Protocol)** is Cisco proprietary.

Cisco PAgP modes:

```text
desirable
= Actively negotiates

auto
= Responds to negotiation
```

Valid combinations:

```text
desirable ↔ desirable = Forms
desirable ↔ auto      = Forms
auto      ↔ auto      = Does not form
```

PAgP is mainly relevant in Cisco-only environments and legacy designs.

---

## Static EtherChannel

Static mode uses:

```text
channel-group <number> mode on
```

No negotiation protocol validates the far end.

```text
on ↔ on
= Can form if both sides are correctly configured
```

But because no protocol checks compatibility:

```text
Misconfiguration
→ Can create forwarding problems or loops
```

> LACP is generally preferred when both ends support it because it provides negotiation and member validation.

---

## Member-Link Consistency

Physical interfaces in one EtherChannel must have compatible operational parameters.

For Layer 2 bundles, important settings include:

```text
Access vs trunk mode
Access VLAN
Native VLAN
Allowed VLAN list
Speed
Duplex
Other switchport characteristics
```

For Layer 3 bundles:

```text
Member interfaces must be routed ports
IP addressing belongs on the Port-Channel
Physical members normally have no individual IP addresses
```

If member parameters are inconsistent, interfaces may be suspended or fail to join the bundle depending on platform and protocol.

---

## Layer 2 EtherChannel

A Layer 2 Port-Channel behaves like one logical switchport.

Example:

```text
Po1
= 802.1Q trunk
```

Configuration belongs on the Port-Channel:

```text
Allowed VLANs
Native VLAN
STP behavior
Other trunk parameters
```

STP sees:

```text
One logical Port-Channel
```

rather than multiple parallel links.

This allows all member links to carry traffic without STP blocking each physical link individually.

---

## Layer 3 EtherChannel

A Layer 3 Port-Channel behaves like one routed interface.

```text
Router/Switch A
      |
      | Po10
      |
Router/Switch B
```

The IP address is configured on the Port-Channel:

```text
Po10 = 192.0.2.1/30
```

The routing table and routing protocols reference the logical Port-Channel rather than individual member links.

---

## Traffic Distribution

EtherChannel does **not** normally send every packet round-robin across all members.

Instead, the device calculates a hash from selected packet fields.

Possible inputs include:

```text
Source MAC
Destination MAC
Source + destination MAC
Source IP
Destination IP
Source + destination IP
Layer 4 ports on supported platforms/modes
```

Conceptually:

```text
Flow information
      ↓
Hash calculation
      ↓
Select one member link
      ↓
Packets in that flow use that member
```

This preserves packet order within a flow.

---

## Per-Flow Bandwidth

A critical implication:

```text
4 × 1-Gbps links
= 4 Gbps aggregate capacity

But one normal flow
≈ limited to one 1-Gbps member
```

Multiple flows can be distributed across different links:

```text
Flow A → Gi1/0/1
Flow B → Gi1/0/2
Flow C → Gi1/0/3
Flow D → Gi1/0/4
```

Therefore EtherChannel increases **aggregate throughput**, not necessarily the throughput of a single flow.

---

## Hash Imbalance

Hash-based forwarding does not guarantee perfectly even utilization.

Example:

```text
Link 1 = 900 Mbps
Link 2 = 150 Mbps
Link 3 = 100 Mbps
Link 4 = 50 Mbps
```

This can occur even when the EtherChannel is healthy because:

```text
Few large flows
+
Hash result
=
Uneven member utilization
```

Changing the load-balancing algorithm can improve distribution, but only if the traffic patterns benefit from different hash inputs.

---

## Member Failure

If one member link fails:

```text
Po1
Gi1/0/1  Up
Gi1/0/2  Up
Gi1/0/3  Down
Gi1/0/4  Up
```

the Port-Channel can remain operational using the surviving members.

```text
Before:
4 × 1 Gbps

After one member fails:
3 × 1 Gbps aggregate
```

Traffic assigned to the failed member is rehashed onto remaining active links.

The logical interface stays Up as long as the platform's operational requirements for the bundle are still satisfied.

---

## LACP Selection and Standby Members

LACP can support more physical candidate links than are actively forwarding, subject to platform limits.

Conceptually:

```text
Candidate member links
       ↓
LACP selection
       ↓
Active members
+
Standby members if applicable
```

The exact maximum number of active/standby members varies by platform and software release.

Verify platform limits rather than assuming a universal number.

---

## LACP System and Port Priority

LACP uses values such as:

```text
System ID
System priority
Port priority
Port number
```

to determine aggregation behavior when more candidate links exist than can be active.

Lower numerical LACP priority values are more preferred.

In most ordinary two-switch EtherChannels, default priorities are sufficient; adjust them only when deterministic member selection is required.

---

## Minimum Links

Some Cisco platforms support a **minimum-links** feature.

Conceptually:

```text
Configured minimum = 2

Active members:
3 → Port-Channel Up
2 → Port-Channel Up
1 → Port-Channel Down
```

This is useful when keeping the logical interface up with too little remaining capacity would be operationally worse than failing the bundle completely.

Syntax and support vary by platform.

---

## Multi-Chassis EtherChannel

A normal EtherChannel terminates on one logical system at each end.

Technologies such as:

```text
StackWise
StackWise Virtual
VSS
vPC on NX-OS
```

can allow links to terminate on two physical chassis while presenting an appropriate multi-chassis aggregation design to the attached device.

Example:

```text
       SW1 --------\
Server              \____ Port-Channel
       SW2 --------/
```

This provides chassis-level redundancy in addition to member-link redundancy.

> Multi-chassis EtherChannel behavior is platform-specific. Do not assume two independent switches can simply share one Port-Channel without a supported multi-chassis technology.

---

## 3. Why and When It Is Used

EtherChannel/link aggregation solves three practical problems:

```text
More aggregate bandwidth
+
Physical-link redundancy
+
One logical interface instead of multiple parallel links
```

Common uses include:

```text
Switch-to-switch uplinks
Distribution/core links
Server NIC teaming
Firewall/router uplinks
Data-center leaf/spine attachments where supported
Virtualization hosts
```

It is appropriate when:

- Multiple parallel links exist between the same logical endpoints.
- Higher aggregate throughput is required.
- Link failure should not drop the entire logical connection.
- STP should treat parallel Layer 2 links as one logical path.

It is unnecessary when:

```text
One physical link provides sufficient bandwidth/redundancy
The endpoints do not support compatible aggregation
The links terminate on unrelated devices without multi-chassis support
```

---

## 4. Key Configuration, Parameters, or CLI

> **Platform:** Cisco Catalyst IOS / IOS XE unless otherwise noted.

---

## Layer 2 LACP EtherChannel

### SW1

```cisco
interface range GigabitEthernet1/0/1-2
 channel-group 1 mode active
exit

interface Port-channel1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
```

### SW2

```cisco
interface range GigabitEthernet1/0/1-2
 channel-group 1 mode passive
exit

interface Port-channel1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
```

This forms because:

```text
active ↔ passive
```

is a valid LACP combination.

---

## Layer 2 PAgP EtherChannel

```cisco
interface range GigabitEthernet1/0/1-2
 channel-group 1 mode desirable
```

The far end can use:

```text
desirable
or
auto
```

---

## Static EtherChannel

```cisco
interface range GigabitEthernet1/0/1-2
 channel-group 1 mode on
```

Use static mode only when both endpoints are intentionally configured for static aggregation.

---

## Layer 3 LACP EtherChannel

```cisco
interface range GigabitEthernet1/0/1-2
 no switchport
 channel-group 10 mode active
exit

interface Port-channel10
 no switchport
 ip address 192.0.2.1 255.255.255.252
 no shutdown
```

Configure the peer Port-Channel in the same Layer 3 mode with a compatible address.

---

## Load-Balancing Algorithm

View the current algorithm:

```cisco
show etherchannel load-balance
```

On supported IOS/IOS XE platforms, the algorithm can be selected globally with:

```cisco
port-channel load-balance <method>
```

Available methods vary by hardware and software release.

Verify supported options with:

```cisco
port-channel load-balance ?
```

before making a change.

---

## Verification

Primary command:

```cisco
show etherchannel summary
```

Typical flags include:

```text
P = Bundled in Port-Channel
I = Stand-alone
s = Suspended
D = Down
S = Layer 2
R = Layer 3
```

Also verify:

```cisco
show interfaces port-channel <number>
show interfaces trunk
show interfaces etherchannel
show lacp neighbor
show pagp neighbor
show etherchannel load-balance
```

For member consistency:

```cisco
show running-config interface <member-interface>
show running-config interface port-channel <number>
```

---

## Practical Troubleshooting Sequence

```text
1. Are all physical member links Up?
        ↓
2. Are both ends using the same protocol?
        ↓
3. Are LACP/PAgP modes compatible?
        ↓
4. Are switchport/routed modes consistent?
        ↓
5. Do VLAN/native/allowed settings match?
        ↓
6. Are members bundled or suspended?
        ↓
7. Is the Port-Channel itself Up?
        ↓
8. Is STP/routing using the Port-Channel?
        ↓
9. Is traffic distribution expected for the current hash?
```

Start with:

```cisco
show etherchannel summary
```

because it quickly reveals whether the physical members are actually bundled.

---

## Cisco NX-OS — Key Difference

NX-OS also uses Port-Channels and LACP, but configuration and multi-chassis behavior can differ.

Basic LACP example:

```cisco
feature lacp

interface ethernet1/1
 channel-group 10 mode active

interface port-channel10
 switchport
 switchport mode trunk
```

For dual-chassis aggregation, NX-OS commonly uses **vPC**, which has additional peer-link, keepalive, consistency, and failure-mode requirements.

Do not apply IOS XE StackWise/VSS assumptions directly to NX-OS vPC.

---

## 5. Common Gotchas and Misconceptions

### EtherChannel Adds the Bandwidth of All Links to One Flow

**Incorrect.**

```text
4 × 1 Gbps
≠
One 4-Gbps flow
```

A normal flow hashes to one member.

EtherChannel provides **aggregate** bandwidth across multiple flows.

---

### LACP Passive + Passive Forms a Bundle

**Incorrect.**

```text
passive ↔ passive
= No negotiation initiation
= No bundle
```

At least one side must be:

```text
active
```

---

### PAgP Auto + Auto Forms a Bundle

**Incorrect.**

At least one side must use:

```text
desirable
```

---

### `mode on` Is the Same as LACP Active

**Incorrect.**

```text
mode active
= LACP negotiation

mode on
= Static EtherChannel
= No negotiation
```

Do not configure one side as LACP and the other as static `on`.

---

### STP Sees Every EtherChannel Member as a Separate Link

**Incorrect.**

When the bundle forms correctly, STP treats the Port-Channel as one logical interface.

If members fail to bundle, however, STP may see separate links and the topology can behave very differently.

---

### All Member Links Can Have Different Trunk Settings

**Incorrect.**

Member interfaces need compatible Layer 2 parameters.

Mismatch can cause:

```text
Suspended interfaces
Bundle failure
Unexpected forwarding
```

Keep configuration consistent and apply common policy to the Port-Channel.

---

### A Port-Channel Can Span Any Two Independent Switches

**Incorrect.**

A standard EtherChannel terminates on one logical device at each end.

Dual-chassis termination requires a supported technology such as:

```text
StackWise / StackWise Virtual
VSS
vPC
Other vendor equivalent
```

---

### Uneven Link Utilization Means EtherChannel Is Broken

**Not necessarily.**

Hashing can produce uneven utilization when traffic consists of a small number of large flows.

Check:

```cisco
show etherchannel load-balance
```

and actual flow characteristics before changing configuration.

---

### Physical Members Should Carry Independent IP Addresses in a Layer 3 Port-Channel

**Incorrect.**

The logical Layer 3 address belongs on the Port-Channel.

```text
Physical members
= Bundle membership

Port-Channel
= IP/routing interface
```

---

## 6. Trade-Offs

### Best Practice

- Prefer **LACP** when both endpoints support it.
- Keep member-link settings consistent.
- Configure shared Layer 2/Layer 3 policy on the Port-Channel rather than independently on each member.
- Verify `show etherchannel summary` after every change.
- Design capacity assuming a member failure, not only the all-links-up state.
- Choose the load-balancing hash based on real traffic patterns when optimization is required.
- Use supported multi-chassis aggregation when chassis redundancy is needed.

---

### Context-Dependent Trade-Off — LACP vs Static

**LACP**

```text
+ Standards-based
+ Negotiates and validates members
+ Better operational visibility
- Adds control-plane negotiation
```

**Static `on`**

```text
+ Simple
+ No negotiation dependency
- No protocol validation
- Misconfiguration risk is higher
```

LACP is normally the safer operational choice.

---

### Context-Dependent Trade-Off — More Member Links

```text
More members
+ More aggregate capacity
+ More link-level redundancy

But
- One flow still normally uses one member
- Hash distribution may remain uneven
- More optics/cabling/ports are consumed
```

Adding links is useful when the traffic mix contains enough independent flows to use them.

---

### Context-Dependent Trade-Off — Minimum Links

```text
Minimum-links configured
+ Prevents degraded bundle from staying operational below required capacity
- Can intentionally take the entire logical link down after member failures
```

Use it when reduced capacity would violate application or design requirements.

---

### Incorrect or Unsafe

- Using static `on` without verifying both ends are identically configured.
- Mixing LACP, PAgP, and static modes across the same bundle.
- Changing member switchport parameters independently and creating consistency failures.
- Assuming a multi-chassis Port-Channel works without a supported chassis-pair technology.
- Changing load-balancing algorithms in production without understanding current traffic flows and impact.
- Designing the network as if the Port-Channel always has all members available.

---

## Quick Reference

```text
Link Aggregation
= Multiple physical links → one logical link

Cisco Logical Interface
= Port-Channel

EtherChannel
= Cisco term for the aggregated bundle

Layer 2 EtherChannel
= One logical switchport/trunk

Layer 3 EtherChannel
= One logical routed interface

LACP
= IEEE 802.1AX
= active / passive

LACP Forms:
active-active
active-passive

Does Not Form:
passive-passive

PAgP
= Cisco proprietary
= desirable / auto

PAgP Forms:
desirable-desirable
desirable-auto

Does Not Form:
auto-auto

Static
= mode on
= No negotiation

Traffic Distribution
= Hash-based
= Normally per flow

One Flow
= Normally limited to one member's bandwidth

Member Failure
= Traffic rehashed to surviving members

STP
= Sees one logical Port-Channel

Layer 3 IP
= Configured on Port-Channel, not member links

Primary Verification
= show etherchannel summary

LACP Verification
= show lacp neighbor

Load-Balance Verification
= show etherchannel load-balance
```

</div>
