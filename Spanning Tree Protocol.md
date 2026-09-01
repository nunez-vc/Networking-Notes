# Spanning Tree Protocol

* It is a L2 control protocol that prevent switching loops.

> **Note:**
>
> * L2 frames do not have a TTL field and can circulate endlessly in a loop (e.g. Broadcast Storm and MAC address Flaps).

---

## Root Bridge

* Determined by the ethernet switch that has the lowest Bridge ID.

---

## Bridge ID (BID)

* Default Bridge Priority is **32768.**

<p align="center">
  <img width="400" alt="Local Account, Named ACL, and Security" src="Images/STP/Bridge ID.png" />
</p>

---

## Root Port

* The one (and only one) port on a non-root bridge that's closest to the Root Bridge, in terms of cost.
* Tie Breakers:

  1. Lowest Cost Bridge ID.
  2. Lowest Port ID connected to the Upstream Switch.

---

## Short Path Cost Method

<p align="center">
  <img width="400" alt="Local Account, Named ACL, and Security" src="Images/STP/Short Path Cost Method.png" />
</p>

---

## Long Path Cost Method

**Formula:** 20 Tbps / Port Speed

<p align="center">
  <img width="400" alt="Local Account, Named ACL, and Security" src="Images/STP/Long Path Cost Method.png" />
</p>

---

## Designated Port

* The one (and only one) port on each segment that is closest to the Root Bridge in terms of cost.
* These are all downstream.
* Tie Breaker: Switch Bridge ID.

---

## Blocking (Non-designated) Port

* A port that is administratively enabled, but is not a root port nor a designated port.

> **Note:**
>
> * Hello timers are **2** seconds by default.

---

## Bridge Protocol Data Unit (BPDU)

* It is a network packet, transmitted to the multicast MAC address (**01:80:C2:00:00:00**) to switches running STP to dynamically discover neighboring switches, elect a root bridge, identify network path hierarchies, and notify the system of any topology.

> **Note:**
>
> * Blocking State (**20** sec).
> * Listening State (**15** sec).
> * Learning State (**15** sec).
> * Total of **50** sec for transitioning from blocking state into forwarding state.
> * Total of **30** sec for transitioning from unused port into forwarding state.

---

## Common Spanning Tree (CST)

* IEEE 802.1D
* Same spanning tree used by all VLANs.

---

## Per VLAN Spanning Tree (PVST/PVST+)

* Each VLAN runs its STP.
* The "+" indicates the switches are connected via 802.1Q trunk encapsulation.

---

## Multiple Spanning Tree Protocol (MSTP)

* IEEE 802.1s
* Instead of configuring STP per VLAN, in MSTP you group VLANs in instances.

<p align="center">
  <img width="400" alt="Local Account, Named ACL, and Security" src="Images/STP/Long Path Cost Method.png" />
</p>

### Multiple Spanning Tree Configuration:

```
spanning-tree mst configuration
instance [instance  #] vlan [VLAN ID]
exit
spanning-tree mode mst
```

---

## Rapid Per VLAN Spanning Tree Protocol (RPVST+)

* IEEE 802.1W
* completely eliminates Listening State.
* Converge within a few seconds (typically 1 to 2 sec, and 10 sec at most) from blocking state to forwarding state.
* RSTP+ states: Discarding, Learning, and Forwarding.

<p align="center">
  <img width="550" alt="Local Account, Named ACL, and Security" src="Images/STP/Summary Reference Table.png" />
</p>

---

## PortFast

* Bypass listening and learning state.
* Can be enabled globally or on a port-by-port basis (for non-trunking ports).
* To enable portfast globally: spanning-tree portfast default
* To enable portfast port-by-port basis: spanning-tree portfast

---

## UplinkFast

* Globally enabled on a switch.
* It enables rapid convergence during a root port link failure by immediately transitioning a blocked alternate port into the forwarding state to serve as the new root port, completely bypassing the traditional 30-second Spanning Tree listening and learning delays.
* To enable UplinkFast globally: spanning-tree uplinkfast

---

## BackboneFast

* Globally enabled on a switch.
* Reacts to an indirect link failure.
* To enable BackboneFast globally: spanning-tree backbonefast

---

## BPDU Guard

* It is a security feature that protects PortFast-enabled access ports by automatically placing the interface into an err-disabled state to prevent loops the moment any Spanning Tree BPDU is received on the port.
* Should be enabled on ports with PortFast enabled.
* Can be enabled globally or on a port-by-port basis (for ports with PortFast enabled).

---

## BPDU Filter

* This feature either conditionally monitors PortFast ports globally, disabling PortFast and reverting to normal STP if a BPDU is received, or absolutely disables STP on a specific interface when configured locally by silently discarding all incoming and outgoing BPDUs.
* Prevents a port from sending BPDUs.
* Should be only used when necessary.
* Most dangerous when enabled at the port level.

---

## Root Guard

* This prevents unauthorized or rogue downstream switches from becoming the root bridge by dynamically putting the port into a non-forwarding root-inconsistent (broken) state if a superior BPDU is received, recovering automatically once those superior BPDUs cease.
* Configured on ports off of which the Root Bridge is unexpected.

---

## Loop Guard

* This prevents Layer 2 forwarding loops caused by unidirectional link failures.
* This causes a non-designated port to enter Loop Inconsistent (broken) state if it stops receiving BPDUs.

## CCNA Configuration

**CCNA 200-301 v1.1 — IOS-XE Rapid PVST+**

| Command | Description |
|---|---|
| **Enable Rapid PVST+:**<br>`(config)#spanning-tree mode rapid-pvst` | Enables Rapid PVST+ as the global spanning-tree mode. |
| **Set primary root:**<br>`(config)#spanning-tree vlan <vlan-id> root primary` | Tunes bridge priority to prefer this switch as root. |
| **Set secondary root:**<br>`(config)#spanning-tree vlan <vlan-id> root secondary` | Tunes bridge priority to prefer this switch as backup root. |
| **Set bridge priority:**<br>`(config)#spanning-tree vlan <vlan-id> priority <priority>` | Sets bridge priority for the specified VLAN. |
| **Set interface cost:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#spanning-tree vlan <vlan-id> cost <cost>` | Sets STP path cost for one VLAN. |
| **Set interface cost globally:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#spanning-tree cost <cost>` | Sets STP path cost for all applicable VLANs. |
| **Set port priority:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#spanning-tree vlan <vlan-id> port-priority <priority>` | Sets STP port priority for one VLAN. |

**CCNA 200-301 v1.1 — IOS-XE PortFast**

| Command | Description |
|---|---|
| **Enable access-port PortFast:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#spanning-tree portfast` | Enables PortFast when the interface operates as access. |
| **Enable trunk PortFast:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#spanning-tree portfast trunk` | Enables PortFast when the interface operates as trunk. |
| **Disable interface PortFast:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#spanning-tree portfast disable` | Disables PortFast on the selected interface. |

**CCNA 200-301 v1.1 — IOS-XE BPDU Guard**

| Command | Description |
|---|---|
| **Enable BPDU Guard:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#spanning-tree bpduguard enable` | Enables BPDU Guard unconditionally on the interface. |
| **Disable BPDU Guard:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#spanning-tree bpduguard disable` | Disables BPDU Guard unconditionally on the interface. |

**CCNA 200-301 v1.1 — IOS-XE Root Guard**

| Command | Description |
|---|---|
| **Enable Root Guard:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#spanning-tree guard root` | Enables Root Guard on the selected interface. |
| **Disable Root Guard:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no spanning-tree guard root` | Disables Root Guard on the selected interface. |

**CCNA 200-301 v1.1 — IOS-XE Loop Guard**

| Command | Description |
|---|---|
| **Enable Loop Guard:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#spanning-tree guard loop` | Enables Loop Guard on the selected interface. |
| **Disable Loop Guard:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no spanning-tree guard loop` | Disables Loop Guard on the selected interface. |

**CCNA 200-301 v1.1 — IOS-XE BPDU Filter**

| Command | Description |
|---|---|
| **Enable interface BPDU Filter:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#spanning-tree bpdufilter enable` | Enables unconditional BPDU filtering on the interface. |
| **Disable interface BPDU Filter:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#spanning-tree bpdufilter disable` | Disables interface BPDU filtering. |

**CCNA 200-301 v1.1 — IOS-XE Verification**

| Command | Description |
|---|---|
| **Show spanning-tree state:**<br>`#show spanning-tree` | Displays STP root, bridge, port roles, and states. |
| **Show VLAN spanning tree:**<br>`#show spanning-tree vlan <vlan-id>` | Displays STP information for the specified VLAN. |
| **Show VLAN interface state:**<br>`#show spanning-tree vlan <vlan-id> interface <interface-id>` | Displays STP state for one VLAN interface. |
| **Show VLAN interface details:**<br>`#show spanning-tree vlan <vlan-id> interface <interface-id> detail` | Displays detailed interface STP parameters and BPDU counters. |
| **Show interface spanning tree:**<br>`#show spanning-tree interface <interface-id>` | Displays STP state for the specified interface. |
| **Show interface details:**<br>`#show spanning-tree interface <interface-id> detail` | Displays PortFast, guard, filter, and BPDU details. |

## CCNP Configuration

**CCNP Enterprise — ENCOR 350-401 v1.2 — IOS-XE STP Path-Cost Method**

| Command | Description |
|---|---|
| **Use long path costs:**<br>`(config)#spanning-tree pathcost method long` | Selects 32-bit long STP path-cost values. |
| **Use short path costs:**<br>`(config)#spanning-tree pathcost method short` | Selects legacy 16-bit short STP path-cost values. |
| **Verify path-cost method:**<br>`#show spanning-tree summary` | Displays global STP mode and configured path-cost method. |

**CCNP Enterprise — ENCOR 350-401 v1.2 — IOS-XE STP Timers**

| Command | Description |
|---|---|
| **Set hello timer:**<br>`(config)#spanning-tree vlan <vlan-id> hello-time <seconds>` | Sets the STP hello interval for the VLAN. |
| **Set maximum age:**<br>`(config)#spanning-tree vlan <vlan-id> max-age <seconds>` | Sets the STP maximum-age timer for the VLAN. |
| **Set forward delay:**<br>`(config)#spanning-tree vlan <vlan-id> forward-time <seconds>` | Sets the STP forwarding-delay timer for the VLAN. |

**CCNP Enterprise — ENCOR 350-401 v1.2 — IOS-XE Advanced Root Placement**

| Command | Description |
|---|---|
| **Set primary root with diameter:**<br>`(config)#spanning-tree vlan <vlan-id> root primary diameter <diameter>` | Tunes root priority using the specified network diameter. |
| **Set secondary root with diameter:**<br>`(config)#spanning-tree vlan <vlan-id> root secondary diameter <diameter>` | Tunes backup-root priority using the specified network diameter. |
| **Show root information:**<br>`#show spanning-tree root` | Displays root bridge, root port, and root-path cost. |
| **Show topology-change details:**<br>`#show spanning-tree vlan <vlan-id> detail` | Displays topology-change timing and originating interface information. |

**CCNP Enterprise — ENCOR 350-401 v1.2 — IOS-XE MST Region Configuration**

| Command | Description |
|---|---|
| **Enable MST mode:**<br>`(config)#spanning-tree mode mst` | Enables Multiple Spanning Tree globally. |
| **Enter MST configuration:**<br>`(config)#spanning-tree mst configuration` | Enters MST region configuration mode. |
| **Set region name:**<br>`(config-mst)#name <region-name>` | Sets the case-sensitive MST region name. |
| **Set revision number:**<br>`(config-mst)#revision <revision-number>` | Sets the MST region revision number. |
| **Map VLANs to instance:**<br>`(config-mst)#instance <instance-id> vlan <vlan-list>` | Maps specified VLANs to an MST instance. |
| **Verify current MST config:**<br>`(config-mst)#show current` | Displays the candidate MST region configuration. |

**CCNP Enterprise — ENCOR 350-401 v1.2 — IOS-XE MST Root Placement**

| Command | Description |
|---|---|
| **Set MST primary root:**<br>`(config)#spanning-tree mst <instance-id> root primary` | Sets preferred root priority for the MST instance. |
| **Set MST secondary root:**<br>`(config)#spanning-tree mst <instance-id> root secondary` | Sets backup-root priority for the MST instance. |
| **Set MST instance priority:**<br>`(config)#spanning-tree mst <instance-id> priority <priority>` | Sets bridge priority for the specified MST instance. |

**CCNP Enterprise — ENCOR 350-401 v1.2 — IOS-XE MST Interface Tuning**

| Command | Description |
|---|---|
| **Set MST interface cost:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#spanning-tree mst <instance-id> cost <cost>` | Sets interface cost for the specified MST instance. |
| **Set MST port priority:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#spanning-tree mst <instance-id> port-priority <priority>` | Sets interface priority for the specified MST instance. |

**CCNP Enterprise — ENCOR 350-401 v1.2 — IOS-XE BPDU Guard Recovery**

| Command | Description |
|---|---|
| **Enable BPDU Guard recovery:**<br>`(config)#errdisable recovery cause bpduguard` | Enables automatic recovery from BPDU Guard errdisable events. |
| **Set recovery interval:**<br>`(config)#errdisable recovery interval <seconds>` | Sets the global errdisable automatic-recovery interval. |
| **Verify recovery settings:**<br>`#show errdisable recovery` | Displays enabled recovery causes and configured interval. |

**CCNP Enterprise — ENCOR 350-401 v1.2 — IOS-XE Advanced Verification**

| Command | Description |
|---|---|
| **Show inconsistent ports:**<br>`#show spanning-tree inconsistentports` | Displays ports blocked by spanning-tree consistency mechanisms. |
| **Show MST configuration:**<br>`#show spanning-tree mst configuration` | Displays MST region attributes and VLAN-to-instance mappings. |
| **Show MST digest:**<br>`#show spanning-tree mst configuration digest` | Displays MST region configuration digest values. |
| **Show all MST instances:**<br>`#show spanning-tree mst` | Displays consolidated state for all MST instances. |
| **Show one MST instance:**<br>`#show spanning-tree mst <instance-id>` | Displays topology information for one MST instance. |
| **Show MST interface:**<br>`#show spanning-tree mst interface <interface-id>` | Displays MST state and settings for one interface. |
| **Show interface STP details:**<br>`#show spanning-tree interface <interface-id> detail` | Displays detailed guards, filters, roles, and BPDU counters. |

