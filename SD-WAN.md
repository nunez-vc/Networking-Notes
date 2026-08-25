**SD-WAN**
- It is a WAN architecture that uses centralized controllers and software-defined policies to connect branches, campuses, data centers, and cloud across different WAN transports.
- Instead of manually configuring every WAN router, SD-WAN lets you centrally control how traffic travel across links such as: Internet, MPLS, and LTE/5G.

**Traditional WAN vs SD-WAN**
<p align="center">
  <img width="600" alt="Local Account, Named ACL, and Security" src="Images/SD-WAN/Traditional WAN vs SD-WAN.png" />
</p>

**VPN vs SD-WAN**
- A traditional site-to-site VPN mainly gives you secure connectivity between sites.
- Meanwhile, SD-WAN adds centralized control, dynamic path selection, application-aware routing, policy, automation, and multi-transport support.
- VPN = secure tunnel
- SD-WAN = centrally managed WAN that uses secure VPN tunnels as part of the design.

**Cisco SD-WAN Components**
**1. vManage**
- Management
- GUI for configuration, monitoring and policy.

**2. vSmart**
- Control
- Distributes routing information and policies.

**3. vBond**
- Orchestration/Authentication
- Authenticates devices and helps them establish controller connectivity.

**4. SD-WAN Edge/cEdge**
- Data forwarding
- Actual routers forwarding user traffic.

**Underlay vs Overlay**
**Underlay**
- Physical/IP WAN connectivity
- MPLS, Internet, LTE, etc.

**Overlay**
- Secure SD-WAN tunnels built on top
- Logical SD-WAN fabric

