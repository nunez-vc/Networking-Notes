**Spanning Tree Protocol**
- It is a L2 control protocol that prevent switching loops.

**Note:**
- L2 frames do not have a TTL field and can circulate endlessly in a loop (e.g. Broadcast Storm and MAC address Flaps). 

**Root Bridge**
- Determined by the ethernet switch that has the lowest Bridge ID.  

**Bridge ID (BID)**
- Default Bridge Priority is **32768.**  
<p align="center">
  <img width="400" alt="Local Account, Named ACL, and Security" src="Images/STP/Bridge ID.png" />
</p>


**Root Port**
- The one (and only one) port on a non-root bridge that's closest to the Root Bridge, in terms of cost.
- Tie Breakers:
1. Lowest Cost Bridge ID.
2. Lowest Port ID connected to the Upstream Switch.

**Short Path Cost Method**
<p align="center">
  <img width="400" alt="Local Account, Named ACL, and Security" src="Images/STP/Short Path Cost Method.png" />
</p>

**Long Path Cost Method**
Formula: 20 Tbps / Port Speed
<p align="center">
  <img width="400" alt="Local Account, Named ACL, and Security" src="Images/STP/Long Path Cost Method.png" />
</p>

**Designated Port**
- The one (and only one) port on each segment that is closest to the Root Bridge in terms of cost.
- These are all downstream.
- Tie Breaker: Switch Bridge ID.  


**Blocking (Non-designated) Port**
- A port that is administratively enabled, but is not a root port nor a designated port.  

**Note:**
- Hello timers are **2** seconds by default.  

**Bridge Protocol Data Unit (BPDU)**
- It is a network packet, transmitted to the multicast MAC address (**01:80:C2:00:00:00**) to switches running STP to dynamically discover neighboring switches, elect a root bridge, identify network path hierarchies, and notify the system of any topology.  

**Note:**
- Blocking State (**20** sec).
- Listening State (**15** sec).
- Learning State (**15** sec).
- Total of **50** sec for transitioning from blocking state into forwarding state.
- Total of **30** sec for transitioning from unused port into forwarding state.  

**Common Spanning Tree (CST)**
- IEEE 802.1D
- Same spanning tree used by all VLANs.  

**Per VLAN Spanning Tree (PVST/PVST+)**
- Each VLAN runs its STP.
- The "+" indicates the switches are connected via 802.1Q trunk encapsulation.  

**Multiple Spanning Tree Protocol (MSTP)**
- IEEE 802.1s
- Instead of configuring STP per VLAN, in MSTP you group VLANs in instances.
<p align="center">
  <img width="400" alt="Local Account, Named ACL, and Security" src="Images/STP/Long Path Cost Method.png" />
</p> 

**Multiple Spanning Tree Configuration:**
```
spanning-tree mst configuration
instance [instance  #] vlan [VLAN ID]
exit
spanning-tree mode mst
```

**Rapid Per VLAN Spanning Tree Protocol (RPVST+)**
- IEEE 802.1W
- completely eliminates Listening State.
- Converge within a few seconds (typically 1 to 2 sec, and 10 sec at most) from blocking state to forwarding state.
- RSTP+ states: Discarding, Learning, and Forwarding.

<p align="center">
  <img width="550" alt="Local Account, Named ACL, and Security" src="Images/STP/Summary Reference Table.png" />
</p>

**PortFast**
- Bypass listening and learning state.
- Can be enabled globally or on a port-by-port basis (for non-trunking ports).
- To enable portfast globally: spanning-tree portfast default
- To enable portfast port-by-port basis: spanning-tree portfast

**UplinkFast**
- Globally enabled on a switch.
- It enables rapid convergence during a root port link failure by immediately transitioning a blocked alternate port into the forwarding state to serve as the new root port, completely bypassing the traditional 30-second Spanning Tree listening and learning delays.
- To enable UplinkFast globally: spanning-tree uplinkfast

**BackboneFast**
- Globally enabled on a switch.
- Reacts to an indirect link failure.
- To enable BackboneFast globally: spanning-tree backbonefast

**BPDU Guard**
- It is a security feature that protects PortFast-enabled access ports by automatically placing the interface into an err-disabled state to prevent loops the moment any Spanning Tree BPDU is received on the port.
- Should be enabled on ports with PortFast enabled.
- Can be enabled globally or on a port-by-port basis (for ports with PortFast enabled).

**BPDU Filter**
- This feature either conditionally monitors PortFast ports globally, disabling PortFast and reverting to normal STP if a BPDU is received, or absolutely disables STP on a specific interface when configured locally by silently discarding all incoming and outgoing BPDUs.
- Prevents a port from sending BPDUs.
- Should be only used when necessary.
- Most dangerous when enabled at the port level.

**Root Guard**
- This prevents unauthorized or rogue downstream switches from becoming the root bridge by dynamically putting the port into a non-forwarding root-inconsistent (broken) state if a superior BPDU is received, recovering automatically once those superior BPDUs cease.
- Configured on ports off of which the Root Bridge is unexpected.

**Loop Guard**
- This prevents Layer 2 forwarding loops caused by unidirectional link failures.
- This causes a non-designated port to enter Loop Inconsistent (broken) state if it stops receiving BPDUs.
