**Spanning Tree Protocol**
- It is a L2 control protocol that prevent switching loops.

**Note:**
- L2 frames do not have a TTL field and can circulate endlessly in a loop (e.g. Broadcast Storm and MAC address Flaps). 

**Root Bridge**
- Determined by the ethernet switch that has the lowest Bridge ID.  

**Bridge ID (BID)**
- Default Bridge Priority is **32768.**  

**Root Port**
- The one (and only one) port on a non-root bridge that's closest to the Root Bridge, in terms of cost.
- Tie Breakers:
1. Lowest Cost Bridge ID.
2. Lowest Port ID connected to the Upstream Switch.

**Short Path Cost Method**

**Long Path Cost Method**
Formula: 20 Tbps / Port Speed

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

**MST Configuration:**
spanning-tree mst configuration
instance <instance #> vlan <VLAN_IDs>
exit
spanning-tree mode mst  

**Rapid Per VLAN Spanning Tree Protocol (RPVST+)**
- IEEE 802.1W
- completely eliminates Listening State.
- Converge within a few seconds (typically 1 to 2 sec, and 10 sec at most) from blocking state to forwarding state.
