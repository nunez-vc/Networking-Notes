Local Area Network (LAN)
- A LAN is a single broadcast domain, including all devices in that broadcast domain.

Broadcast Domain
- It is the group of devices which will receive a Broadcast Frame (Destination MAC: FFFF.FFFF.FFFF.FFFF) sent by any one of the members.

Broadcast Frame
- Flooding all our subnets with uncessary traffic.

VLAN
- Logically separate end-host at layer 2.
- One VLAN = One Layer 2 broadcast domain.

Purpose of VLANs:
1. Network Performance:
- Reduce unnecessary Broadcast traffic, which helps prevent network congestion, and improve network performance.
2. Network Security:
- Limiting Broadcast and unknown Unicast traffic, also improves netwoork security, since messages won't be received by devices outside of the VLAN.

Note:
- A switch will not forward traffic between VLANs, including broadcast/unknown unicast traffic.
- A switch does not perform inter-VLAN routing. It must send the traffic through the router.

Access Ports
- It is a switchport that carries traffic for only one data VLAN.
- Typically connected to an endpoint.
- Untagged Ports

Trunk Ports
- It is a switchport that carries multiple VLANs over one physical link.
- Typically connected between devices such as switches, routers, firewalls and virtualization hosts.
- Tagged Ports
- IEEE 802.1Q 

Native VLAN
- VLAN 1 by default on all trunk ports but still can be manually configured.
- When a switch receives an untagged frame on a trunk port, it assumes the frame belongs to the native VLAN.
- It is very important that the native VLAN matches because if the two ends disagree about which VLAN is native, the same untagged frame can be interpreted as belonging to different VLANs on each switch.

VLAN Ranges
- The range of VLANs (1-4096) is divided into two sections:
1. Normal VLANs: 1-1005
2. Extended VLANs: 1006-4094
