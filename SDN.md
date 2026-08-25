# Software-Defined Networking (SDN)

> **Software-Defined Networking (SDN)** is a controller-based networking architecture that separates network control logic from individual network devices, enabling centralized or logically centralized policy enforcement, automation, programmability, and network-wide management.

Network devices such as routers, switches, firewalls, and wireless access points continue to perform high-speed **data-plane forwarding**, while the SDN controller provides centralized control and network intelligence.

---

## SDN Architecture

SDN generally organizes network functionality into layers, with the **SDN Controller** serving as the central point of communication between applications and network infrastructure.

### Complete SDN Architecture

<div align="center">
  <img
    src="Images/SDN/SDN Complete Architecture.png"
    alt="SDN Complete Architecture"
    width="600"
  />
</div>

### Key Components

- **Application / Business Layer**
  - Contains network applications, automation platforms, orchestration systems, and business logic.
  - Communicates with the SDN controller through **northbound APIs**.

- **SDN Controller**
  - Maintains a logically centralized view of the network.
  - Collects and processes network state and topology information.
  - Translates application requirements and policies into network configurations.
  - Provides automation, programmability, analytics, and centralized control.

- **Infrastructure / Data Plane**
  - Contains physical and virtual network devices.
  - Performs actual packet and frame forwarding.
  - Receives forwarding rules, configuration, and policies from the SDN controller.

---

# Three Networking Planes

Network operations are commonly divided into three logical planes:

1. **Data Plane**
2. **Control Plane**
3. **Management Plane**

---

## 1. Data Plane

> **Purpose:** Forward network traffic.

The **Data Plane**, also known as the **Forwarding Plane**, is responsible for processing and forwarding actual frames and packets through a network device.

### Responsibilities

- Forwards packets and frames.
- Performs Layer 2 and Layer 3 forwarding.
- Applies forwarding decisions created by the control plane.
- Enforces policies directly on network traffic.
- Processes traffic at high speed.

### Examples

- Cisco Express Forwarding (**CEF**)
- MAC address table lookup
- Routing/FIB lookup
- Packet forwarding
- Frame forwarding
- Access Control List (**ACL**) enforcement
- Quality of Service (**QoS**) processing
- Network Address Translation (**NAT**)

---

## 2. Control Plane

> **Purpose:** Determine how traffic should be forwarded.

The **Control Plane** makes network forwarding decisions and builds the information that the data plane uses to forward traffic.

### Responsibilities

- Learns network topology.
- Exchanges routing information.
- Calculates optimal paths.
- Builds routing and forwarding information.
- Maintains Layer 2 and Layer 3 control information.
- Responds to network topology changes.

### Examples

- Open Shortest Path First (**OSPF**)
- Enhanced Interior Gateway Routing Protocol (**EIGRP**)
- Border Gateway Protocol (**BGP**)
- Routing Information Protocol (**RIP**)
- Spanning Tree Protocol (**STP**)
- Address Resolution Protocol (**ARP**)
- IPv6 Neighbor Discovery Protocol (**NDP**)

---

## 3. Management Plane

> **Purpose:** Configure, monitor, and administer network devices.

The **Management Plane** defines how network administrators, management platforms, and automation systems interact with network devices.

### Responsibilities

- Device configuration
- Network monitoring
- Logging and telemetry
- Troubleshooting
- Software and firmware management
- Configuration backup and restoration
- Network automation

### Examples

- Secure Shell (**SSH**)
- Simple Network Management Protocol (**SNMP**)
- NETCONF
- RESTCONF
- Web-based GUI
- APIs
- Syslog
- Streaming telemetry

---

## Networking Planes Summary

| Plane | Primary Function | Examples |
|---|---|---|
| **Data Plane** | Forwards actual network traffic | CEF, MAC lookup, FIB lookup, ACL enforcement, NAT |
| **Control Plane** | Determines how traffic should be forwarded | OSPF, EIGRP, BGP, RIP, STP, ARP/NDP |
| **Management Plane** | Configures and monitors network devices | SSH, SNMP, NETCONF, RESTCONF, GUI, Syslog |

---

# SDN Interfaces

The SDN controller communicates with applications and network infrastructure through two primary interface categories:

- **Northbound Interfaces**
- **Southbound Interfaces**

<div align="center">
  <img
    src="Images/SDN/Northbound and Southbound Interfaces.png"
    alt="SDN Northbound and Southbound Interfaces"
    width="600"
  />
</div>

---

## Northbound Interface

> **Application → SDN Controller**

A **Northbound Interface (NBI)** connects applications, orchestration platforms, automation systems, and business services to the SDN controller.

It allows applications to communicate their desired network behavior or policies without directly configuring individual network devices.

### Common Uses

- Network automation
- Policy definition
- Network orchestration
- Application-driven networking
- Network analytics
- Intent-based networking

### Common Technologies

- REST APIs
- RESTful APIs
- HTTP/HTTPS
- Controller-specific APIs
- Software Development Kits (**SDKs**)

## Southbound Interface

> **SDN Controller → Network Devices**

A **Southbound Interface (SBI)** connects the SDN controller to network infrastructure devices such as routers, switches, firewalls, and wireless access points.

It allows the controller to **configure, manage, and control network devices**, translating high-level policies and network intent into device-level instructions.

### Common Uses

* Device configuration
* Forwarding and routing control
* Policy enforcement
* Network monitoring
* Collecting device state and telemetry
* Automating network changes

### Common Technologies

* NETCONF
* RESTCONF
* OpenFlow
* gNMI
* SNMP
* Device-specific APIs and protocols
