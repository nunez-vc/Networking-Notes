<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# VLAN Trunking Protocol (VTP)

> **Core idea:** VTP is a Cisco Layer 2 control protocol that distributes VLAN database changes between switches in the same VTP domain across trunk links. It can simplify VLAN provisioning, but incorrect VTP state—especially with older VTP versions—can propagate destructive VLAN changes across many switches.

---

## 1. What It Is

**VTP (VLAN Trunking Protocol)** is a Cisco-proprietary protocol used to synchronize VLAN information between switches in the same VTP domain.

VTP controls **VLAN database distribution**; it does **not** create trunks, route between VLANs, or determine the Spanning Tree topology.

```text
VTP
= VLAN database synchronization

Not
= Trunk negotiation
= Inter-VLAN routing
= STP
```

---

## 2. How It Works

## VTP Domain

Switches that participate in the same VTP management domain use a common:

```text
VTP domain name
VTP version
VTP authentication/password when configured
```

VTP advertisements are carried across Layer 2 trunk links.

```text
        VTP Domain: CAMPUS

     +----------------+
     | VTP Server     |
     +----------------+
          | trunk
          v
     +----------------+
     | VTP Client     |
     +----------------+
          | trunk
          v
     +----------------+
     | VTP Client     |
     +----------------+
```

A switch outside the domain does not synchronize that VLAN database.

---

## VTP Operating Modes

### Server

A VTP server participates in VLAN synchronization.

For VTP versions 1 and 2, VLAN changes can be made on VTP servers and propagated through the domain.

For VTP version 3, VLAN changes that are to be propagated must be made by the **primary server**.

```text
VLAN created / changed / deleted
        ↓
VTP advertisement
        ↓
Clients update VLAN database
```

---

### Client

A VTP client:

```text
Receives VTP advertisements
Updates its VLAN database
Forwards VTP advertisements
```

VLANs learned through VTP should be managed from the VTP server rather than independently on the client.

---

### Transparent

A transparent switch does **not synchronize its local VLAN database** from VTP.

```text
Received VTP update
        ↓
Forward through trunk as applicable
        ↓
Local VLAN database unchanged
```

VLANs are configured locally on the transparent switch.

This mode is useful when the switch should pass VTP control traffic without allowing VTP to manage its own VLAN database.

---

### Off

A switch in VTP off mode:

```text
Does not participate in VTP
Does not synchronize VLANs
Does not forward VTP advertisements
```

VLAN configuration remains local.

> **Transparent and off are not identical:** transparent can forward received VTP advertisements; off does not participate in VTP forwarding.

---

## VTP Advertisements

VTP uses several advertisement types.

### Summary Advertisement

Contains domain-level information such as:

```text
VTP version
Domain name
Configuration revision
Timestamp
```

It is sent periodically and when VLAN changes occur.

---

### Subset Advertisement

Carries the VLAN database information required to apply actual VLAN changes.

```text
VLAN added
VLAN removed
VLAN modified
        ↓
Subset advertisement
        ↓
Receiving VTP switches update database
```

---

### Advertisement Request

A switch can request detailed VLAN information when it determines that another switch has a newer VLAN database.

---

## Configuration Revision Number

The **configuration revision number** identifies the relative version of the VTP VLAN database.

Conceptually:

```text
Revision 10
   vs
Revision 15
        ↓
Revision 15 is newer
```

When a VLAN is added, modified, or deleted on an authoritative VTP server, the revision increases.

This creates one of the most important VTP operational risks.

### Classic VTP Revision Risk

In VTP versions 1 and 2, connecting a switch that has:

```text
Correct VTP domain
Compatible VTP settings
Higher revision number
Unexpected VLAN database
```

can cause its database to be propagated through the VTP domain.

If that database is missing production VLANs, VLANs can be deleted across multiple switches.

```text
Old switch
Revision 50
Missing VLAN 20
        ↓
Connected to production VTP domain
        ↓
Higher revision accepted
        ↓
VLAN 20 removed across domain
```

> **Before introducing a switch into an existing VTP v1/v2 domain, verify and reset its VTP revision state according to the platform procedure.**

Changing the VTP domain is one method that resets the local revision to `0` on IOS/IOS XE.

---

## VTP Version 3 and the Primary Server

VTPv3 adds a **primary server** role.

```text
VTPv3 Server
      ↓
vtp primary
      ↓
Primary Server
      ↓
Authorized VLAN database changes
```

Only the VTPv3 primary server can originate VLAN database changes for the domain.

This reduces the risk that an accidentally connected server with a newer revision will overwrite the production VLAN database.

The primary-server role is established with an EXEC-mode command and is not simply equivalent to configuring:

```cisco
vtp mode server
```

---

## VTP Versions

| Version | VLAN Propagation | Key Operational Point |
|---|---|---|
| **VTPv1** | Normal-range VLANs | Legacy behavior and revision-number risk |
| **VTPv2** | Normal-range VLANs | Similar VLAN synchronization model to v1 |
| **VTPv3** | VLANs 1–4094 | Adds primary-server model and extended VLAN support |

For VLAN propagation:

```text
VTPv1 / VTPv2
= VLANs 1-1005

VTPv3
= VLANs 1-4094
```

> If VTP is intentionally deployed on current Catalyst infrastructure, VTPv3 is generally the safer operational model, subject to platform and software support.

---

## VTP Pruning

VTP pruning can reduce unnecessary flooded VLAN traffic on trunks where that VLAN is not required downstream.

Conceptually:

```text
VLAN 30 exists across VTP domain
          |
          +---- SW2 has VLAN 30 users
          |
          +---- SW3 has no need for VLAN 30
                       ↓
              VTP pruning can prevent
              VLAN 30 flooded traffic
              from using that trunk
```

VTP pruning affects whether a VLAN is forwarded on a trunk; it does **not** delete the VLAN from the VLAN database.

Verify trunk forwarding state with:

```cisco
show interfaces trunk
```

The output distinguishes VLANs that are:

```text
Allowed
Active
STP forwarding
Not pruned
```

---

## VTP Does Not Create the Trunk

VTP advertisements normally travel across trunks, but VTP itself does not establish the trunk.

```text
VTP
= VLAN database propagation

DTP
= Can negotiate trunk formation

802.1Q
= VLAN tagging on the trunk
```

These are separate functions.

A VLAN learned through VTP still cannot cross a trunk unless the VLAN is permitted and forwarding on that trunk.

---

## 3. Why and When It Is Used

VTP solves the operational problem of keeping VLAN databases synchronized across many Cisco switches.

Without VTP:

```text
Create VLAN 30 on SW1
Create VLAN 30 on SW2
Create VLAN 30 on SW3
Create VLAN 30 on SW4
...
```

With VTP:

```text
Create VLAN 30 on authoritative VTP server
              ↓
VTP propagates VLAN 30
              ↓
Clients learn VLAN 30
```

VTP is appropriate when:

- A Cisco Catalyst environment intentionally centralizes VLAN database management.
- Many switches must maintain the same VLAN definitions.
- The organization has controlled VTP operational procedures.
- VTPv3 and its primary-server controls are supported and deliberately used.

VTP is unnecessary or unsuitable when:

- VLANs are already managed reliably through automation or configuration management.
- The network prefers explicit per-switch VLAN control.
- The environment is multivendor.
- The operational risk of automatic VLAN propagation outweighs the administrative benefit.

Many production environments intentionally use:

```text
VTP transparent
or
VTP off
```

and provision VLANs through automation or controlled configuration instead.

---

## 4. Key Configuration, Parameters, or CLI

> **Platform:** Cisco Catalyst IOS / IOS XE.  
> Exact defaults and supported VTP versions vary by switch family and software release. Verify with `show vtp status` before making changes.

---

## VTPv3 Server

```cisco
vtp version 3
vtp domain CAMPUS
vtp mode server
vtp password <password>
```

Then from privileged EXEC mode:

```cisco
vtp primary
```

Only the VTPv3 primary server should be used to make VLAN database changes that need to propagate.

---

## VTP Client

```cisco
vtp version 3
vtp domain CAMPUS
vtp mode client
vtp password <password>
```

---

## Transparent Mode

```cisco
vtp version 3
vtp domain CAMPUS
vtp mode transparent
```

The local VLAN database is managed locally.

---

## Off Mode

On platforms that support it:

```cisco
vtp mode off
```

This disables VTP participation and forwarding.

---

## Verification

Primary command:

```cisco
show vtp status
```

Verify:

```text
VTP version
Domain name
Operating mode
Primary server
Configuration revision
VTP pruning state
Number of VLANs
```

Verify VLAN database:

```cisco
show vlan brief
```

Verify trunks and pruning state:

```cisco
show interfaces trunk
```

A practical validation sequence is:

```text
Correct VTP version?
        ↓
Correct domain?
        ↓
Correct mode?
        ↓
Authentication consistent?
        ↓
Expected revision?
        ↓
Correct primary server for VTPv3?
        ↓
VLAN exists locally?
        ↓
VLAN allowed/forwarding on trunks?
```

---

## 5. Common Gotchas and Misconceptions

### VTP Creates Trunks

**Incorrect.**

VTP distributes VLAN information.

```text
VTP ≠ DTP
VTP ≠ 802.1Q
```

A trunk must already exist or be negotiated separately.

---

### VTP Propagates Interface-to-VLAN Assignments

**Incorrect.**

VTP distributes VLAN database information such as VLAN IDs and associated VLAN properties.

It does not automatically configure:

```text
switchport access vlan 30
```

on endpoint-facing switchports.

Port assignments remain local configuration.

---

### A Higher Revision Number Means the VLAN Database Is Correct

**Incorrect.**

A higher revision means **newer according to VTP**, not necessarily operationally correct.

This is why an old or lab switch with an unexpected VLAN database can be dangerous in VTPv1/v2.

---

### Transparent Mode Is the Same as Off Mode

**Incorrect.**

```text
Transparent
= Local VLAN database not synchronized
= Can forward received VTP advertisements

Off
= Does not participate
= Does not forward VTP advertisements
```

---

### VTPv3 Server Automatically Means Primary Server

**Incorrect.**

In VTPv3:

```text
vtp mode server
```

does not by itself make the switch the primary server.

The primary role is established separately:

```cisco
vtp primary
```

---

### VTP Pruning Deletes VLANs

**Incorrect.**

Pruning limits unnecessary VLAN traffic on specific trunks.

It does not remove the VLAN from the VTP database.

---

### A VLAN Learned Through VTP Must Be Forwarding on Every Trunk

**Incorrect.**

A VLAN can exist locally but still fail to cross a trunk because it is:

```text
Not allowed on the trunk
Inactive
Blocked by STP
VTP-pruned
```

Verify:

```cisco
show interfaces trunk
```

---

## 6. Trade-Offs

### Best Practice

- Prefer **VTPv3** if VTP is intentionally used and all required switches support it.
- Use a controlled primary-server workflow for VLAN changes.
- Verify VTP domain, version, mode, authentication, and revision before connecting a switch to an existing VTP domain.
- Back up and document the production VLAN database before major VTP changes.
- Use explicit operational change control because VLAN deletion can have a large Layer 2 blast radius.

---

### Context-Dependent Trade-Off — VTP vs Local/Automated VLAN Management

**VTP**

```text
+ Central VLAN database propagation
+ Reduces repeated manual VLAN creation
+ Rapid synchronization across many switches
- Incorrect changes can propagate widely
- Cisco-specific
- Requires disciplined domain/revision management
```

**Local / Automated Provisioning**

```text
+ Explicit per-device control
+ Works well with Infrastructure as Code / automation
+ Avoids VTP revision-domain risk
- Requires external tooling or repeated configuration
```

For modern automated networks, VTP may provide little additional value. In a controlled Cisco campus that intentionally uses VTPv3, it can still simplify VLAN database synchronization.

---

### Context-Dependent Trade-Off — Transparent vs Off

**Transparent**

```text
+ Local VLAN control
+ Can pass VTP advertisements to downstream switches
- VTP traffic still traverses the switch
```

**Off**

```text
+ Completely removes switch from VTP propagation
+ Lowest accidental VTP interaction
- Cannot relay VTP advertisements through that switch
```

Choose based on whether downstream VTP continuity is required.

---

### Incorrect or Unsafe

- Connecting an unknown switch to a production VTPv1/v2 domain without checking/resetting its revision state.
- Changing VTP domain, version, or mode in production without understanding the VLAN database impact.
- Treating VTP authentication as a substitute for disciplined change control.
- Assuming a VTP-learned VLAN is automatically allowed and forwarding on every trunk.
- Experimenting with VTP on equipment that still has physical connectivity to a production Layer 2 domain.

---

## Quick Reference

```text
VTP
= VLAN Trunking Protocol
= Cisco proprietary

Purpose
= Synchronize VLAN database information

Transport
= Layer 2 across trunks

Modes
= Server
= Client
= Transparent
= Off

Server
= Participates in VLAN synchronization

Client
= Learns VLAN database through VTP

Transparent
= Local VLAN configuration
= Does not synchronize local database
= Forwards received VTP advertisements

Off
= No VTP participation
= No VTP forwarding

Revision Number
= Version of VLAN database

Classic Risk
= Higher unexpected revision can overwrite VTPv1/v2 domain

VTPv1 / VTPv2
= VLANs 1-1005

VTPv3
= VLANs 1-4094
= Primary server model

VTPv3 Primary
= vtp primary

Verification
= show vtp status
= show vlan brief
= show interfaces trunk

VTP Pruning
= Reduces unnecessary flooded VLAN traffic on trunks
= Does not delete VLANs

VTP ≠ DTP
VTP ≠ 802.1Q
VTP ≠ STP
```

## CCNA Configuration

**Current live CCNA 200-301 v1.1 — VTP configuration is outside exam scope.**

## CCNP Configuration

**CCNP Enterprise — IOS-XE 17.x — VTP Base Configuration (supporting switching reference; not explicit ENCOR 350-401 v1.2 blueprint)**

| Command | Description |
|---|---|
| **Set VTP domain:**<br>`(config)#vtp domain <domain-name>` | Configures the VTP administrative domain name. |
| **Select VTP version:**<br>`(config)#vtp version <1|2|3>` | Selects the VTP protocol version. |
| **Set server mode:**<br>`(config)#vtp mode server` | Sets the switch to VTP server mode. |
| **Set client mode:**<br>`(config)#vtp mode client` | Sets the switch to VTP client mode. |
| **Set transparent mode:**<br>`(config)#vtp mode transparent` | Sets the switch to VTP transparent mode. |
| **Disable VTP:**<br>`(config)#vtp mode off` | Disables VTP participation and advertisement forwarding. |
| **Set VTP password:**<br>`(config)#vtp password <password>` | Configures the VTP domain password. |
| **Remove VTP password:**<br>`(config)#no vtp password` | Removes the configured VTP domain password. |

**CCNP Enterprise — IOS-XE 17.x — VTPv3 Database Mode**

| Command | Description |
|---|---|
| **Set VLAN server mode:**<br>`(config)#vtp mode server vlan` | Sets VTPv3 server mode for the VLAN database. |
| **Set VLAN client mode:**<br>`(config)#vtp mode client vlan` | Sets VTPv3 client mode for the VLAN database. |
| **Set VLAN transparent mode:**<br>`(config)#vtp mode transparent vlan` | Sets VTPv3 transparent mode for the VLAN database. |
| **Disable VLAN VTP instance:**<br>`(config)#vtp mode off vlan` | Disables VTPv3 for the VLAN database. |
| **Set MST server mode:**<br>`(config)#vtp mode server mst` | Sets VTPv3 server mode for the MST database. |
| **Set MST client mode:**<br>`(config)#vtp mode client mst` | Sets VTPv3 client mode for the MST database. |
| **Set MST transparent mode:**<br>`(config)#vtp mode transparent mst` | Sets VTPv3 transparent mode for the MST database. |
| **Disable MST VTP instance:**<br>`(config)#vtp mode off mst` | Disables VTPv3 for the MST database. |

**CCNP Enterprise — IOS-XE 17.x — VTPv3 Password Protection**

| Command | Description |
|---|---|
| **Configure hidden password:**<br>`(config)#vtp password <password> hidden` | Stores a derived secret instead of cleartext password. |
| **Configure secret directly:**<br>`(config)#vtp password <32-hex-secret> secret` | Configures the VTPv3 secret in hexadecimal form. |
| **Verify VTP password:**<br>`#show vtp password` | Displays configured VTP password status and storage form. |

**CCNP Enterprise — IOS-XE 17.x — VTPv3 Primary Server**

| Command | Description |
|---|---|
| **Become VLAN primary:**<br>`#vtp primary vlan` | Promotes the switch to VLAN VTPv3 primary server. |
| **Force VLAN takeover:**<br>`#vtp primary vlan force` | Forces VLAN primary takeover despite conflicting servers. |
| **Become MST primary:**<br>`#vtp primary mst` | Promotes the switch to MST VTPv3 primary server. |
| **Force MST takeover:**<br>`#vtp primary mst force` | Forces MST primary takeover despite conflicting servers. |

**CCNP Enterprise — IOS-XE 17.x — VTP VLAN Propagation**

| Command | Description |
|---|---|
| **Create VLAN:**<br>`(config)#vlan <vlan-id>` | Creates a VLAN on the writable VTP server database. |
| **Name VLAN:**<br>`(config-vlan)#name <vlan-name>` | Assigns a name to the selected VLAN. |
| **Delete VLAN:**<br>`(config)#no vlan <vlan-id>` | Deletes the specified VLAN from the writable database. |
| **Verify propagated VLANs:**<br>`#show vlan brief` | Displays locally present VLANs and access-port assignments. |

**CCNP Enterprise — IOS-XE 17.x — VTP Pruning**

| Command | Description |
|---|---|
| **Enable VTP pruning:**<br>`(config)#vtp pruning` | Enables VTP pruning for the administrative domain. |
| **Disable VTP pruning:**<br>`(config)#no vtp pruning` | Disables VTP pruning. |
| **Add pruning-eligible VLANs:**<br>`(config)#interface <trunk-interface>`<br>&nbsp;&nbsp;○ `(config-if)#switchport trunk pruning vlan add <vlan-list>` | Adds VLANs to the trunk pruning-eligible list. |
| **Remove pruning-eligible VLANs:**<br>`(config)#interface <trunk-interface>`<br>&nbsp;&nbsp;○ `(config-if)#switchport trunk pruning vlan remove <vlan-list>` | Removes VLANs from the trunk pruning-eligible list. |
| **Exclude pruning VLANs:**<br>`(config)#interface <trunk-interface>`<br>&nbsp;&nbsp;○ `(config-if)#switchport trunk pruning vlan except <vlan-list>` | Makes all supported VLANs except listed VLANs pruning-eligible. |
| **Disable trunk pruning eligibility:**<br>`(config)#interface <trunk-interface>`<br>&nbsp;&nbsp;○ `(config-if)#switchport trunk pruning vlan none` | Makes no VLANs pruning-eligible on the trunk. |
| **Verify pruning status:**<br>`#show vtp status` | Displays global VTP pruning mode. |
| **Verify pruning eligibility:**<br>`#show interfaces <trunk-interface> switchport` | Displays pruning-enabled VLANs for the trunk interface. |

**CCNP Enterprise — IOS-XE 17.x — VTPv3 Per-Port Control**

| Command | Description |
|---|---|
| **Enable VTP on trunk:**<br>`(config)#interface <trunk-interface>`<br>&nbsp;&nbsp;○ `(config-if)#vtp` | Enables VTPv3 processing on the trunk interface. |
| **Disable VTP on trunk:**<br>`(config)#interface <trunk-interface>`<br>&nbsp;&nbsp;○ `(config-if)#no vtp` | Disables VTPv3 processing on the trunk interface. |
| **Verify per-port VTP:**<br>`#show vtp interface [<interface-id>]` | Displays VTP state for all or one interface. |
| **Verify interface configuration:**<br>`#show running-config interface <interface-id>` | Displays configured per-port VTP commands. |

**CCNP Enterprise — IOS-XE 17.x — VTP Verification**

| Command | Description |
|---|---|
| **Show VTP status:**<br>`#show vtp status` | Displays VTP version, domain, mode, revision, and pruning. |
| **Show VTP counters:**<br>`#show vtp counters` | Displays VTP advertisement, error, and pruning counters. |
| **Show VTPv3 devices:**<br>`#show vtp devices` | Displays discovered VTPv3 devices in the domain. |
| **Show primary conflicts:**<br>`#show vtp devices conflicts` | Displays VTPv3 devices with conflicting primary servers. |
| **Show VTP interfaces:**<br>`#show vtp interface` | Displays VTP status across participating interfaces. |
| **Show VTP password:**<br>`#show vtp password` | Displays configured VTP password information. |
| **Show VLAN database:**<br>`#show vlan` | Displays VLANs currently present on the switch. |


</div>
