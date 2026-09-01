<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# Virtual Private Network (VPN)

> **Core idea:** A VPN creates a logical private path across a shared or untrusted network. In secure enterprise VPNs, the peers authenticate each other and encapsulate traffic so it can be encrypted, integrity-protected, and carried between sites or remote users without exposing the original traffic to the transit network.

---

## 1. What It Is

A **Virtual Private Network (VPN)** provides logical private connectivity across another network, commonly the Internet. Enterprise VPNs typically use **IPsec** or **TLS-based remote access** to protect traffic between VPN endpoints.

> **VPN does not automatically mean encryption.** Some VPN technologies provide traffic separation without cryptographic protection. When confidentiality over an untrusted network is required, use a VPN technology that explicitly provides encryption and integrity.

---

## 2. How It Works

## VPN Architecture

A VPN creates an **overlay** across an existing **underlay** network.

```text
Private Network A                         Private Network B
10.10.0.0/16                             10.20.0.0/16
      |                                        |
      v                                        v
+-------------+                          +-------------+
| VPN Peer A  |==========================| VPN Peer B  |
+-------------+      Secure Tunnel       +-------------+
      \                                        /
       \______________ Internet ______________/
                    Underlay
```

The underlay must first provide IP reachability between the VPN endpoints.

```text
Underlay
= Reaches the VPN peer addresses

Overlay
= Carries the protected private traffic
```

A working tunnel therefore depends on both:

```text
Peer reachability
+
Correct VPN negotiation and forwarding
```

---

# Main VPN Models

## Site-to-Site VPN

A **site-to-site VPN** connects networks through VPN gateways.

```text
Branch LAN
10.10.0.0/16
     |
     v
 Branch Router
      ║
      ║ IPsec
      ║
 HQ Firewall
     |
     v
HQ LAN
10.20.0.0/16
```

Endpoints behind the VPN gateways do not normally run VPN software themselves.

Typical use:

```text
Branch ↔ Headquarters
Data Center ↔ Data Center
Enterprise ↔ Cloud
Enterprise ↔ Partner
```

---

## Remote-Access VPN

A **remote-access VPN** connects an individual endpoint to an enterprise VPN gateway.

```text
Remote User
Cisco Secure Client
      |
      | Internet
      |
      ║ Secure VPN
      ║
Cisco Secure Firewall
      |
      v
Enterprise Network
```

The VPN gateway normally performs:

```text
User/device authentication
VPN address assignment
Authorization / group policy
Route or split-tunnel assignment
Security-policy enforcement
```

Cisco Secure Firewall remote access can use **TLS/DTLS** or **IPsec with IKEv2**, depending on the deployed policy and client capabilities.

---

# IPsec VPN Mechanics

IPsec is the primary technology used for secure site-to-site VPNs and is also used for some remote-access VPNs.

The process can be separated into:

```text
Control Plane
= Authenticate peers
= Negotiate cryptography
= Establish Security Associations

Data Plane
= Encrypt / authenticate user packets
= Encapsulate them across the tunnel
```

---

## Step 1 — Reach the VPN Peer

Before negotiation can begin:

```text
Peer A must route to Peer B
Peer B must route to Peer A
```

If the outer peer addresses are unreachable, IPsec cannot establish.

Check underlay reachability first.

---

## Step 2 — IKEv2 Establishes the Control Relationship

**IKEv2 (Internet Key Exchange version 2)** negotiates the parameters used to secure the VPN and authenticates the peers.

IKEv2 begins using:

```text
UDP 500
```

The first major exchange is:

```text
IKE_SA_INIT
```

It negotiates information such as:

```text
Encryption algorithm
Integrity / authenticated-encryption parameters
Pseudo-random function
Diffie-Hellman group
Nonces
Key-exchange material
```

The next major exchange is:

```text
IKE_AUTH
```

It authenticates the peers and establishes the first IPsec **CHILD_SA**.

Authentication commonly uses:

```text
Pre-shared keys
Digital certificates
EAP / user authentication in applicable remote-access designs
```

Conceptually:

```text
Peer A                              Peer B

IKE_SA_INIT ----------------------->
           <---------------- IKE_SA_INIT

IKE_AUTH -------------------------->
           <------------------ IKE_AUTH

IKE SA established
IPsec CHILD SAs established
```

> IKEv2 should be understood in terms of **IKE SA** and **CHILD SA** exchanges rather than forcing the older IKEv1 "Phase 1 / Phase 2" terminology onto it.

---

## Security Associations (SAs)

A **Security Association (SA)** describes how traffic is protected.

The IKE SA protects IKE control traffic.

The IPsec CHILD SAs protect user data.

IPsec SAs are directional:

```text
A → B SA
B → A SA
```

A functioning bidirectional VPN therefore uses inbound and outbound IPsec SA state.

Each SA includes information such as:

```text
Security Parameter Index (SPI)
Encryption/integrity parameters
Keys
Traffic selectors
Lifetime
Sequence / anti-replay state
```

---

## Step 3 — Traffic Is Selected for the VPN

The VPN must determine which traffic belongs in the tunnel.

Two common models are:

### Policy-Based VPN

Traffic is matched using configured selectors such as:

```text
Source subnet
Destination subnet
Protocol / ports when applicable
```

Conceptually:

```text
10.10.0.0/16 → 10.20.0.0/16
        ↓
Matches VPN policy
        ↓
Encrypt
```

Traffic that does not match the selectors is not protected by that IPsec policy.

Traditional crypto-map VPNs are policy based.

---

### Route-Based VPN

The VPN is represented by a logical tunnel interface such as a VTI.

```text
Routing table
      ↓
Tunnel interface
      ↓
IPsec protection
```

The routing table determines which traffic enters the tunnel.

Route-based VPNs generally integrate more naturally with:

```text
Static routing
OSPF / EIGRP / BGP where supported by the design
Multiple protected prefixes
Redundant routed topologies
```

> **Routing and encryption policy are separate decisions.** A tunnel can be cryptographically established while the data path still fails because the required route is missing.

---

## Step 4 — ESP Protects the Data

Modern IPsec VPNs normally use **Encapsulating Security Payload (ESP)**.

```text
ESP = IP protocol 50
```

ESP can provide:

```text
Confidentiality
Integrity/authentication
Anti-replay protection
```

For a typical site-to-site VPN, IPsec uses **tunnel mode**.

Original packet:

```text
+-------------------------------+
| Original IP | TCP/UDP | Data  |
+-------------------------------+
```

After ESP tunnel-mode encapsulation:

```text
+------------------------------------------------------+
| Outer IP | ESP | Original IP | TCP/UDP | Data | ESP |
+------------------------------------------------------+
```

Conceptually:

```text
Original private packet
        ↓
Encrypt/protect
        ↓
Add ESP information
        ↓
Add outer IP header
        ↓
Send across underlay
```

The outer IP header identifies the VPN peers.

The encrypted inner packet retains the original communicating endpoints.

---

## IPsec Tunnel Mode vs Transport Mode

### Tunnel Mode

Protects the original IP packet by encapsulating it inside a new outer IP packet.

```text
[Outer IP][ESP][Original IP][Payload]
```

Typical use:

```text
Gateway-to-gateway site-to-site VPN
```

---

### Transport Mode

Protects the payload of the original IP packet while retaining the original outer IP header.

```text
[Original IP][ESP][Payload]
```

Used in specific designs, including some tunnel combinations such as GRE protected by IPsec.

For most normal site-to-site IPsec designs:

```text
Tunnel mode
```

is the expected model.

---

## NAT Traversal (NAT-T)

Native ESP has no TCP or UDP port number.

If NAT/PAT exists between the VPN peers, the peers can use **NAT Traversal**.

```text
IKE initially
= UDP 500

NAT detected
      ↓
IPsec/IKE uses UDP 4500
```

The protected traffic is encapsulated so that NAT/PAT devices can process it correctly.

```text
IPsec ESP
      ↓
UDP 4500 encapsulation
      ↓
NAT/PAT device
      ↓
Internet
```

> Seeing UDP/4500 does not mean IPsec is absent; it commonly means IPsec is operating through NAT-T.

---

## Rekey and Peer Liveness

VPN keys and SAs have finite lifetimes.

Before an SA expires:

```text
Existing SA
    ↓
Rekey
    ↓
New key material / SA
    ↓
Old SA removed
```

IKEv2 also supports peer-liveness mechanisms such as Dead Peer Detection behavior so a peer can remove stale state when the remote endpoint is no longer reachable.

Failures during rekey or liveness processing can produce intermittent VPN outages even when the initial tunnel establishes correctly.

---

# Remote-Access VPN Mechanics

A remote-access VPN adds user/device policy to the tunnel process.

Typical sequence:

```text
Remote client reaches VPN gateway
        ↓
TLS or IKEv2 session established
        ↓
Gateway identity validated
        ↓
User/device authentication
        ↓
Authorization / group policy
        ↓
VPN address assigned
        ↓
Routes / split-tunnel policy delivered
        ↓
Protected application traffic begins
```

Authentication may involve:

```text
Certificates
Username/password
RADIUS
LDAP / Active Directory
MFA through integrated identity services
```

The exact authentication chain depends on the VPN gateway and policy design.

---

## Split Tunnel vs Full Tunnel

### Split Tunnel

Only selected enterprise traffic enters the VPN.

```text
Enterprise traffic → VPN
Internet traffic   → Local Internet connection
```

Benefits:

```text
Less VPN-headend bandwidth
Lower central Internet-egress load
Often better Internet performance
```

Security consideration:

```text
Some traffic bypasses enterprise inspection
```

---

### Full Tunnel

Most or all client traffic is sent through the VPN gateway.

```text
Enterprise traffic → VPN
Internet traffic   → VPN → Enterprise security stack
```

Benefits:

```text
Central policy enforcement
Central logging/inspection
Consistent egress controls
```

Cost:

```text
Higher VPN and Internet-edge bandwidth
More headend capacity required
```

---

# MTU and MSS

VPN encapsulation adds headers.

Example:

```text
Original packet
      +
ESP
      +
Outer IP
      +
Possible UDP 4500
```

The resulting packet may exceed the physical-path MTU.

Symptoms can include:

```text
Ping works
Small sessions work
Large transfers stall
Some websites/applications fail
```

Operational fixes depend on the design and may include:

```text
Correct tunnel MTU
TCP MSS adjustment
Working Path MTU Discovery
Preserving required ICMP / ICMPv6 messages
```

> Do not treat MTU as an application problem until the full encapsulation overhead has been considered.

---

## 3. Why and When It Is Used

Use VPNs when private or protected connectivity is required across infrastructure that is not physically dedicated to the communicating endpoints.

Typical use cases:

```text
Branch-to-headquarters connectivity
Remote employee access
Cloud connectivity
Partner/extranet connectivity
Encryption across the Internet
Encryption across a provider WAN when policy/compliance requires it
```

### Site-to-Site VPN Is Appropriate When

```text
Multiple systems at each location need connectivity
Traffic should be transparent to endpoints
Sites have stable VPN gateways
```

### Remote-Access VPN Is Appropriate When

```text
Individual users connect from arbitrary networks
User/device authentication is required
Corporate access policy must follow the user
```

### VPN Is Not a Substitute For

```text
Routing
Firewall policy
Endpoint security
Identity policy
High availability
Application security
```

A VPN can securely deliver traffic that should never have been permitted in the first place. Encryption and authorization are separate controls.

---

## 4. Key Configuration, Parameters, or CLI

VPN syntax differs significantly across Cisco IOS XE, ASA, and Secure Firewall. Do not copy syntax between platforms.

---

# Cisco IOS XE — IPsec / FlexVPN / VTI

A route-based IKEv2/IPsec design normally contains these configuration building blocks:

```text
IKEv2 proposal
      ↓
IKEv2 policy
      ↓
Peer authentication / keyring or PKI
      ↓
IKEv2 profile
      ↓
IPsec transform / proposal
      ↓
IPsec profile
      ↓
Tunnel interface / VTI
      ↓
Routing through tunnel
```

The exact syntax depends on IOS XE release, authentication method, static/dynamic peers, and whether the design uses FlexVPN or another IPsec model.

### Core Verification

```cisco
show crypto ikev2 sa
show crypto ipsec sa
show crypto session detail
show interfaces tunnel
show ip route
```

Interpretation:

```text
No IKEv2 SA
→ Peer reachability, IKE policy, or authentication problem

IKEv2 SA Up
No IPsec/CHILD SA
→ IPsec proposal or traffic-selector problem

IPsec SA Up
No traffic / counters
→ Routing, selectors, NAT, ACL, or application-path problem
```

---

# Cisco ASA — Site-to-Site IPsec

Traditional ASA site-to-site VPNs commonly use:

```text
IKEv2 policy
Tunnel group
IPsec proposal
Crypto map
Traffic selectors / ACL
NAT exemption or appropriate NAT policy
Interface crypto-map application
```

### Core Verification

```cisco
show crypto ikev2 sa
show crypto ipsec sa
show route
show xlate
```

Validate:

```text
Peer authentication
IKE SA state
IPsec encaps/decaps counters
Routing
NAT behavior
```

> ASA software releases support multiple VPN models. Verify command syntax and supported cryptographic algorithms against the exact release before deploying production configuration.

---

# Cisco Secure Firewall Threat Defense (FTD)

Secure Firewall VPN policy is normally configured through **Firewall Management Center (FMC)** or the supported device-management workflow rather than by copying IOS XE VPN syntax.

Typical policy areas include:

```text
Site-to-site VPN
Remote-access VPN
Peer/interface selection
IKE/IPsec parameters
Authentication
Address pools
Group policy
Split tunneling
NAT
Access Control Policy
```

### FMC Remote Access

Common workflow:

```text
Devices
→ VPN
→ Remote Access
```

Secure Firewall remote access can support:

```text
TLS-based remote access
IPsec-IKEv2 remote access
Cisco Secure Client
```

### Core Verification

```cisco
show vpn-sessiondb detail
show vpn-sessiondb detail l2l
```

These commands provide VPN session information, including site-to-site/IKEv2/IPsec session state where applicable.

For software-version-specific diagnostics, confirm the supported CLI through the relevant Cisco Secure Firewall release documentation.

---

## Useful Packet-Level Checks

When packet capture is available, verify:

```text
UDP 500
= IKE negotiation

UDP 4500
= IKE/IPsec NAT Traversal

IP protocol 50
= Native ESP
```

Then determine whether failure occurs at:

```text
Underlay reachability
IKE authentication
CHILD SA / IPsec negotiation
Encrypted data forwarding
Decryption
Post-decryption routing/policy
Return path
```

---

## Practical Troubleshooting Sequence

```text
1. Can the peers reach each other's outer IP addresses?
        ↓
2. Is UDP 500/4500 or ESP being blocked?
        ↓
3. Does the IKEv2 SA establish?
        ↓
4. Do authentication identities/credentials match?
        ↓
5. Does the IPsec CHILD SA establish?
        ↓
6. Do traffic selectors or tunnel routes match the intended networks?
        ↓
7. Is NAT bypass/exemption correct where required?
        ↓
8. Are firewall/ACL policies permitting the traffic?
        ↓
9. Do encrypt/encaps counters increase?
        ↓
10. Do decrypt/decaps counters increase?
        ↓
11. Is the post-decryption route correct?
        ↓
12. Is the return path valid?
        ↓
13. Is MTU/MSS causing only larger packets to fail?
```

---

## 5. Common Gotchas and Misconceptions

### A VPN Tunnel Being "Up" Means Applications Must Work

**Incorrect.**

An IKE SA only proves that part of the control plane succeeded.

```text
IKE Up
≠
IPsec forwarding correct
≠
Routing correct
≠
Firewall policy correct
≠
Application reachable
```

Verify the data plane separately.

---

### VPN Automatically Means Encrypted

**Incorrect.**

VPN describes logical private connectivity.

Encryption depends on the specific VPN technology.

```text
IPsec VPN
= Cryptographic protection

Provider VPN / traffic separation
= May not be encrypted
```

---

### IPsec Replaces Routing

**Incorrect.**

The VPN protects traffic; the device still needs to know where to send it.

Route-based VPN:

```text
Route → Tunnel
```

Policy-based VPN:

```text
Route + matching encryption policy
```

Both still depend on valid routing.

---

### IPsec Replaces Firewall Policy

**Incorrect.**

Encryption proves protection in transit, not authorization.

A VPN can transport unwanted traffic securely.

Apply explicit policy to:

```text
Which networks
Which users
Which applications
Which directions
```

---

### IKEv2 Up Means the IPsec SA Is Up

**Incorrect.**

IKEv2 creates the secure control relationship, but the data-plane CHILD SA can still fail because of:

```text
Traffic-selector mismatch
IPsec proposal mismatch
Policy error
Authentication/authorization conditions
```

Check both:

```cisco
show crypto ikev2 sa
show crypto ipsec sa
```

---

### NAT and VPN Are Independent

**Incorrect.**

NAT can affect:

```text
VPN peer reachability
Traffic selectors
Protected source/destination addresses
Return routing
```

Site-to-site firewall designs frequently require explicit NAT exemption or identity-NAT logic for protected networks.

Always evaluate NAT **before assuming an IPsec failure**.

---

### ESP Uses TCP or UDP Port 50

**Incorrect.**

Native ESP is:

```text
IP protocol 50
```

not TCP/UDP port 50.

With NAT-T:

```text
ESP-protected traffic
→ UDP 4500 encapsulation
```

---

### UDP 4500 Means the VPN Is Not Using IPsec

**Incorrect.**

UDP 4500 commonly indicates **IPsec NAT Traversal**.

---

### A Pre-Shared Key Alone Makes a VPN Secure

**Incorrect.**

VPN security also depends on:

```text
Strong cryptographic algorithms
Key strength
Peer identity validation
Credential protection
PFS/rekey design
Access policy
Endpoint security
Patch level
Logging/monitoring
```

---

### Split Tunneling Is Always Insecure

**Too broad.**

Split tunneling reduces centralized inspection, but it may be appropriate when endpoint controls, DNS/security policy, SaaS architecture, bandwidth, and threat model support it.

Treat it as a security/design trade-off, not an automatic failure.

---

### Blocking ICMP Has No Effect on VPNs

**Incorrect or Unsafe.**

VPN encapsulation reduces usable MTU.

Blocking required ICMP or ICMPv6 messages can break Path MTU Discovery and produce difficult intermittent application failures.

---

## 6. Trade-Offs

### Best Practice

- Prefer **IKEv2** for new IPsec deployments when supported by all required peers.
- Use current cryptographic algorithms approved by the organization's security policy and current Cisco guidance; do not copy legacy DES/3DES/weak-hash examples.
- Authenticate peers strongly; use PKI where the scale and identity requirements justify it.
- Treat routing, NAT, VPN selectors, and firewall policy as separate controls and verify each independently.
- Monitor IKE/IPsec SA state, tunnel throughput, authentication failures, rekeys, and tunnel availability.
- Design MTU/MSS intentionally.
- Use redundant peers/headends where VPN availability is business-critical.

---

### Context-Dependent Trade-Off — Policy-Based vs Route-Based

**Policy-Based**

```text
+ Simple for small fixed subnet pairs
+ Common on legacy/third-party VPN implementations
- Harder to scale with many prefixes
- Less natural for dynamic routing
```

**Route-Based**

```text
+ Routing determines tunnel usage
+ Easier multiple-prefix operation
+ Better fit for dynamic routing and redundant topologies
- Requires compatible tunnel-interface support
```

For modern routed enterprise designs, route-based VPNs are often operationally simpler when both peers support them.

---

### Context-Dependent Trade-Off — Full Tunnel vs Split Tunnel

**Full Tunnel**

```text
+ Centralized security inspection
+ Consistent corporate egress policy
- More headend/bandwidth demand
```

**Split Tunnel**

```text
+ Lower VPN bandwidth
+ Better direct SaaS/Internet performance
- Some traffic bypasses central inspection
```

Choose based on security policy, endpoint controls, bandwidth, and application architecture.

---

### Context-Dependent Trade-Off — Pre-Shared Keys vs Certificates

**Pre-Shared Keys**

```text
+ Simple for small deployments
- Difficult key lifecycle at scale
- Shared secret may identify many peers if poorly designed
```

**Certificates**

```text
+ Strong scalable identity
+ Better revocation/lifecycle model
- Requires PKI design and operations
```

---

### Context-Dependent Trade-Off — Perfect Forward Secrecy (PFS)

PFS performs additional key exchange so compromise of one key does not expose other independently derived session keys.

```text
+ Stronger key-separation properties
- Additional negotiation/CPU cost
```

Use it according to current security policy, peer capability, and performance requirements.

---

### Incorrect or Unsafe

- Using obsolete cryptographic algorithms because they appear in an old configuration example.
- Exposing broad internal networks through a VPN without explicit firewall/authorization policy.
- Deploying a single VPN headend where the service has high availability requirements.
- Ignoring NAT behavior in site-to-site VPN troubleshooting.
- Clearing production VPN SAs indiscriminately without understanding the session impact.
- Reducing MTU or MSS blindly without measuring the actual encapsulation and path requirements.
- Assuming a green tunnel-status indicator proves end-to-end application reachability.

---

## Quick Reference

```text
VPN
= Logical private connectivity over another network

Underlay
= Carries the VPN outer packets

Overlay
= Private/protected connectivity

Site-to-Site
= Network ↔ Network

Remote Access
= User/device ↔ Enterprise

IPsec
= Common secure VPN data-plane technology

IKEv2
= Peer authentication + key/SA negotiation

IKE
= UDP 500

NAT-T
= UDP 4500

ESP
= IP protocol 50

IKE SA
= Protects IKE control traffic

CHILD SA
= Protects user data with IPsec

IPsec SA
= Directional

Tunnel Mode
= Protects original IP packet inside new outer IP header

Policy-Based VPN
= Select traffic with crypto selectors

Route-Based VPN
= Route traffic through tunnel interface

Split Tunnel
= Selected traffic through VPN

Full Tunnel
= Most/all client traffic through VPN

VPN Up
≠ Routing correct
≠ NAT correct
≠ Firewall policy correct
≠ Application healthy

Core Troubleshooting
= Underlay → IKE → IPsec SA → Routing → NAT → Policy → Return Path → MTU
```

## CCNA Configuration

VPN configuration is outside current CCNA 200-301 configuration scope.

## CCNP Configuration

**CCNP Enterprise — IOS-XE — GRE over IPsec (IKEv1 Pre-Shared Key)**

| Command | Description |
|---|---|
| **Create crypto ACL:**<br>`(config)#ip access-list extended <acl-name>`<br>&nbsp;&nbsp;○ `(config-ext-nacl)#permit gre host <local-public-ip> host <remote-public-ip>` | Matches GRE traffic protected by the IPsec crypto map. |
| **Create ISAKMP policy:**<br>`(config)#crypto isakmp policy <priority>` | Enters IKEv1 ISAKMP policy configuration mode. |
| **Set IKE encryption:**<br>`(config-isakmp)#encryption <des|3des|aes|aes 192|aes 256>` | Sets IKEv1 Phase 1 encryption algorithm. |
| **Set IKE hash:**<br>`(config-isakmp)#hash <sha|sha256|sha384|md5>` | Sets IKEv1 Phase 1 integrity algorithm. |
| **Set IKE authentication:**<br>`(config-isakmp)#authentication pre-share` | Selects pre-shared-key peer authentication. |
| **Set Diffie-Hellman group:**<br>`(config-isakmp)#group <1|2|5|14|15|16|19|20|24>` | Sets the IKEv1 Diffie-Hellman group. |
| **Configure peer pre-shared key:**<br>`(config)#crypto isakmp key <key-string> address <peer-ip> [<mask>]` | Associates a pre-shared key with the remote peer. |
| **Create IPsec transform set:**<br>`(config)#crypto ipsec transform-set <transform-set-name> <transform1> [<transform2> [<transform3>]]` | Creates the IPsec Phase 2 transform set. |
| **Set transport mode:**<br>`(cfg-crypto-trans)#mode transport` | Sets IPsec transport mode for GRE protection. |
| **Create crypto map:**<br>`(config)#crypto map <map-name> <sequence> ipsec-isakmp` | Creates an IPsec crypto-map entry. |
| **Match crypto ACL:**<br>`(config-crypto-map)#match address <acl-name>` | Associates interesting traffic with the crypto-map entry. |
| **Set VPN peer:**<br>`(config-crypto-map)#set peer <peer-ip>` | Assigns the remote IPsec peer. |
| **Set transform set:**<br>`(config-crypto-map)#set transform-set <transform-set-name>` | Associates the IPsec transform set with the crypto map. |
| **Apply crypto map:**<br>`(config)#interface <outside-interface>`<br>&nbsp;&nbsp;○ `(config-if)#crypto map <map-name>` | Applies the crypto map to the outside interface. |
| **Create GRE tunnel:**<br>`(config)#interface Tunnel<tunnel-number>`<br>&nbsp;&nbsp;○ `(config-if)#ip address <tunnel-ip> <subnet-mask>`<br>&nbsp;&nbsp;○ `(config-if)#tunnel source <source-interface|source-ip>`<br>&nbsp;&nbsp;○ `(config-if)#tunnel destination <peer-public-ip>` | Creates the GRE tunnel protected by IPsec. |

**CCNP Enterprise — IOS-XE — GRE over IPsec Using IPsec Profile**

| Command | Description |
|---|---|
| **Create transport transform set:**<br>`(config)#crypto ipsec transform-set <transform-set-name> <transform1> [<transform2>]`<br>&nbsp;&nbsp;○ `(cfg-crypto-trans)#mode transport` | Creates a transport-mode transform set for GRE. |
| **Create IPsec profile:**<br>`(config)#crypto ipsec profile <profile-name>` | Enters IPsec profile configuration mode. |
| **Attach transform set:**<br>`(ipsec-profile)#set transform-set <transform-set-name>` | Associates the transform set with the IPsec profile. |
| **Protect GRE tunnel:**<br>`(config)#interface Tunnel<tunnel-number>`<br>&nbsp;&nbsp;○ `(config-if)#tunnel protection ipsec profile <profile-name>` | Applies IPsec protection directly to the GRE tunnel. |

**CCNP Enterprise — IOS-XE — Static VTI over IPsec**

| Command | Description |
|---|---|
| **Create tunnel-mode transform set:**<br>`(config)#crypto ipsec transform-set <transform-set-name> <transform1> [<transform2>]`<br>&nbsp;&nbsp;○ `(cfg-crypto-trans)#mode tunnel` | Creates a tunnel-mode transform set for VTI. |
| **Create IPsec profile:**<br>`(config)#crypto ipsec profile <profile-name>`<br>&nbsp;&nbsp;○ `(ipsec-profile)#set transform-set <transform-set-name>` | Creates the IPsec profile used by the VTI. |
| **Create static VTI:**<br>`(config)#interface Tunnel<tunnel-number>`<br>&nbsp;&nbsp;○ `(config-if)#ip address <tunnel-ip> <subnet-mask>`<br>&nbsp;&nbsp;○ `(config-if)#tunnel source <source-interface|source-ip>`<br>&nbsp;&nbsp;○ `(config-if)#tunnel destination <peer-public-ip>`<br>&nbsp;&nbsp;○ `(config-if)#tunnel mode ipsec ipv4`<br>&nbsp;&nbsp;○ `(config-if)#tunnel protection ipsec profile <profile-name>` | Creates an IPv4 static VTI protected by IPsec. |
| **Create IPv6 VTI mode:**<br>`(config)#interface Tunnel<tunnel-number>`<br>&nbsp;&nbsp;○ `(config-if)#tunnel mode ipsec ipv6` | Sets the tunnel interface to IPv6 IPsec VTI mode. |

**CCNP Enterprise — IOS-XE — Dynamic VTI**

| Command | Description |
|---|---|
| **Create loopback source:**<br>`(config)#interface Loopback<id>`<br>&nbsp;&nbsp;○ `(config-if)#ip address <ipv4-address> 255.255.255.255` | Creates an address source for the dynamic tunnel template. |
| **Create virtual template:**<br>`(config)#interface Virtual-Template<id> type tunnel`<br>&nbsp;&nbsp;○ `(config-if)#ip unnumbered Loopback<id>`<br>&nbsp;&nbsp;○ `(config-if)#tunnel mode ipsec ipv4`<br>&nbsp;&nbsp;○ `(config-if)#tunnel protection ipsec profile <profile-name>` | Creates a dynamic VTI template protected by IPsec. |

**CCNP Enterprise / Security — IOS-XE — IKEv2 FlexVPN Peer Identity**

| Command | Description |
|---|---|
| **Create IKEv2 keyring:**<br>`(config)#crypto ikev2 keyring <keyring-name>` | Creates the IKEv2 peer keyring. |
| **Create keyring peer:**<br>`(config-ikev2-keyring)#peer <peer-name>`<br>&nbsp;&nbsp;○ `(config-ikev2-keyring-peer)#address <peer-ip>`<br>&nbsp;&nbsp;○ `(config-ikev2-keyring-peer)#pre-shared-key <key-string>` | Defines the peer address and pre-shared key. |
| **Create IKEv2 profile:**<br>`(config)#crypto ikev2 profile <profile-name>` | Enters IKEv2 profile configuration mode. |
| **Match remote FQDN:**<br>`(config-ikev2-profile)#match identity remote fqdn <remote-fqdn>` | Matches the remote peer identity by FQDN. |
| **Set local FQDN:**<br>`(config-ikev2-profile)#identity local fqdn <local-fqdn>` | Sets the local IKEv2 FQDN identity. |
| **Set local authentication:**<br>`(config-ikev2-profile)#authentication local pre-share` | Selects local pre-shared-key authentication. |
| **Set remote authentication:**<br>`(config-ikev2-profile)#authentication remote pre-share` | Selects remote pre-shared-key authentication. |
| **Attach local keyring:**<br>`(config-ikev2-profile)#keyring local <keyring-name>` | Associates the local IKEv2 keyring with the profile. |

**CCNP Enterprise / Security — IOS-XE — VPN Verification**

| Command | Description |
|---|---|
| **Show IKEv1 SAs:**<br>`#show crypto isakmp sa` | Displays active IKEv1 security associations. |
| **Show IKEv2 SAs:**<br>`#show crypto ikev2 sa detailed` | Displays detailed IKEv2 security-association information. |
| **Show IKEv2 sessions:**<br>`#show crypto ikev2 session` | Displays active IKEv2 sessions and child SAs. |
| **Show detailed IKEv2 sessions:**<br>`#show crypto ikev2 session detailed` | Displays detailed IKEv2 session information. |
| **Show IPsec SAs:**<br>`#show crypto ipsec sa` | Displays IPsec security associations and packet counters. |
| **Show tunnel interface:**<br>`#show interface Tunnel<tunnel-number>` | Displays tunnel state, protocol, addressing, and counters. |
| **Show IKEv2 statistics:**<br>`#show crypto ikev2 stats` | Displays IKEv2 protocol statistics and counters. |
| **Show IKEv2 event errors:**<br>`#show monitor event-trace crypto ikev2 error all` | Displays recorded IKEv2 error events. |
| **Show IPsec event trace:**<br>`#show monitor event-trace crypto ipsec` | Displays recorded IPsec event-trace information. |

**CCNP Security — ASA 9.x — Site-to-Site IKEv2 Policy**

| Command | Description |
|---|---|
| **Enable IKEv2:**<br>`(config)#crypto ikev2 enable <interface-name>` | Enables IKEv2 negotiation on the selected ASA interface. |
| **Create IKEv2 policy:**<br>`(config)#crypto ikev2 policy <priority>` | Enters ASA IKEv2 policy configuration mode. |
| **Set encryption:**<br>`(config-ikev2-policy)#encryption <algorithm>` | Sets IKEv2 encryption algorithm. |
| **Set integrity:**<br>`(config-ikev2-policy)#integrity <algorithm>` | Sets IKEv2 integrity algorithm. |
| **Set DH group:**<br>`(config-ikev2-policy)#group <dh-group>` | Sets IKEv2 Diffie-Hellman group. |
| **Set PRF:**<br>`(config-ikev2-policy)#prf <algorithm>` | Sets the IKEv2 pseudo-random function. |
| **Set IKE lifetime:**<br>`(config-ikev2-policy)#lifetime seconds <seconds>` | Sets the IKEv2 security-association lifetime. |

**CCNP Security — ASA 9.x — Site-to-Site Peer Authentication**

| Command | Description |
|---|---|
| **Create L2L tunnel group:**<br>`(config)#tunnel-group <peer-ip> type ipsec-l2l` | Creates a LAN-to-LAN IPsec connection profile. |
| **Enter IPsec attributes:**<br>`(config)#tunnel-group <peer-ip> ipsec-attributes` | Enters tunnel-group IPsec attribute configuration mode. |
| **Set remote pre-shared key:**<br>`(config-tunnel-ipsec)#ikev2 remote-authentication pre-shared-key <key-string>` | Configures the remote IKEv2 authentication key. |
| **Set local pre-shared key:**<br>`(config-tunnel-ipsec)#ikev2 local-authentication pre-shared-key <key-string>` | Configures the local IKEv2 authentication key. |

**CCNP Security — ASA 9.x — Site-to-Site IPsec Proposal**

| Command | Description |
|---|---|
| **Create IKEv2 IPsec proposal:**<br>`(config)#crypto ipsec ikev2 ipsec-proposal <proposal-name>` | Enters ASA IKEv2 IPsec proposal configuration mode. |
| **Set ESP encryption:**<br>`(config-ipsec-proposal)#protocol esp encryption <algorithm>` | Sets ESP encryption for the proposal. |
| **Set ESP integrity:**<br>`(config-ipsec-proposal)#protocol esp integrity <algorithm>` | Sets ESP integrity for the proposal. |

**CCNP Security — ASA 9.x — Site-to-Site Crypto Map**

| Command | Description |
|---|---|
| **Create crypto ACL:**<br>`(config)#access-list <acl-name> extended permit ip <local-network> <local-mask> <remote-network> <remote-mask>` | Defines protected traffic for the site-to-site VPN. |
| **Match crypto ACL:**<br>`(config)#crypto map <map-name> <sequence> match address <acl-name>` | Associates protected traffic with the crypto-map entry. |
| **Set VPN peer:**<br>`(config)#crypto map <map-name> <sequence> set peer <peer-ip>` | Assigns the remote site-to-site VPN peer. |
| **Set IKEv2 proposal:**<br>`(config)#crypto map <map-name> <sequence> set ikev2 ipsec-proposal <proposal-name>` | Associates the IKEv2 IPsec proposal with the peer. |
| **Enable PFS:**<br>`(config)#crypto map <map-name> <sequence> set pfs group<dh-group>` | Enables Perfect Forward Secrecy for the crypto-map entry. |
| **Apply crypto map:**<br>`(config)#crypto map <map-name> interface <interface-name>` | Applies the crypto map to the terminating interface. |

**CCNP Security — ASA 9.x — VPN NAT Exemption**

| Command | Description |
|---|---|
| **Create local network object:**<br>`(config)#object network <local-object>`<br>&nbsp;&nbsp;○ `(config-network-object)#subnet <local-network> <local-mask>` | Creates the local VPN network object. |
| **Create remote network object:**<br>`(config)#object network <remote-object>`<br>&nbsp;&nbsp;○ `(config-network-object)#subnet <remote-network> <remote-mask>` | Creates the remote VPN network object. |
| **Configure identity NAT:**<br>`(config)#nat (<inside-name>,<outside-name>) source static <local-object> <local-object> destination static <remote-object> <remote-object>` | Exempts site-to-site VPN traffic from address translation. |

**CCNP Security — ASA 9.x — Remote-Access IKEv2**

| Command | Description |
|---|---|
| **Set remote-access trustpoint:**<br>`(config)#crypto ikev2 remote-access trustpoint <trustpoint-name>` | Assigns the identity certificate for IKEv2 remote access. |
| **Create dynamic crypto map:**<br>`(config)#crypto dynamic-map <dynamic-map-name> <sequence> set ikev2 ipsec-proposal <proposal-name>` | Associates an IKEv2 proposal with dynamic remote peers. |
| **Attach dynamic map:**<br>`(config)#crypto map <map-name> <sequence> ipsec-isakmp dynamic <dynamic-map-name>` | Attaches the dynamic map to the parent crypto map. |
| **Apply parent crypto map:**<br>`(config)#crypto map <map-name> interface <interface-name>` | Applies remote-access crypto processing to the interface. |
| **Create group policy:**<br>`(config)#group-policy <group-policy-name> internal` | Creates a locally stored remote-access group policy. |
| **Enter group-policy attributes:**<br>`(config)#group-policy <group-policy-name> attributes` | Enters remote-access group-policy attribute mode. |
| **Permit IKEv2 tunnel protocol:**<br>`(config-group-policy)#vpn-tunnel-protocol ikev2` | Allows IKEv2 sessions for the group policy. |
| **Create client address pool:**<br>`(config)#ip local pool <pool-name> <start-ip>-<end-ip> mask <subnet-mask>` | Creates the IPv4 pool assigned to VPN clients. |
| **Create remote-access tunnel group:**<br>`(config)#tunnel-group <tunnel-group-name> type remote-access` | Creates an ASA remote-access connection profile. |
| **Enter general attributes:**<br>`(config)#tunnel-group <tunnel-group-name> general-attributes` | Enters tunnel-group general attribute mode. |
| **Assign client address pool:**<br>`(config-tunnel-general)#address-pool <pool-name>` | Assigns the VPN client address pool. |
| **Assign default group policy:**<br>`(config-tunnel-general)#default-group-policy <group-policy-name>` | Associates the default group policy with the profile. |
| **Set authentication server:**<br>`(config-tunnel-general)#authentication-server-group <aaa-server-group>` | Selects the AAA server group for user authentication. |
| **Enter IPsec attributes:**<br>`(config)#tunnel-group <tunnel-group-name> ipsec-attributes` | Enters IKEv2 remote-access IPsec attribute mode. |
| **Set remote certificate authentication:**<br>`(config-tunnel-ipsec)#ikev2 remote-authentication certificate` | Requires certificate authentication from remote clients. |
| **Set local certificate authentication:**<br>`(config-tunnel-ipsec)#ikev2 local-authentication certificate <trustpoint-name>` | Uses the specified certificate for local authentication. |

**CCNP Security — ASA 9.x — Clientless SSL VPN**

| Command | Description |
|---|---|
| **Enter WebVPN configuration:**<br>`(config)#webvpn` | Enters ASA WebVPN configuration mode. |
| **Enable SSL VPN interface:**<br>`(config-webvpn)#enable <interface-name>` | Enables SSL VPN termination on the selected interface. |
| **Create WebType ACL:**<br>`(config)#access-list <acl-name> webtype permit url <url>` | Permits the specified URL through clientless SSL VPN. |
| **Enter policy WebVPN attributes:**<br>`(config)#group-policy <group-policy-name> attributes`<br>&nbsp;&nbsp;○ `(config-group-policy)#webvpn` | Enters WebVPN attributes for the group policy. |
| **Apply WebType filter:**<br>`(config-group-webvpn)#filter value <acl-name>` | Applies the WebType ACL to the clientless policy. |

**CCNP Security — ASA 9.x — VPN Verification**

| Command | Description |
|---|---|
| **Show IKEv2 SAs:**<br>`#show crypto ikev2 sa` | Displays active ASA IKEv2 security associations. |
| **Show IPsec SAs:**<br>`#show crypto ipsec sa` | Displays ASA IPsec SAs and packet counters. |
| **Show site-to-site sessions:**<br>`#show vpn-sessiondb l2l` | Displays active LAN-to-LAN VPN sessions. |
| **Show detailed VPN sessions:**<br>`#show vpn-sessiondb detail` | Displays detailed VPN session statistics. |
| **Show remote-access sessions:**<br>`#show vpn-sessiondb anyconnect` | Displays active Cisco Secure Client sessions. |

**CCNP Security — Secure Firewall Threat Defense — VPN Verification Only**

| Command | Description |
|---|---|
| **Show deployed crypto configuration:**<br>`> show running-config crypto` | Displays FMC-deployed VPN crypto configuration. |
| **Show all VPN sessions:**<br>`> show vpn-sessiondb detail` | Displays detailed active VPN session information. |
| **Show site-to-site sessions:**<br>`> show vpn-sessiondb detail l2l` | Displays detailed LAN-to-LAN VPN session information. |
| **Show remote-access sessions:**<br>`> show vpn-sessiondb anyconnect` | Displays active Secure Client remote-access sessions. |
| **Show IKEv2 SAs:**<br>`> show crypto ikev2 sa` | Displays active Threat Defense IKEv2 security associations. |
| **Show IPsec SAs:**<br>`> show crypto ipsec sa` | Displays active Threat Defense IPsec security associations. |


</div>
