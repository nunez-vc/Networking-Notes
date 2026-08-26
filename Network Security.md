<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# Network Security

> **Core idea:** Network security protects the confidentiality, integrity, availability, and authorized use of networked systems by controlling **who can connect, what they can reach, what traffic is allowed, and how attacks or failures are detected and contained**. Effective security is layered across the access, data, control, and management planes rather than relying on one firewall or ACL.

---

## 1. What It Is

**Network security** is the set of technical controls and operational practices used to protect network infrastructure, communications, and connected systems from unauthorized access, misuse, alteration, disruption, and data exposure.

At a practical level, it enforces:

```text
Identity
   ↓
Authentication
   ↓
Authorization
   ↓
Segmentation / Policy
   ↓
Inspection / Enforcement
   ↓
Logging / Detection
   ↓
Response
```

The security objective is normally expressed through:

```text
Confidentiality
Integrity
Availability
```

with **authentication, authorization, accountability, and least privilege** supporting those objectives.

---

## 2. How It Works

## Defense in Depth

No single control reliably protects every attack path.

A layered design may include:

```text
Endpoint security
      ↓
Access-layer controls
      ↓
Identity / NAC
      ↓
Segmentation
      ↓
ACL / Firewall policy
      ↓
IDS / IPS
      ↓
VPN / Encryption
      ↓
Secure management
      ↓
Logging / Monitoring
```

Each layer reduces either:

```text
Attack probability
Blast radius
Unauthorized reachability
Time to detection
Time to recovery
```

---

# Security Planes

## Data Plane

The data plane carries user and application traffic.

Typical protections:

```text
ACLs
Stateful firewalls
Segmentation
NAT where required
IDS / IPS
QoS / rate controls
VPN encryption
Anti-spoofing
```

The security question is:

```text
Should this packet / flow be allowed to pass?
```

---

## Control Plane

The control plane runs protocols that build forwarding state.

Examples:

```text
OSPF
BGP
STP
ARP / NDP
FHRPs
LACP
Routing adjacencies
```

Threats include:

```text
Forged protocol messages
Route injection
Neighbor spoofing
Control-plane exhaustion
Unauthorized adjacencies
```

Controls may include:

```text
Protocol authentication where supported
Peer filtering
Control-plane policing
Infrastructure ACLs
Routing policy
Layer 2 protections
```

---

## Management Plane

The management plane controls the devices themselves.

Examples:

```text
SSH
HTTPS
NETCONF / RESTCONF
SNMP
AAA
Syslog
NTP
Configuration APIs
```

This plane should be tightly restricted because compromise can grant control of the network.

Typical protections:

```text
Dedicated management network or VRF
AAA
MFA through external identity systems where supported
SSH / HTTPS only
Source-address restrictions
SNMPv3
Central logging
Configuration backups
Role-based access
```

---

# Identity, Authentication, Authorization, and Accounting

## AAA

AAA separates three functions:

```text
Authentication
= Who are you?

Authorization
= What are you allowed to do?

Accounting
= What did you do?
```

Common enterprise protocols:

```text
RADIUS
TACACS+
```

Typical use:

```text
RADIUS
→ Network access / 802.1X / VPN

TACACS+
→ Device-administration AAA
```

The exact choice depends on platform and policy.

---

## Network Access Control

**802.1X** provides port-based network access control.

Typical sequence:

```text
Endpoint
  ↓
Authenticator
(Switch / AP)
  ↓
RADIUS
  ↓
Identity / policy decision
  ↓
Permit, deny, VLAN, ACL, or segmentation result
```

Roles:

```text
Supplicant
= Endpoint

Authenticator
= Switch or wireless infrastructure

Authentication server
= RADIUS / identity platform
```

802.1X determines whether and how a device/user gains network access.

It does not replace firewalling or application authorization.

---

# Segmentation

Segmentation limits which systems can communicate.

Common mechanisms:

```text
VLANs
VRFs
ACLs
Firewalls
Security groups
TrustSec / SGT policy
Overlay segmentation
Microsegmentation
```

Important distinction:

```text
VLAN
= Layer 2 segmentation construct

Security policy
= Determines permitted communication
```

A VLAN by itself is not a complete security boundary if routing between VLANs is unrestricted.

---

## Macrosegmentation vs Microsegmentation

### Macrosegmentation

Separates broad zones:

```text
Users
Servers
Guest
Management
Internet Edge
DMZ
```

### Microsegmentation

Applies more granular policy between workloads or identities:

```text
App server → Database: TCP/5432 permit
User subnet → Database: deny
```

The objective is to reduce lateral movement.

---

# Stateless ACLs vs Stateful Firewalls

## ACL

A traditional router/switch ACL evaluates packets independently.

```text
Packet arrives
    ↓
Match rule
    ↓
Permit / deny
```

It does not normally maintain session state.

---

## Stateful Firewall

A stateful firewall tracks connection state.

Example:

```text
Client → Server
TCP session permitted
        ↓
State table created
        ↓
Return traffic validated against state
```

A state table may track:

```text
Source / destination IP
Ports
Protocol
TCP state
Timeouts
NAT mappings
Security policy
```

> A stateful firewall provides richer session-aware enforcement than a basic stateless ACL, but it still depends on correct policy and routing.

---

# IDS vs IPS

## IDS

**Intrusion Detection System**

```text
Observes traffic
Detects suspicious activity
Generates alerts
```

Normally does not directly block traffic.

---

## IPS

**Intrusion Prevention System**

```text
Traffic passes through inspection
        ↓
Threat detected?
   /          \
 No           Yes
 |             |
Forward      Drop / reset / block
```

Because IPS is inline, false positives and performance capacity must be considered carefully.

---

# Encryption and VPNs

Encryption protects traffic against unauthorized observation or modification while in transit.

Common technologies:

```text
IPsec
TLS
MACsec
SSH
HTTPS
```

### IPsec

Typical use:

```text
Site-to-site VPN
Remote-access VPN
Untrusted WAN transport
```

Provides combinations of:

```text
Confidentiality
Integrity
Peer authentication
Anti-replay protection
```

---

### MACsec

MACsec protects Ethernet traffic on supported links.

Conceptually:

```text
Ethernet frame
      ↓
MACsec protection
      ↓
Protected Layer 2 link
```

It is useful where hop-by-hop Layer 2 confidentiality/integrity is required.

---

# Layer 2 Security

Layer 2 attacks can bypass higher-layer assumptions if access switching is not protected.

Important controls include:

```text
Port security
DHCP snooping
Dynamic ARP Inspection
IP Source Guard
BPDU Guard
Root Guard
Storm control
Unused-port shutdown
```

---

## DHCP Snooping

DHCP snooping distinguishes:

```text
Trusted ports
= Expected DHCP-server/uplink direction

Untrusted ports
= Client-facing ports
```

It can block rogue DHCP server messages arriving from untrusted interfaces and build a DHCP binding table.

That binding information can support other features such as:

```text
Dynamic ARP Inspection
IP Source Guard
```

---

## Dynamic ARP Inspection

DAI validates ARP traffic against trusted information such as the DHCP snooping binding database or configured ARP ACLs.

Conceptually:

```text
ARP received
    ↓
Validate sender binding
    ↓
Valid?
 /      \
Yes      No
 |        |
Forward  Drop
```

This helps mitigate ARP spoofing.

---

## STP Protections

### BPDU Guard

Protects edge ports where BPDUs should never appear.

```text
PortFast edge port
      ↓
Unexpected BPDU
      ↓
Port placed into error-disabled state
```

### Root Guard

Prevents an interface from accepting a superior BPDU that would make an unexpected device the STP root.

These solve different problems and should not be treated as interchangeable.

---

# Anti-Spoofing

A network should reject traffic with source addresses that are impossible or unauthorized on the receiving interface.

Possible controls include:

```text
Ingress ACLs
Unicast Reverse Path Forwarding (uRPF)
IP Source Guard
Firewall policy
Provider anti-spoofing
```

The exact mechanism depends on routing symmetry and topology.

Strict uRPF can be inappropriate on asymmetric paths.

---

# Routing Security

## BGP

Common protections include:

```text
Explicit prefix filtering
AS-path filtering
Maximum-prefix limits
Peer authentication where required
Infrastructure ACLs
Route-policy controls
RPKI-based origin validation where supported in the architecture
```

A valid BGP session does not imply the routes received are trustworthy.

---

## IGPs

For protocols such as OSPF or IS-IS:

```text
Restrict adjacencies to intended interfaces
Use protocol authentication where supported and required
Make user-facing interfaces passive
Protect control-plane resources
Filter route redistribution carefully
```

Routing security is primarily about preventing unauthorized adjacency, route injection, and control-plane overload.

---

# Device Hardening

A hardened network device should expose only required services.

Typical controls:

```text
Disable unused services
Disable unused interfaces
Use SSH instead of Telnet
Use HTTPS instead of HTTP
Use SNMPv3 instead of SNMPv1/v2c where feasible
Restrict management-source networks
Use AAA
Use secure NTP design
Apply control-plane protections
Keep software supported and patched
Back up configuration
Centralize logs
```

---

# Logging, Telemetry, and Detection

Preventive controls will eventually miss something.

Useful evidence includes:

```text
Syslog
SNMP / telemetry
NetFlow / IPFIX
Firewall events
IDS / IPS events
AAA logs
VPN logs
DNS logs
Endpoint events
Configuration changes
```

A useful security-monitoring pipeline is:

```text
Collect
  ↓
Normalize
  ↓
Correlate
  ↓
Detect
  ↓
Investigate
  ↓
Contain / Respond
```

Logging without alerting, retention, time synchronization, and review provides limited security value.

---

# Zero Trust

Zero Trust is a security model based on explicit verification and least privilege rather than assuming that traffic is trustworthy because it is "inside" the network.

Core principles:

```text
Verify explicitly
Use least privilege
Assume breach
Continuously evaluate context
Limit lateral movement
```

It is an architectural approach, not a single product or protocol.

---

## 3. Why and When It Is Used

Network security is required anywhere connectivity creates risk to systems, users, or data.

It is particularly important in:

```text
Enterprise networks
Campus networks
Data centers
Internet edges
Cloud environments
Remote-access networks
Wireless networks
Industrial/OT environments
Service-provider networks
```

The practical goals are to:

- Prevent unauthorized access.
- Limit lateral movement.
- Protect management and routing infrastructure.
- Encrypt sensitive traffic across untrusted links.
- Detect attacks and policy violations.
- Preserve service availability.
- Produce evidence for response, audit, and compliance.

Security controls should be proportional to:

```text
Asset value
Threat model
Exposure
Business impact
Compliance requirements
Operational capability
```

Security that cannot be operated correctly can create both outages and blind spots.

---

## 4. Key Configuration, Parameters, or CLI

> **Platform:** Cisco IOS / IOS XE unless otherwise stated.  
> Exact security features and syntax vary by hardware family and software release. Validate feature support before production deployment.

---

# Secure Management — IOS / IOS XE

## SSH

```cisco
hostname R1
ip domain name example.com
crypto key generate rsa modulus 2048

username netadmin privilege 15 secret <secret>

line vty 0 15
 login local
 transport input ssh
```

Restrict VTY access with a standard ACL:

```cisco
ip access-list standard MGMT_ONLY
 permit 10.10.10.0 0.0.0.255

line vty 0 15
 access-class MGMT_ONLY in
```

For production, centralized AAA is preferable where infrastructure exists.

---

## AAA Framework

```cisco
aaa new-model
```

AAA server-group, method-list, authorization, and accounting configuration depends on whether RADIUS or TACACS+ is used and on the enterprise identity design.

Do not enable centralized AAA in production without a tested local fallback and verified management path.

---

# Interface ACL — IOS / IOS XE

```cisco
ip access-list extended USERS_TO_SERVER
 10 permit tcp 10.10.10.0 0.0.0.255 host 172.16.20.10 eq 443
 20 deny ip any any log
```

Apply:

```cisco
interface Vlan10
 ip access-group USERS_TO_SERVER in
```

Verify:

```cisco
show ip access-lists
show ip interface Vlan10
```

Use `log` selectively; high-volume logging can affect CPU and logging infrastructure.

---

# DHCP Snooping — Catalyst IOS / IOS XE

```cisco
ip dhcp snooping
ip dhcp snooping vlan 10
```

Trust only the legitimate DHCP-server/uplink direction:

```cisco
interface GigabitEthernet1/0/48
 ip dhcp snooping trust
```

Client-facing ports remain untrusted by default when snooping is active for the VLAN.

Verify:

```cisco
show ip dhcp snooping
show ip dhcp snooping binding
```

---

# Dynamic ARP Inspection — Catalyst IOS / IOS XE

```cisco
ip arp inspection vlan 10
```

Trust an infrastructure-facing interface only when the design requires it:

```cisco
interface GigabitEthernet1/0/48
 ip arp inspection trust
```

Verify:

```cisco
show ip arp inspection
```

> DAI depends on a valid trust/binding design. Enabling it without DHCP snooping bindings or appropriate ARP ACLs can disrupt legitimate hosts.

---

# BPDU Guard — Catalyst IOS / IOS XE

On an edge interface:

```cisco
interface GigabitEthernet1/0/10
 spanning-tree portfast
 spanning-tree bpduguard enable
```

Verify:

```cisco
show spanning-tree interface GigabitEthernet1/0/10 detail
```

Do not use BPDU Guard on a normal switch-to-switch link that is expected to receive BPDUs.

---

# Cisco Secure Firewall / FTD

> **Platform:** Cisco Secure Firewall Threat Defense managed by FMC or the supported management workflow.

Core policy areas include:

```text
Access Control Policy
Security zones
Network/object groups
Intrusion policy
URL/application controls
NAT
Site-to-site VPN
Remote-access VPN
Logging
```

Verification should use the exact FMC/FTD release documentation because available CLI and diagnostic workflows vary by release.

Useful operational evidence includes:

```text
Connection events
Intrusion events
Rule hit counts
NAT behavior
Routing table
VPN session state
Packet captures
```

Do not copy IOS XE ACL syntax into FTD policy configuration.

---

## Practical Troubleshooting Sequence

```text
1. Define the expected permitted flow.
        ↓
2. Identify source/destination zones and interfaces.
        ↓
3. Verify routing in both directions.
        ↓
4. Verify NAT behavior.
        ↓
5. Check ACL/firewall policy and rule order.
        ↓
6. Confirm session/state creation.
        ↓
7. Check IDS/IPS or security inspection verdicts.
        ↓
8. Verify return traffic.
        ↓
9. Check endpoint/application behavior.
        ↓
10. Review logs/counters/packet capture.
```

For infrastructure-security failures, also verify:

```text
Management path
AAA reachability
Control-plane policy
DHCP snooping / DAI state
STP protections
Routing adjacencies
Time synchronization
```

---

## 5. Common Gotchas and Misconceptions

### A Firewall Makes the Internal Network Trusted

**Incorrect.**

An internal compromise can still move laterally if internal segmentation and identity policy are weak.

```text
Perimeter security
≠
Internal trust
```

---

### VLANs Alone Provide Security

**Incorrect.**

VLANs separate Layer 2 domains.

Security requires enforcement between them:

```text
ACL
Firewall
Identity policy
TrustSec / segmentation policy
```

---

### NAT Is a Security Control

**Misleading.**

NAT changes addresses.

It may incidentally hide internal addressing, but authorization should come from:

```text
Firewall policy
ACLs
Identity / segmentation controls
```

NAT is not a substitute for a firewall.

---

### Encryption Means the Traffic Is Safe

**Incorrect.**

Encryption protects confidentiality/integrity in transit.

It does not prove that:

```text
The user is authorized
The application is safe
The endpoint is uncompromised
The traffic should be permitted
```

---

### An ACL Is Equivalent to a Stateful Firewall

**Incorrect.**

A basic ACL evaluates packet fields statelessly.

A firewall can track session state and often apply deeper inspection.

---

### Blocking All ICMP Is More Secure

**Incorrect or Unsafe.**

ICMP and ICMPv6 support required network functions including:

```text
Error reporting
Path MTU Discovery
IPv6 Neighbor Discovery
Diagnostics
```

Filter by function and threat model rather than blocking indiscriminately.

---

### More Security Rules Always Mean More Security

**Incorrect.**

Large, duplicated, or poorly ordered policies increase:

```text
Misconfiguration risk
Shadowed rules
Operational complexity
Troubleshooting time
```

A smaller explicit policy is often safer.

---

### Internal Traffic Does Not Need Monitoring

**Incorrect.**

Many attacks involve:

```text
Credential theft
Lateral movement
Internal reconnaissance
Compromised endpoints
```

East-west visibility is important where the risk justifies it.

---

### 802.1X Replaces Firewalling

**Incorrect.**

802.1X controls **network admission and access context**.

Firewall/segmentation controls determine what the admitted endpoint can reach.

---

### IDS and IPS Are the Same

**Incorrect.**

```text
IDS
= Detect / alert

IPS
= Inline detect + prevent
```

An IPS can directly affect production traffic, so policy tuning matters.

---

### Zero Trust Means Trust Nothing and Block Everything

**Incorrect.**

Zero Trust means trust is not granted solely by network location.

Access is explicitly evaluated using identity, device, context, and policy.

---

## 6. Trade-Offs

### Best Practice

- Use **least privilege** for users, devices, applications, and administrators.
- Separate user, server, guest, management, and security-sensitive zones.
- Restrict the management plane to trusted sources.
- Use centralized AAA and logging where operationally appropriate.
- Prefer encrypted management protocols.
- Apply Layer 2 protections at the access edge.
- Use explicit inbound and outbound routing/security policy at trust boundaries.
- Preserve required control protocols such as ICMPv6 rather than blocking blindly.
- Validate security changes with counters, logs, packet capture, and application tests.

---

### Context-Dependent Trade-Off — Segmentation Granularity

```text
More segmentation
+ Smaller blast radius
+ Better least privilege
+ Better policy visibility
- More routing/policy complexity
- More operational overhead
```

Segment based on risk and application dependency, not merely because smaller segments are possible.

---

### Context-Dependent Trade-Off — Inline IPS

```text
Inline IPS
+ Can block threats immediately
+ Strong enforcement point
- False positives can disrupt production
- Adds processing/latency requirements
```

Use staged policy, monitoring, and capacity validation before aggressive prevention.

---

### Context-Dependent Trade-Off — Strict uRPF

```text
Strict uRPF
+ Strong source-spoofing protection
- Can drop legitimate traffic in asymmetric-routing designs
```

Use only when the routing topology supports the selected uRPF mode.

---

### Context-Dependent Trade-Off — Centralized vs Distributed Enforcement

**Centralized Firewalling**

```text
+ Fewer policy points
+ Deep inspection
+ Central visibility
- Traffic hairpinning
- Larger firewall failure/capacity domain
```

**Distributed Enforcement**

```text
+ Policy closer to workload/user
+ Reduced lateral movement
+ Better scale in some designs
- More policy-distribution complexity
- Harder troubleshooting without strong tooling
```

Choose based on topology, scale, application flows, and operational maturity.

---

### Incorrect or Unsafe

- Exposing device-management services to untrusted networks without explicit controls.
- Using shared administrator credentials.
- Enabling DAI, DHCP snooping, AAA, or control-plane policies in production without validating dependencies and rollback.
- Treating NAT, VLANs, or encryption as substitutes for authorization.
- Leaving unrestricted east-west connectivity by default in high-value environments.
- Blocking required routing, DHCP, DNS, ICMP/ICMPv6, or authentication traffic without understanding the dependency.
- Applying security rules without verifying the return path, stateful behavior, and application requirements.

---

## Quick Reference

```text
Network Security
= Protect networked systems, traffic, and infrastructure

Primary Objectives
= Confidentiality
= Integrity
= Availability

AAA
= Authentication
= Authorization
= Accounting

802.1X
= Network access control

Segmentation
= Limit reachability and lateral movement

ACL
= Stateless packet classification/filtering

Stateful Firewall
= Session-aware enforcement

IDS
= Detect / alert

IPS
= Detect + block inline

VPN / IPsec / TLS / MACsec
= Protect traffic in transit

DHCP Snooping
= Blocks rogue DHCP behavior
= Builds binding table

DAI
= Validates ARP against trusted bindings/policy

BPDU Guard
= Protects edge ports from unexpected BPDUs

Anti-Spoofing
= ACL / uRPF / IP Source Guard / firewall policy

Management Plane
= Restrict heavily
= AAA + SSH/HTTPS + logging

Control Plane
= Protect routing and infrastructure protocols

Zero Trust
= Verify explicitly
= Least privilege
= Assume breach

Core Rule
= Identity + segmentation + enforcement + visibility + response
  are all required; no single control provides complete network security.
```

</div>
