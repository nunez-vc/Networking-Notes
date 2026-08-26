<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# Wireless Security

> **Core idea:** Wireless security controls **who may join a WLAN, how that identity is verified, how 802.11 traffic is cryptographically protected over the air, and what access the client receives after authentication**. In enterprise WLANs, this normally means WPA2/WPA3, strong key management, 802.1X/EAP with RADIUS where practical, Protected Management Frames (PMF), and segmentation after admission.

---

## 1. What It Is

**Wireless security** is the set of authentication, encryption, key-management, management-frame protection, access-control, and monitoring mechanisms used to protect 802.11 WLANs.

Unlike a switched Ethernet access port, a WLAN uses a shared RF medium, so an attacker does not need physical cable access to observe or transmit frames; security must therefore be enforced before and after association.

```text
Wireless Security
=
Client authentication
+ Key establishment
+ Over-the-air encryption/integrity
+ Management-frame protection
+ Authorization / segmentation
+ Rogue/threat detection
```

---

## 2. How It Works

## Secure Client Join Sequence

A protected WLAN generally follows this sequence:

```text
1. Discover WLAN / AP
        ↓
2. 802.11 authentication and association
        ↓
3. Authenticate client
   - PSK / SAE
   - or 802.1X / EAP
        ↓
4. Establish cryptographic key material
        ↓
5. EAPOL 4-way handshake
        ↓
6. Install unicast/group keys
        ↓
7. Apply authorization policy
        ↓
8. Permit protected data traffic
```

Association alone does not mean the client has been authorized to use the production network.

---

# WPA Security Modes

## WPA2-Personal

WPA2-Personal uses a shared **pre-shared key (PSK)**.

```text
Client             AP/WLAN

Shared PSK         Same PSK
    \                /
     \              /
      → Key derivation
            ↓
       4-way handshake
            ↓
      Per-session keys
```

With modern WPA2, data protection normally uses:

```text
AES-CCMP
```

### Strength

```text
+ Simple
+ No RADIUS infrastructure
```

### Limitation

Every device typically knows the same credential.

If the PSK is weak or exposed:

```text
One leaked secret
→ Unauthorized WLAN access
→ Rekeying may require updating many clients
```

A captured WPA2-Personal handshake can also be used for offline password guessing, making strong random PSKs important.

---

## WPA3-Personal

WPA3-Personal replaces the traditional PSK authentication exchange with **SAE (Simultaneous Authentication of Equals)**.

```text
Password
   ↓
SAE exchange
   ↓
Authenticated key agreement
   ↓
PMK
   ↓
4-way handshake
   ↓
Session keys
```

SAE is designed to resist offline dictionary attacks based only on a captured authentication exchange.

WPA3-Personal also provides stronger protection against later password compromise than traditional WPA2-PSK.

> A weak password is still undesirable. SAE improves the authentication exchange; it does not make poor credential practices harmless.

---

## WPA2/WPA3-Enterprise

Enterprise mode uses:

```text
802.1X
+
EAP
+
RADIUS
```

instead of one shared WLAN credential.

Typical architecture:

```text
Supplicant
(Client)
    |
    | EAPOL
    v
Authenticator
(AP/WLC infrastructure)
    |
    | RADIUS
    v
Authentication Server
(ISE / RADIUS)
```

### Roles

| Role | Function |
|---|---|
| **Supplicant** | Client software requesting access |
| **Authenticator** | WLAN infrastructure controlling access |
| **Authentication server** | RADIUS/ISE system validating identity and returning policy |

The authentication server can return authorization attributes that affect the client session, such as:

```text
VLAN
ACL
Session timeout
QoS
Other supported policy attributes
```

Enterprise mode scales better because users/devices can have individual identities rather than sharing one WLAN secret.

---

# 802.1X and EAP

**802.1X** controls network access until authentication succeeds.

**EAP (Extensible Authentication Protocol)** is the framework used to carry authentication methods.

```text
802.1X
= Access-control framework

EAP
= Authentication framework

RADIUS
= Carries EAP/policy between authenticator and authentication server
```

Common EAP methods include:

```text
EAP-TLS
PEAP
EAP-FAST
```

Exact method support depends on client, WLC, and authentication-server capabilities.

---

## EAP-TLS

EAP-TLS uses X.509 certificates for both client and server authentication.

```text
Client certificate
        +
Server certificate
        ↓
Mutual authentication
```

Advantages:

```text
Strong machine/user identity
No reusable user password transmitted through the EAP method
Strong resistance to credential phishing when PKI is operated correctly
```

Operational dependency:

```text
PKI
Certificate enrollment
Renewal
Revocation
Trust-chain management
Time synchronization
```

For managed enterprise devices, EAP-TLS is commonly the strongest practical WLAN authentication model.

---

## Server-Certificate Validation

For tunneled/password-based EAP methods, clients must validate the authentication server's certificate and expected identity.

If they do not:

```text
Attacker deploys evil-twin WLAN
        ↓
Client connects
        ↓
Client accepts untrusted authentication server
        ↓
Credentials may be exposed or attacked
```

> **Do not disable RADIUS/EAP server-certificate validation merely to make onboarding easier.**

---

# Key Hierarchy and the 4-Way Handshake

After authentication produces suitable master key material, the client and WLAN infrastructure perform the **EAPOL 4-way handshake**.

Its core functions are to:

```text
Prove both sides possess the PMK
Derive/install the PTK
Distribute/install the GTK
Enable protected data traffic
```

### Key Roles

```text
PMK
= Pairwise Master Key

PTK
= Pairwise Transient Key
= Protects client-specific unicast traffic

GTK
= Group Temporal Key
= Protects group-addressed traffic
```

Conceptually:

```text
Authentication / SAE
       ↓
      PMK
       ↓
EAPOL 4-way handshake
       ↓
+-------------------+
| PTK               |
| Client-specific   |
+-------------------+

+-------------------+
| GTK               |
| Group traffic     |
+-------------------+
```

> The 4-way handshake is primarily **key establishment/confirmation**, not the original user-authentication exchange in an 802.1X design.

---

# Encryption and Integrity

## AES-CCMP

WPA2 commonly uses:

```text
AES-CCMP
```

for confidentiality and integrity.

WPA3 uses stronger mandatory security requirements and modern cryptographic suites appropriate to the selected WPA3 mode.

Legacy mechanisms should not be used:

```text
WEP
TKIP
WPA1
```

These are obsolete/deprecated and should remain disabled unless an exceptional legacy dependency is explicitly accepted.

---

# Protected Management Frames (PMF)

802.11 management frames control association and mobility.

Certain management frames historically could be forged to disrupt clients.

**PMF**, based on IEEE 802.11w mechanisms, protects selected robust management frames such as:

```text
Deauthentication
Disassociation
Certain Action frames
```

PMF helps prevent spoofed management-frame attacks.

```text
Without PMF:
Forged deauthentication
→ Client can be disconnected

With PMF:
Protected robust management frame
→ Integrity/authenticity validation
```

Important:

```text
PMF does not make RF jamming impossible.
```

WPA3 requires PMF. WPA2 deployments can support PMF where client compatibility allows it.

---

# Open WLAN and Enhanced Open

## Traditional Open WLAN

An open WLAN provides no link-layer confidentiality.

```text
No PSK
No 802.1X
No over-the-air data encryption
```

A captive portal may authenticate a user later, but:

> **Web authentication does not retroactively encrypt the 802.11 radio link.**

Application protocols such as HTTPS can still encrypt their own sessions, but the WLAN itself remains open.

---

## Enhanced Open / OWE

**OWE (Opportunistic Wireless Encryption)** provides per-client encryption on an otherwise open-access WLAN without requiring a shared password.

```text
No user identity authentication
+
Encrypted radio link
```

OWE improves privacy against passive over-the-air observation but does not authenticate the WLAN as a trusted enterprise service in the same way that 802.1X/EAP does.

---

# Guest Wireless

Guest access should normally be separated from trusted enterprise resources.

Typical flow:

```text
Guest SSID
   ↓
Open / OWE / supported secure onboarding
   ↓
Captive portal if required
   ↓
Guest authorization
   ↓
Internet / explicitly permitted services
```

Guest policy should normally prevent:

```text
Direct access to internal production networks
Unnecessary peer-to-peer client communication
Access to management infrastructure
```

WebAuth provides portal-based admission/acceptance; it is not a replacement for WLAN encryption, firewalling, or segmentation.

---

# Authorization and Segmentation

Successful authentication answers:

```text
Who are you?
```

Authorization answers:

```text
What may you access?
```

Enterprise WLAN policy can place clients into different logical access policies based on identity or device state.

Example:

```text
Managed employee
→ Corporate policy

BYOD
→ Restricted policy

Guest
→ Internet-only policy

IoT
→ Limited application-only policy
```

Possible enforcement mechanisms include:

```text
VLAN assignment
ACLs
Firewall policy
Security Group Tags / TrustSec
VRFs / segmentation
```

> Authentication without segmentation can still leave excessive east-west reachability.

---

# Fast Secure Roaming

Enterprise WLANs often need users to roam without repeating a full authentication process at every AP.

Mechanisms can include:

```text
PMK caching
Opportunistic key caching where supported
802.11r Fast BSS Transition (FT)
```

802.11r reduces the authentication/key-establishment delay during roaming.

This matters for:

```text
Voice
Real-time video
Latency-sensitive applications
```

Client compatibility must be validated before requiring fast-roaming features.

---

# Rogue APs and Evil Twins

## Rogue AP

A rogue AP is an unauthorized AP that may be connected to or operating near the enterprise network.

Risk depends on whether it provides an actual path into protected infrastructure.

---

## Evil Twin

An evil twin imitates a legitimate WLAN:

```text
Same / similar SSID
Strong nearby signal
Attacker-controlled AP
```

The objective is often to:

```text
Capture credentials
Intercept traffic
Mislead users
```

Strong enterprise authentication, correct certificate validation, PMF, endpoint configuration, and wireless threat monitoring reduce the risk.

---

# WIDS / WIPS

Wireless monitoring systems can detect conditions such as:

```text
Rogue APs
Rogue clients
Certain spoofing patterns
Wireless attacks
Interference/security events
```

### WIDS

```text
Detect + Alert
```

### WIPS

```text
Detect + Potentially Contain/Prevent
```

Containment must be used carefully because RF enforcement can affect legitimate neighboring networks and may be subject to regulatory/legal constraints.

---

# Wireless DoS

Cryptography cannot prevent all availability attacks.

Examples:

```text
RF jamming
Severe interference
Channel exhaustion
Association/authentication floods
```

Wireless availability therefore also depends on:

```text
RF design
Spectrum monitoring
Rate/control-plane protections
Redundant coverage
Operational detection
```

---

## 3. Why and When It Is Used

Wireless security is required whenever a WLAN carries traffic that should not be freely observable or accessible to any device within RF range.

Typical enterprise use cases:

```text
Corporate employee WLAN
Voice/handheld WLAN
IoT WLAN
BYOD
Guest wireless
Warehouse/industrial wireless
Healthcare wireless
Remote-site wireless
```

### Normal Recommendations

```text
Managed enterprise clients
→ WPA2/WPA3-Enterprise with 802.1X
→ Prefer certificate-based EAP where practical

Small/isolated networks or constrained devices
→ WPA3-Personal where supported
→ Strong unique operational credential strategy

Guest/public access
→ OWE where supported
→ Captive portal only when business/policy requires it
→ Strong segmentation
```

Security selection must account for the **least-capable supported client**. A WLAN that requires a feature unsupported by critical endpoints simply becomes unusable.

---

## 4. Key Configuration, Parameters, or CLI

# Catalyst 9800 WLC — IOS XE

> Exact WPA3, PMF, SAE, 802.1X, and cipher options vary by IOS XE release, AP model, radio band, and client support. Validate the exact feature matrix and command reference before production changes.

---

## Configuration Model

On Catalyst 9800:

```text
WLAN Profile
= SSID + Layer 2 security/authentication

Policy Profile
= VLAN + forwarding/access policy

Policy Tag
= Maps WLAN profile to policy profile

AP
= Receives applicable tags/profiles
```

For GUI-based configuration:

```text
Configuration
→ Tags & Profiles
→ WLANs
→ Security
→ Layer 2
```

AAA/RADIUS configuration:

```text
Configuration
→ Security
→ AAA
```

---

## Key WLAN Parameters to Validate

For a protected enterprise WLAN:

```text
WPA version
AKM / authentication method
Cipher
PMF policy
802.1X method list
RADIUS server group
Policy profile / VLAN
AAA override if used
Fast-roaming policy if used
```

For WPA3-Personal:

```text
WPA3 / SAE
Strong passphrase
PMF
Client compatibility
```

---

## Catalyst 9800 Verification

```cisco
show wlan summary
show wlan name <wlan-name>
show wireless client summary
show wireless client mac-address <client-mac> detail
show aaa servers
show radius statistics
```

Use the client-detail output to verify items such as:

```text
WLAN / policy
Authentication state
Security method
Client VLAN
AP / radio association
Session state
```

Available displayed fields vary by IOS XE release.

---

## Catalyst 9800 Client Troubleshooting

A practical sequence:

```text
1. Can the client see the SSID?
        ↓
2. Does 802.11 association succeed?
        ↓
3. Does authentication start?
        ↓
4. Does RADIUS receive the request?
        ↓
5. Does authentication succeed?
        ↓
6. Does the 4-way handshake complete?
        ↓
7. Is the correct policy/VLAN applied?
        ↓
8. Does DHCP succeed?
        ↓
9. Can the client reach the default gateway?
        ↓
10. Does security policy permit the application?
```

For difficult client failures, Catalyst 9800 **Radioactive Trace** is a high-value diagnostic tool because it follows client state transitions and authentication events from the controller's perspective.

GUI location varies somewhat by IOS XE release, but it is available through the controller troubleshooting workflow.

---

# Cisco ISE — Enterprise WLAN

> **Platform:** Cisco Identity Services Engine.

ISE commonly provides:

```text
RADIUS authentication
Authorization policy
Certificate-based identity
Endpoint profiling
Dynamic VLAN / ACL / SGT policy
Guest/BYOD workflows
Change of Authorization (CoA)
```

When troubleshooting:

```text
Check RADIUS Live Logs
        ↓
Find client authentication
        ↓
Verify authentication result
        ↓
Verify matched authorization rule/profile
        ↓
Compare returned policy with WLC session
```

A WLAN can authenticate successfully yet receive the wrong authorization policy, so both authentication and authorization results matter.

---

# AireOS

> **Platform:** Cisco AireOS WLC — legacy platform.

AireOS uses a different configuration model and CLI from Catalyst 9800.

Do not copy Catalyst 9800 IOS XE WLAN/AAA syntax into AireOS.

For existing AireOS environments, verify:

```text
WLAN Layer 2 security
AAA/RADIUS server
802.1X settings
Client security details
PMF/roaming capabilities supported by that release
```

against the specific AireOS release documentation.

---

## 5. Common Gotchas and Misconceptions

### Hiding the SSID Secures the WLAN

**Incorrect.**

The SSID can still be learned from normal 802.11 operation.

Hidden SSIDs add operational complexity without providing meaningful access control.

---

### MAC Filtering Is Strong Authentication

**Incorrect.**

Wireless MAC addresses are observable and can be spoofed.

MAC-based controls can be useful for limited identification or legacy-device handling, but they are not strong proof of identity.

---

### WPA2/WPA3 Authentication Means the User Can Reach Everything

**Incorrect.**

Authentication and authorization are separate.

```text
Authenticated
≠
Unrestricted access
```

Apply VLAN/ACL/firewall/identity-based policy after admission.

---

### WebAuth Encrypts an Open WLAN

**Incorrect.**

A captive portal controls access after association.

It does not provide 802.11 link encryption unless the WLAN separately uses a protection mechanism such as OWE or WPA.

---

### The 4-Way Handshake Authenticates the User to RADIUS

**Incorrect.**

In enterprise WLANs, EAP/RADIUS authentication produces keying material first.

The 4-way handshake then derives/confirms session keys and enables protected traffic.

---

### WPA3 Means Every Client Will Connect

**Incorrect.**

Client support varies.

Validate:

```text
WPA3
SAE
PMF
Cipher support
802.1X/EAP method
802.11r if required
```

before enforcing them on a production SSID.

---

### WPA3 Transition Mode Is Identical to WPA3-Only Security

**Incorrect.**

Transition modes improve compatibility with older WPA2 clients but preserve a legacy compatibility path.

Use transition mode only where client migration requires it.

---

### PMF Prevents RF Jamming

**Incorrect.**

PMF protects selected management frames.

It cannot prevent a transmitter from disrupting the RF medium.

---

### A Strong PSK Gives Per-User Accountability

**Incorrect.**

A shared PSK identifies possession of the shared secret, not a unique person.

802.1X/EAP provides stronger identity and per-user/device policy.

---

### Enterprise WLAN Security Is Strong Even If Clients Ignore the RADIUS Certificate

**Incorrect or Unsafe.**

Failure to validate the authentication server can expose clients to evil-twin credential attacks.

Certificate trust and expected server identity are part of the security design.

---

### Guest Wireless Is Safe Because It Cannot Reach the Internet Edge Directly

**Incorrect.**

Guest networks still require:

```text
Internal segmentation
Peer isolation where required
DNS/DHCP security
Firewall policy
Logging
Abuse controls
```

---

### WEP or TKIP Is Acceptable for Legacy Compatibility

**Incorrect or Unsafe** unless a formally accepted legacy exception requires it.

Preferred approach:

```text
Replace / isolate the legacy client
```

rather than weakening a production WLAN for every user.

---

## 6. Trade-Offs

### Best Practice

- Prefer **WPA3** where the complete client population supports it.
- Use **WPA2/WPA3-Enterprise with 802.1X** for managed enterprise users and devices.
- Prefer **EAP-TLS** where certificate lifecycle operations are mature enough to support it.
- Require correct authentication-server certificate validation.
- Use PMF according to WPA version and client capability; WPA3 requires it.
- Segment guest, BYOD, IoT, and trusted corporate devices according to risk.
- Keep WEP, TKIP, and WPA1 disabled.
- Validate security and roaming features against actual endpoint capabilities before enforcement.
- Monitor for rogue/evil-twin behavior and authentication anomalies.

---

### Context-Dependent Trade-Off — Personal vs Enterprise

**Personal / SAE**

```text
+ Simple
+ No RADIUS dependency
+ Good for small or constrained environments
- Shared credential lifecycle
- Limited individual accountability
```

**802.1X Enterprise**

```text
+ Per-user/device identity
+ Centralized authorization
+ Better credential revocation
+ Dynamic policy
- Requires RADIUS/identity infrastructure
- More supplicant/certificate complexity
```

For managed enterprise fleets, 802.1X is normally the better operational/security model.

---

### Context-Dependent Trade-Off — WPA3-Only vs Transition Mode

**WPA3-Only**

```text
+ Stronger minimum security baseline
+ No WPA2 compatibility path
- Excludes unsupported clients
```

**Transition Mode**

```text
+ Easier migration
+ Supports older clients
- Security baseline remains constrained by legacy compatibility
```

Use transition mode as a migration mechanism, not as a permanent default without a requirement.

---

### Context-Dependent Trade-Off — EAP-TLS vs Password-Based EAP

**EAP-TLS**

```text
+ Strong mutual certificate authentication
+ Strong device/user identity
+ No reusable user password dependency
- PKI lifecycle complexity
```

**Password-based tunneled EAP**

```text
+ Easier where passwords already exist
+ Broad legacy compatibility
- Greater credential-phishing exposure if server validation is weak
- Password lifecycle remains a dependency
```

---

### Context-Dependent Trade-Off — Open Guest vs OWE

**Traditional Open**

```text
+ Maximum compatibility
- No WLAN-layer confidentiality
```

**OWE / Enhanced Open**

```text
+ Per-client radio encryption
+ No shared guest password
- Requires client support
- Does not provide authenticated user/device identity
```

---

### Incorrect or Unsafe

- Deploying WEP, TKIP, or WPA1 on a production WLAN.
- Disabling certificate validation to make 802.1X onboarding easier.
- Using one shared corporate PSK where individual identity and revocation are required.
- Treating hidden SSIDs or MAC filtering as strong security.
- Placing guests/BYOD/IoT into trusted production segments merely because WLAN authentication succeeded.
- Enabling WPA3/PMF/802.11r globally without validating critical client compatibility.
- Assuming wireless encryption protects traffic after it leaves the WLAN; end-to-end application security and network segmentation are still required.

---

## Quick Reference

```text
Wireless Security
= Authentication
+ Key management
+ Encryption/integrity
+ PMF
+ Authorization
+ Monitoring

WPA2-Personal
= PSK
= AES-CCMP
= Shared credential

WPA3-Personal
= SAE
= Stronger password-authenticated key exchange
= PMF required

Enterprise WLAN
= 802.1X + EAP + RADIUS

Supplicant
= Wireless client

Authenticator
= AP/WLC infrastructure

Authentication Server
= RADIUS / ISE

EAPOL
= Client ↔ authenticator

RADIUS
= Authenticator ↔ authentication server

EAP-TLS
= Client + server certificates
= Strong enterprise authentication

PMK
= Master key material

PTK
= Client-specific unicast key

GTK
= Group traffic key

4-Way Handshake
= Derives/confirms/installs session keys

PMF / 802.11w
= Protects selected robust management frames

Open WLAN
= No WLAN-layer encryption

OWE
= Encrypted open access
= No user identity authentication

WebAuth
= Portal-based access control
≠ WLAN encryption

802.11r
= Fast BSS Transition
= Reduces secure-roaming delay

WIDS
= Detect / alert

WIPS
= Detect + containment/prevention capability

Legacy:
WEP / TKIP / WPA1
= Do not use

Core Rule
= Authenticate strongly, establish unique session keys,
  protect management traffic, authorize least privilege,
  and verify the complete client-to-application path.
```

</div>
