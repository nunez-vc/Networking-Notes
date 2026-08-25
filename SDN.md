**Software Defined Networking (SDN)**
<p align="justify">
- A controller-based networking architecture that abstracts network control from individual devices and enables centralized or logically centralized policy, automation, programmability, and network-wide management while network devices continue performing high-speed data plane forwarding.
- 
</p>

**3 Networking Planes**
1. Data Plane
- Forwards traffic
- Handles actual frames and packets
- e.g. CEF, MAC lookup, packet forwarding, ACL enforecement

2. Control Plane
- Decides how traffic should be forwarded
- Creates the information used by the data plane
- e.g. OSPF, EIGRP, BGP, RIP, STP, ARP/NDP

3. Management Plane
- Configure and monitor devices
- How administrator and management systems interact with network devices
- e.g. SSH, SNMP, NETCONF, RESTCONF, GUI


**Northbound and Southbound Interfaces**
<p align="center">
  <img width="500" alt="Image" src="Images/.png" />
</p>

**Northbound Interface**
- Connects application or automation systems to the controller.

**Southbound Interface**
- Connects the controller to infrastructure devices
