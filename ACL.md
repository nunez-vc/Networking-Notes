<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# Access Control List (ACL)

> **Core idea:** An ACL is an ordered list of match conditions used to classify packets. When applied as an interface filter, IOS/IOS XE evaluates entries **top-down, first match wins**, and packets that match no entry are denied by the implicit final deny.

---

## 1. What It Is

An **Access Control List (ACL)** is a Layer 3/Layer 4 packet-matching policy used to permit, deny, or classify traffic based on fields such as source/destination IP address, protocol, and TCP/UDP ports.

On Cisco IOS/IOS XE, ACLs are commonly used for:

```text
Interface traffic filtering
Management-plane restriction
NAT classification
QoS classification
Policy-based routing / route-map matching
Other feature-specific packet selection
```

> An ACL is fundamentally a **classifier**. When it is attached as a packet filter, `permit` allows matching traffic and `deny` drops it. When another feature references the ACL, that feature determines what a match means.

---

## 2. How It Works

## Ordered Processing

An ACL contains individual rules called **Access Control Entries (ACEs)**.

Example:

```text
10 permit ...
20 deny ...
30 permit ...
```

Packet processing is:

```text
Packet enters ACL
      ↓
Check first ACE
      ↓
Match?
  ┌───┴───┐
 Yes      No
  │        │
Apply     Check next ACE
action        ↓
  │       Continue
  ↓
Stop processing
```

The first matching ACE determines the result.

```text
First match wins
```

If no ACE matches:

```text
Implicit deny
```

Conceptually:

```text
deny ip any any
```

exists at the end of an IPv4 ACL even though it is not normally shown in the configuration.

> ACL order is therefore part of the policy. A broad ACE placed before a specific ACE can make the later entry unreachable.

---

## Standard vs Extended IPv4 ACLs

### Standard IPv4 ACL

A standard ACL matches only the **source IPv4 address**.

```text
Source IP
   ↓
Permit / Deny
```

Example intent:

```text
Permit traffic sourced from 10.10.10.0/24
Deny everything else
```

Because it cannot evaluate destination, protocol, or port, it is less precise.

---

### Extended IPv4 ACL

An extended ACL can match:

```text
Protocol
Source IPv4 address
Destination IPv4 address
TCP/UDP source port
TCP/UDP destination port
Selected protocol-specific fields
```

Example intent:

```text
Permit TCP
from 10.10.10.0/24
to 172.16.20.10
destination port 443
```

Extended ACLs are the normal choice when traffic filtering must distinguish applications or destinations.

---

## Wildcard Masks

IOS/IOS XE IPv4 ACLs commonly use **wildcard masks**.

Memory rule:

```text
Wildcard bit 0
= Must match

Wildcard bit 1
= Ignore
```

Example:

```text
10.10.10.0 0.0.0.255
```

means:

```text
Match 10.10.10.x
```

Relationship to a normal subnet mask:

```text
Subnet mask:   255.255.255.0
Wildcard mask:   0.0.0.255
```

For ordinary contiguous masks:

```text
Wildcard = 255.255.255.255 - Subnet Mask
```

Useful keywords:

```cisco
host 10.10.10.10
```

is equivalent to:

```text
10.10.10.10 0.0.0.0
```

and:

```cisco
any
```

is equivalent to:

```text
0.0.0.0 255.255.255.255
```

---

## Inbound vs Outbound ACLs

An interface ACL can be applied in either direction.

### Inbound

```text
Packet enters interface
        ↓
Inbound ACL
        ↓
If permitted:
Routing / forwarding decision
```

Inbound ACLs filter traffic before the router forwards it.

---

### Outbound

```text
Routing / forwarding decision
        ↓
Selected exit interface
        ↓
Outbound ACL
        ↓
Transmit if permitted
```

The direction is always from the router's perspective:

```text
in  = entering the interface
out = leaving the interface
```

---

## Example Packet Decision

ACL:

```cisco
ip access-list extended WEB_ONLY
 10 permit tcp 10.10.10.0 0.0.0.255 host 172.16.20.10 eq 443
 20 deny ip any any
```

Traffic:

```text
10.10.10.25 → 172.16.20.10 TCP/443
```

matches ACE 10:

```text
Permit
```

Traffic:

```text
10.10.10.25 → 172.16.20.10 TCP/22
```

does not match ACE 10 and reaches ACE 20:

```text
Deny
```

---

## Named vs Numbered ACLs

IOS/IOS XE supports both numbered and named IPv4 ACLs.

### Numbered

```cisco
access-list 10 permit 10.10.10.0 0.0.0.255
```

### Named

```cisco
ip access-list standard MANAGEMENT
 permit 10.10.10.0 0.0.0.255
```

Named ACLs are generally easier to operate because they are self-describing and support structured editing with sequence numbers.

---

## Sequence Numbers

Named ACLs use sequence numbers to control ACE order.

Example:

```cisco
ip access-list extended USERS_TO_SERVER
 10 permit tcp 10.10.10.0 0.0.0.255 host 172.16.20.10 eq 443
 20 deny ip any any
```

A new ACE can be inserted between them:

```cisco
ip access-list extended USERS_TO_SERVER
 15 permit icmp 10.10.10.0 0.0.0.255 host 172.16.20.10 echo
```

Result:

```text
10 ...
15 ...
20 ...
```

This avoids rebuilding the entire ACL simply to insert one rule.

---

## ACLs Are Stateless by Default

A normal IOS/IOS XE ACL evaluates each packet independently.

```text
Packet 1 → Match policy
Packet 2 → Match policy
Packet 3 → Match policy
```

It does not maintain connection state like a stateful firewall.

For example, a permit rule for outbound TCP traffic does not automatically create a stateful reverse-flow entry.

The extended ACL keyword:

```cisco
established
```

does **not** create true stateful inspection. It matches TCP packets with ACK or RST set and is only a limited stateless check.

> For stateful security policy, use a stateful firewall capability rather than treating a router ACL as equivalent.

---

## ACL as Filter vs ACL as Classifier

This distinction is operationally important.

### Applied to an Interface

```cisco
ip access-group ACL_NAME in
```

ACL semantics are:

```text
permit → Forward if all other checks allow it
deny   → Drop
```

---

### Referenced by Another Feature

Example:

```text
NAT references ACL
QoS references ACL
Route-map references ACL
```

The ACL identifies matching traffic, but the feature decides what happens next.

Example:

```text
ACL permit
+
NAT rule
=
Traffic is selected for translation
```

It does **not** mean the ACL itself is acting as a security permit on the interface.

---

## IPv6 ACLs

IPv6 ACLs use IPv6 prefixes rather than IPv4 wildcard masks.

Example:

```cisco
ipv6 access-list V6_WEB
 permit tcp 2001:db8:10::/64 host 2001:db8:20::10 eq 443
```

Apply to an interface:

```cisco
interface GigabitEthernet0/0
 ipv6 traffic-filter V6_WEB in
```

Key differences:

```text
IPv4 ACL
= Usually uses wildcard masks

IPv6 ACL
= Uses IPv6 prefix notation
```

On IOS/IOS XE, IPv6 ACL behavior includes special implicit handling for essential Neighbor Discovery traffic before the final implicit deny. Exact behavior can vary by software train and feature context, so validate with the applicable Cisco command reference when writing restrictive IPv6 policies.

> Do not blindly copy an IPv4 ACL strategy into IPv6. ICMPv6 is fundamental to Neighbor Discovery and Path MTU Discovery.

---

## Hardware Forwarding

On modern Catalyst and routing platforms, many ACLs are compiled into hardware forwarding resources such as TCAM.

Conceptually:

```text
Configured ACL
      ↓
Compiled into forwarding hardware
      ↓
Packets matched at line rate
```

Hardware capacity is finite. Large or complex ACL policies can consume TCAM resources, and scale limits vary by platform and software release.

---

## 3. Why and When It Is Used

ACLs are appropriate when traffic must be matched or restricted based on deterministic packet fields.

Common uses:

```text
Restrict access between subnets
Protect management interfaces
Limit traffic entering or leaving an interface
Select traffic for NAT
Classify traffic for QoS
Match traffic in policy-based routing
Control access to device management services
```

ACLs are useful for simple, explicit policy.

They are less suitable when the requirement needs:

```text
Application awareness
User identity
Deep inspection
Dynamic session state
Threat detection
Complex security policy
```

Those requirements normally belong on a stateful firewall or other security platform.

---

## 4. Key Configuration, Parameters, or CLI

> **Platform:** Cisco IOS / IOS XE.

---

## Standard IPv4 ACL

Permit only management subnet `10.10.10.0/24`:

```cisco
ip access-list standard MGMT_ONLY
 permit 10.10.10.0 0.0.0.255
```

The implicit deny blocks all other source addresses.

---

## Extended IPv4 ACL

Permit HTTPS from users to one server:

```cisco
ip access-list extended USERS_TO_WEB
 10 permit tcp 10.10.10.0 0.0.0.255 host 172.16.20.10 eq 443
 20 deny ip any any
```

Apply inbound:

```cisco
interface Vlan10
 ip access-group USERS_TO_WEB in
```

---

## Management-Plane ACL

Restrict VTY access:

```cisco
ip access-list standard VTY_MGMT
 permit 10.10.10.0 0.0.0.255
```

```cisco
line vty 0 15
 access-class VTY_MGMT in
 transport input ssh
```

`access-class` applies the ACL to management access to the device rather than to transit traffic on a routed interface.

---

## IPv6 ACL

```cisco
ipv6 access-list V6_USERS_TO_WEB
 permit tcp 2001:db8:10::/64 host 2001:db8:20::10 eq 443
```

Apply:

```cisco
interface Vlan10
 ipv6 traffic-filter V6_USERS_TO_WEB in
```

---

## Remarks

Use remarks to document intent:

```cisco
ip access-list extended USERS_TO_WEB
 remark Allow user HTTPS to application server
 10 permit tcp 10.10.10.0 0.0.0.255 host 172.16.20.10 eq 443
 20 deny ip any any
```

---

## Logging

An ACE can optionally log matches:

```cisco
ip access-list extended USERS_TO_WEB
 20 deny ip any any log
```

Use logging selectively.

```text
High-volume ACE + log
= Potential CPU/logging impact
```

Exact logging behavior and rate limiting vary by platform and release.

---

## Verification

```cisco
show access-lists
show ip access-lists
show ipv6 access-list
show ip interface <interface>
show ipv6 interface <interface>
show running-config interface <interface>
```

Useful things to verify:

```text
Correct ACL name
Correct interface
Correct direction
Correct ACE order
Correct wildcard/prefix
Correct protocol and ports
Hit counters increasing
Expected implicit deny behavior
```

---

## Practical Troubleshooting Sequence

```text
1. Which interface does the packet enter?
        ↓
2. Is an inbound ACL applied there?
        ↓
3. Which route selects the exit interface?
        ↓
4. Is an outbound ACL applied there?
        ↓
5. Which ACE should match?
        ↓
6. Is a broader earlier ACE matching first?
        ↓
7. Are ACL counters increasing?
        ↓
8. Is return traffic permitted separately?
```

For difficult cases, use a packet capture plus ACL counters to prove where the flow is being denied.

---

## 5. Common Gotchas and Misconceptions

### ACLs Continue Checking After a Match

**Incorrect.**

```text
First match wins
```

Once a packet matches an ACE, later entries are not evaluated.

---

### There Is No Deny Unless You Configure One

**Incorrect.**

Every ACL has an implicit deny at the end.

```text
No match
=
Denied
```

This is why adding only a single `deny` statement can accidentally block all other traffic.

---

### Standard ACLs Match Destination Addresses

**Incorrect.**

Standard IPv4 ACLs match the **source IPv4 address** only.

Use an extended ACL when destination, protocol, or ports matter.

---

### Wildcard Masks Work Like Subnet Masks

**Incorrect.**

Wildcard logic is reversed:

```text
0 = Must match
1 = Ignore
```

Example:

```text
0.0.0.255
```

means ignore the last octet.

---

### `in` and `out` Refer to the Traffic Source and Destination

**Incorrect.**

They refer to the interface from the router's perspective.

```text
in
= Packet entering router through that interface

out
= Packet leaving router through that interface
```

---

### A Permit ACE Automatically Allows Return Traffic

**Incorrect.**

Router ACLs are stateless by default.

Return traffic must independently match a permit rule if it traverses an ACL in the reverse direction.

---

### `established` Makes an ACL Stateful

**Incorrect.**

The TCP `established` keyword checks TCP ACK/RST bits. It does not maintain a real connection table and is not equivalent to stateful firewall inspection.

---

### ACL `permit` Always Means "Allow Through the Router"

**Incorrect.**

When an ACL is used by NAT, QoS, route-maps, or another feature, `permit` often means:

```text
This traffic matches the feature
```

not:

```text
This packet is globally authorized
```

Always interpret the ACL in the context of the feature that references it.

---

### ACL Placement Never Matters

**Incorrect.**

Placement determines which traffic is evaluated and can affect scale, troubleshooting, and policy scope.

A traditional rule of thumb is:

```text
Extended ACLs → closer to source
Standard ACLs → closer to destination
```

but modern production placement should follow the actual policy boundary, topology, platform capability, and failure domain rather than applying that rule blindly.

---

### Denying All ICMPv6 Is Safe

**Incorrect or Unsafe.**

IPv6 depends on ICMPv6 for functions such as:

```text
Neighbor Discovery
Router Discovery
Path MTU Discovery
Error signaling
```

IPv6 ACLs must preserve required ICMPv6 control traffic.

---

## 6. Trade-Offs

### Best Practice

- Use **named ACLs** with sequence numbers and remarks for operational clarity.
- Put specific ACEs before broad ACEs.
- Explicitly document the intended default behavior at the end of important ACLs.
- Verify interface and direction before applying an ACL.
- Use hit counters and packet captures to validate policy behavior.
- Treat IPv4 and IPv6 ACLs as separate policies.
- Preserve required infrastructure/control traffic such as routing protocols, DHCP, ICMP/ICMPv6, and management flows.

---

### Context-Dependent Trade-Off — Explicit Final Deny

An ACL already has an implicit deny, but adding:

```cisco
deny ip any any
```

can make policy intent clearer and allows options such as logging.

```text
Explicit deny
+ Easier to read
+ Can log matches
- Logging can create operational overhead
```

Use logging only where the operational value justifies the cost.

---

### Context-Dependent Trade-Off — ACL Location

Filtering closer to the source can reduce unwanted traffic earlier:

```text
+ Saves downstream bandwidth/resources
+ Limits propagation of unwanted traffic
```

Filtering closer to the protected destination can centralize policy:

```text
+ Fewer enforcement points
+ Easier centralized management in some designs
```

Choose based on topology, scale, platform capabilities, and security policy.

---

### Context-Dependent Trade-Off — Router ACL vs Stateful Firewall

```text
Router ACL
+ Simple
+ Fast
+ Deterministic L3/L4 filtering
- Stateless
- Limited application awareness

Stateful Firewall
+ Session awareness
+ Richer application/security policy
- More state and operational complexity
```

Use an ACL where stateless packet classification is sufficient. Use a stateful firewall where session-aware security is required.

---

### Incorrect or Unsafe

- Applying a restrictive ACL without confirming the management path and rollback method.
- Relying on ACLs as equivalent to a stateful firewall.
- Placing a broad permit before a required deny.
- Applying an ACL in the wrong direction.
- Using logging on high-volume ACEs without evaluating platform impact.
- Blocking required routing, DHCP, ICMP, or ICMPv6 control traffic without understanding the consequences.

---

## Quick Reference

```text
ACL
= Ordered packet classifier

ACE
= Access Control Entry

Processing
= Top-down
= First match wins
= Stop after match

No match
= Implicit deny

Standard IPv4 ACL
= Source address only

Extended IPv4 ACL
= Protocol
= Source
= Destination
= Ports / selected L4 fields

Wildcard Mask
= 0 must match
= 1 ignore

Inbound
= Before normal forwarding decision

Outbound
= After exit interface is selected

Named ACL
= Easier structured editing

Sequence Numbers
= Control ACE order

ACL on Interface
permit = allow
deny   = drop

ACL Used by Feature
= Match semantics depend on that feature

IPv6 ACL
= Uses IPv6 prefixes
= Applied with ipv6 traffic-filter

Router ACL
= Stateless by default

established
= TCP ACK/RST test
= Not true stateful inspection

Core Rule
= Correct ACL + correct interface + correct direction + correct order
```
## CCNA Configuration Cheatsheet

**CCNA 200-301 v2.0 — IOS-XE Standard Numbered IPv4 ACL**

| Command | Description |
|---|---|
| **Create standard numbered ACE:**<br>`(config)#access-list <1-99\|1300-1999> {permit\|deny} <source> [<source-wildcard>] [log]` | Defines a numbered standard IPv4 ACL entry. |
| **Match exact source host:**<br>`(config)#access-list <acl-number> {permit\|deny} host <source-ip> [log]` | Matches one source IPv4 address. |
| **Match any source:**<br>`(config)#access-list <acl-number> {permit\|deny} any [log]` | Matches any source IPv4 address. |
| **Add numbered ACL remark:**<br>`(config)#access-list <acl-number> remark <text>` | Adds descriptive text to the ACL. |

**CCNA 200-301 v2.0 — IOS-XE Extended Numbered IPv4 ACL**

| Command | Description |
|---|---|
| **Create extended numbered ACE:**<br>`(config)#access-list <100-199\|2000-2699> {permit\|deny} <protocol> <source> <source-wildcard> <destination> <destination-wildcard> [log]` | Defines a numbered extended IPv4 ACL entry. |
| **Match TCP ports:**<br>`(config)#access-list <acl-number> {permit\|deny} tcp <source> <source-wildcard> [{eq\|gt\|lt\|neq} <source-port>\|range <source-start> <source-end>] <destination> <destination-wildcard> [{eq\|gt\|lt\|neq} <destination-port>\|range <destination-start> <destination-end>] [log]` | Matches TCP addresses and optional Layer 4 ports. |
| **Match UDP ports:**<br>`(config)#access-list <acl-number> {permit\|deny} udp <source> <source-wildcard> [{eq\|gt\|lt\|neq} <source-port>\|range <source-start> <source-end>] <destination> <destination-wildcard> [{eq\|gt\|lt\|neq} <destination-port>\|range <destination-start> <destination-end>] [log]` | Matches UDP addresses and optional Layer 4 ports. |
| **Match ICMP:**<br>`(config)#access-list <acl-number> {permit\|deny} icmp <source> <source-wildcard> <destination> <destination-wildcard> [log]` | Matches IPv4 ICMP traffic. |
| **Match exact hosts:**<br>`(config)#access-list <acl-number> {permit\|deny} tcp host <source-ip> host <destination-ip> eq <destination-port> [log]` | Matches TCP between two exact hosts. |
| **Add numbered ACL remark:**<br>`(config)#access-list <acl-number> remark <text>` | Adds descriptive text to the ACL. |

**CCNA 200-301 v2.0 — IOS-XE Named IPv4 ACL**

| Command | Description |
|---|---|
| **Enter named standard ACL:**<br>`(config)#ip access-list standard <acl-name>` | Enters named standard IPv4 ACL configuration mode. |
| **Add standard named ACE:**<br>`(config-std-nacl)#<sequence> {permit\|deny} <source> [<source-wildcard>] [log]` | Adds a sequenced standard IPv4 ACE. |
| **Enter named extended ACL:**<br>`(config)#ip access-list extended <acl-name>` | Enters named extended IPv4 ACL configuration mode. |
| **Add extended named ACE:**<br>`(config-ext-nacl)#<sequence> {permit\|deny} <protocol> <source> <source-wildcard> <destination> <destination-wildcard> [log]` | Adds a sequenced extended IPv4 ACE. |
| **Add TCP named ACE:**<br>`(config-ext-nacl)#<sequence> {permit\|deny} tcp <source> <source-wildcard> <destination> <destination-wildcard> [{eq\|gt\|lt\|neq} <destination-port>\|range <destination-start> <destination-end>] [log]` | Adds a TCP-specific named ACE. |
| **Add ACL-mode remark:**<br>`(config-std-nacl)#remark <text>`<br>&nbsp;&nbsp;○ `(config-ext-nacl)#remark <text>` | Adds descriptive text inside a named ACL. |
| **Delete ACE by sequence:**<br>`(config-std-nacl)#no <sequence>`<br>&nbsp;&nbsp;○ `(config-ext-nacl)#no <sequence>` | Removes one ACE by sequence number. |

**CCNA 200-301 v2.0 — IOS-XE Interface Application**

| Command | Description |
|---|---|
| **Apply IPv4 ACL:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip access-group <acl-number\|acl-name> {in\|out}` | Applies an IPv4 ACL to an interface direction. |
| **Remove IPv4 ACL:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no ip access-group <acl-number\|acl-name> {in\|out}` | Removes an IPv4 ACL from an interface direction. |

**CCNA 200-301 v2.0 — IOS-XE Verification**

| Command | Description |
|---|---|
| **Show IPv4 ACLs:**<br>`#show ip access-lists [<acl-number\|acl-name>]` | Displays IPv4 ACL entries, sequences, and counters. |
| **Show all ACLs:**<br>`#show access-lists [<acl-number\|acl-name>]` | Displays configured ACLs and match counters. |
| **Show interface ACL binding:**<br>`#show ip interface [<interface-id>]` | Displays inbound and outbound IPv4 ACL assignments. |
| **Show running configuration:**<br>`#show running-config` | Displays configured ACLs and interface bindings. |

## CCNP Configuration Cheatsheet

**CCNP Enterprise — ENCOR 350-401 v1.2 — IOS-XE Port ACL (PACL)**

| Command | Description |
|---|---|
| **Create PACL:**<br>`(config)#ip access-list extended <acl-name>`<br>&nbsp;&nbsp;○ `(config-ext-nacl)#<sequence> {permit\|deny} <protocol> <source> <source-wildcard> <destination> <destination-wildcard>` | Creates the IPv4 ACL used as a PACL. |
| **Apply PACL inbound:**<br>`(config)#interface <switchport-id>`<br>&nbsp;&nbsp;○ `(config-if)#switchport`<br>&nbsp;&nbsp;○ `(config-if)#ip access-group <acl-name> in` | Applies a PACL inbound on a Layer 2 port. |
| **Verify PACL configuration:**<br>`#show running-config interface <switchport-id>` | Displays the switchport PACL configuration. |
| **Verify PACL counters:**<br>`#show ip access-lists <acl-name>` | Displays PACL ACEs and match counters. |

**CCNP Enterprise — ENCOR 350-401 v1.2 — IOS-XE VLAN ACL (VACL)**

| Command | Description |
|---|---|
| **Create VACL match ACL:**<br>`(config)#ip access-list extended <acl-name>`<br>&nbsp;&nbsp;○ `(config-ext-nacl)#permit <protocol> <source> <source-wildcard> <destination> <destination-wildcard>` | Creates an IPv4 ACL used for VACL matching. |
| **Create VLAN access-map sequence:**<br>`(config)#vlan access-map <map-name> <sequence>` | Enters a VLAN access-map sequence. |
| **Match IPv4 ACL:**<br>`(config-access-map)#match ip address <acl-number\|acl-name>` | Selects traffic matching an IPv4 ACL. |
| **Match MAC ACL:**<br>`(config-access-map)#match mac address <acl-name>` | Selects traffic matching a named MAC ACL. |
| **Forward matched traffic:**<br>`(config-access-map)#action forward` | Forwards traffic matched by the sequence. |
| **Drop matched traffic:**<br>`(config-access-map)#action drop` | Drops traffic matched by the sequence. |
| **Apply VACL to VLANs:**<br>`(config)#vlan filter <map-name> vlan-list <vlan-list>` | Applies the VLAN access-map to specified VLANs. |
| **Verify VLAN access-map:**<br>`#show vlan access-map [<map-name>]` | Displays configured VLAN access-map sequences. |
| **Verify VACL assignment:**<br>`#show vlan filter` | Displays VLAN access-map assignments to VLANs. |

**CCNP Enterprise — ENCOR 350-401 v1.2 — IOS-XE CoPP ACL Classification**

| Command | Description |
|---|---|
| **Create CoPP classifier ACL:**<br>`(config)#ip access-list extended <acl-name>`<br>&nbsp;&nbsp;○ `(config-ext-nacl)#permit <protocol> <source> <source-wildcard> <destination> <destination-wildcard>` | Defines traffic selected for a CoPP class. |
| **Create CoPP class-map:**<br>`(config)#class-map match-all <class-name>`<br>&nbsp;&nbsp;○ `(config-cmap)#match access-group name <acl-name>` | Maps ACL-matched traffic into a CoPP class. |
| **Create CoPP policy-map:**<br>`(config)#policy-map <policy-name>`<br>&nbsp;&nbsp;○ `(config-pmap)#class <class-name>` | Enters the CoPP class inside a policy-map. |
| **Configure class policer:**<br>`(config-pmap-c)#police <rate-bps> conform-action transmit exceed-action transmit violate-action {transmit\|drop}` | Configures class policing actions and rate. |
| **Apply CoPP policy:**<br>`(config)#control-plane`<br>&nbsp;&nbsp;○ `(config-cp)#service-policy input <policy-name>` | Applies the policy to control-plane input traffic. |
| **Verify CoPP policy:**<br>`#show policy-map control-plane input` | Displays CoPP class counters and policing statistics. |

**CCNP Enterprise — ENCOR 350-401 v1.2 — IOS-XE ACL Resequencing**

| Command | Description |
|---|---|
| **Resequence named ACL:**<br>`(config)#ip access-list resequence <acl-name> <starting-sequence> <increment>` | Reassigns ACE sequence numbers using a specified increment. |
| **Verify resequenced ACL:**<br>`#show ip access-lists <acl-name>` | Displays updated ACL sequence numbers and counters. |

**CCNP Security — SCOR 350-701 v2.0 / SNCF — FTD Management Access Lists**

| Command | Description |
|---|---|
| **Configure SSH management ACL:**<br>`>configure ssh-access-list <ip-address>/<prefix-length>[,<ip-address>/<prefix-length>...]` | Replaces allowed SSH management source networks. |
| **Configure HTTPS management ACL:**<br>`>configure https-access-list <ip-address>/<prefix-length>[,<ip-address>/<prefix-length>...]` | Replaces allowed HTTPS management source networks. |
| **Disable SSH management access:**<br>`>configure disable-ssh-access` | Disables SSH access to the FTD management interface. |
| **Disable HTTPS management access:**<br>`>configure disable-https-access` | Disables HTTPS access to the FTD management interface. |
| **Verify SSH management ACL:**<br>`>show ssh-access-list` | Displays the configured SSH management access list. |
| **Verify HTTPS management ACL:**<br>`>show https-access-list` | Displays the configured HTTPS management access list. |

**CCNP Security — SCOR 350-701 v2.0 / SNCF — FTD Generated ACP ACL Verification**

| Command | Description |
|---|---|
| **Show generated access list:**<br>`>show access-list [<acl-id>]` | Displays generated low-level ACL rules and hit counters. |
| **Show ACL element count:**<br>`>show access-list element-count` | Displays total generated access control entries. |
| **Show ACP rule hits:**<br>`>show rule hits` | Displays hit information for evaluated ACP and prefilter rules. |
| **Show one ACP rule hit:**<br>`>show rule hits id <rule-id>` | Displays hit information for one policy rule. |

</div>
