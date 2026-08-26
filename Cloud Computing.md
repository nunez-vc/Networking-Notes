<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# Cloud Computing

> **Core idea:** Cloud computing delivers compute, storage, networking, platforms, or applications as **on-demand services from pooled infrastructure**, exposed through APIs/portals and designed for rapid provisioning, elasticity, and measured consumption. The key engineering shift is that infrastructure is consumed as programmable services rather than individually built and operated physical systems.

---

## 1. What It Is

**Cloud computing** is an operating model for delivering IT resources as remotely consumable services with on-demand provisioning, shared resource pools, elastic capacity, broad network access, and measured usage.

It can provide anything from raw virtual infrastructure to complete applications:

```text
IaaS  → Infrastructure
PaaS  → Application platform
SaaS  → Complete application
```

> **Virtualization is an enabling technology; it is not the same thing as cloud computing.** A virtualized data center becomes cloud-like when resources are automated, self-service, elastic, network-accessible, pooled, and measured.

---

## 2. How It Works

## Core Operating Model

A simplified cloud workflow is:

```text
User / Application
       ↓
Portal / API / IaC
       ↓
Cloud Control Plane
       ↓
Resource Scheduler / Orchestration
       ↓
Compute + Network + Storage Resources
       ↓
Running Service
```

The consumer requests a service rather than manually coordinating individual server, network, and storage changes.

Examples:

```text
Create VM
Create network
Allocate storage
Deploy container
Create managed database
Scale application
```

The provider control plane translates those requests into infrastructure state.

---

## Essential Characteristics

A cloud service normally has these characteristics:

| Characteristic | Meaning |
|---|---|
| **On-demand self-service** | Consumers provision resources without manual provider intervention |
| **Broad network access** | Services are reachable through standard network mechanisms |
| **Resource pooling** | Shared infrastructure is dynamically allocated among consumers |
| **Rapid elasticity** | Capacity can expand or contract quickly |
| **Measured service** | Usage is metered for visibility, quotas, or billing |

These characteristics distinguish cloud computing from simply hosting a VM in a traditional data center.

---

# Service Models

## IaaS — Infrastructure as a Service

The provider supplies the underlying infrastructure:

```text
Physical compute
Physical networking
Physical storage
Hypervisor / virtualization layer
```

The customer typically manages:

```text
Guest operating system
Applications
Application data
Host-level security/configuration
Much of the virtual network policy
```

Conceptually:

```text
Provider:
Hardware + Hypervisor
        ↓
Customer:
VM OS + Applications + Data
```

Typical use:

```text
Custom server workloads
Lift-and-shift migrations
Virtual routers/firewalls
Systems requiring OS-level control
```

IaaS provides the most control of the common cloud service models, but also leaves the customer with the most operational responsibility.

---

## PaaS — Platform as a Service

The provider operates more of the application stack.

The customer normally manages:

```text
Application code
Application configuration
Data
Identity/access assigned to the application
```

The provider manages much of:

```text
Operating system
Runtime
Middleware
Underlying compute/network/storage
```

Typical use:

```text
Web applications
APIs
Application development
Managed application runtimes
Managed databases or similar platform services
```

PaaS reduces infrastructure management but increases dependence on provider-specific platform behavior and APIs.

---

## SaaS — Software as a Service

The provider delivers a complete application.

```text
Customer
→ Uses/configures application

Provider
→ Operates application and underlying platform/infrastructure
```

Typical customer responsibilities still include:

```text
User identities
Access permissions
Data classification
Application configuration
Endpoint security
Data handling
```

Examples include hosted collaboration, email, file sharing, CRM, and similar finished applications.

> SaaS does not remove customer security responsibility; it moves most infrastructure and application-operation responsibility to the provider.

---

## Serverless / FaaS

**Serverless** means the consumer does not manage the underlying servers directly. The provider allocates runtime capacity when code is invoked.

Conceptually:

```text
Event
  ↓
Function invoked
  ↓
Provider allocates runtime
  ↓
Code executes
  ↓
Runtime scales or is released
```

Typical triggers:

```text
HTTP request
Queue/message
Object upload
Database event
Timer
```

Serverless is useful for event-driven or bursty workloads, but execution limits, cold-start behavior, state handling, and provider-specific integration must be considered.

---

# Deployment Models

## Public Cloud

Infrastructure is operated by a cloud provider and consumed by multiple customers with logical isolation.

```text
Enterprise
    |
Internet / VPN / Private Interconnect
    |
Public Cloud Provider
```

The customer does not control the provider's physical infrastructure.

---

## Private Cloud

Cloud operating principles are applied to infrastructure dedicated to one organization.

```text
Self-service
Automation
Resource pooling
Elastic provisioning
Measured usage
```

A private cloud can run on-premises or on dedicated hosted infrastructure.

> An on-premises virtualized environment is not automatically a private cloud unless the cloud operating characteristics are present.

---

## Hybrid Cloud

A hybrid design integrates multiple environments, commonly:

```text
On-premises
+
Public cloud
```

Applications, users, or data may cross between them.

Hybrid cloud therefore depends heavily on:

```text
Routing
DNS
Identity
Security policy
Connectivity
Address planning
Observability
```

---

## Multicloud

**Multicloud** means using services from more than one cloud provider.

```text
Cloud A
+
Cloud B
+
Possibly on-premises
```

It can reduce dependence on one provider or satisfy application/regulatory requirements, but it does not automatically provide resilience because providers use different architectures, IAM models, APIs, and network constructs.

---

# Resource Abstraction

Cloud platforms abstract physical infrastructure into logical resources.

## Compute

Common execution models:

```text
Virtual machines
Containers
Managed container platforms
Serverless functions
```

### Virtual Machines

A hypervisor allocates physical resources to VMs:

```text
Physical CPU → vCPU
Physical RAM → Virtual memory allocation
Physical NIC → vNIC
Physical storage → Virtual disk
```

Each VM normally runs its own guest operating system.

---

### Containers

Containers share the host operating-system kernel while isolating application processes.

```text
Application
+
Libraries/runtime
+
Container
```

Compared with VMs:

```text
Containers
= Lighter-weight
= Faster startup
= Less isolation from a separate guest kernel

VMs
= Full guest OS
= Stronger OS boundary
= More overhead
```

Container orchestration platforms automate placement, scaling, service discovery, and recovery.

---

# Cloud Networking

Cloud networks are software-defined logical networks created through provider control planes.

Common constructs include:

```text
Virtual network / VPC / VNet
Subnets
Virtual interfaces
Route tables
Internet gateways
NAT gateways
Virtual firewalls / security groups
Load balancers
Private service endpoints
VPN gateways
Private WAN/interconnect gateways
```

Provider names and exact behavior differ, but the forwarding concepts remain familiar.

---

## Virtual Network and Subnets

A cloud virtual network defines an isolated Layer 3 address space.

Example:

```text
Cloud Virtual Network
10.20.0.0/16

├── Subnet A  10.20.1.0/24
├── Subnet B  10.20.2.0/24
└── Subnet C  10.20.3.0/24
```

Address planning matters because overlapping prefixes complicate:

```text
VPN connectivity
Private interconnects
Cloud-to-cloud routing
Mergers/integration
Network security
```

> Cloud does not eliminate IP addressing or routing. It virtualizes how those functions are implemented and controlled.

---

## Route Tables

Cloud route tables determine where subnet or interface traffic is forwarded.

Typical destinations:

```text
Local virtual network
Internet gateway
NAT gateway
VPN gateway
Private interconnect
Virtual firewall
Transit-routing service
```

Route selection still depends on the provider's documented routing behavior, normally including longest-prefix matching.

---

## Security Groups and Network Firewalls

Cloud security commonly exists at multiple layers:

```text
Instance / interface policy
Subnet/network policy
Central firewall
Application security
Identity policy
```

Many cloud security-group constructs are **stateful**, unlike a classic stateless router ACL.

Do not assume a cloud security group behaves exactly like:

```text
IOS ACL
ASA ACL
NX-OS ACL
```

The provider's documented state model, directionality, default behavior, and attachment scope must be verified.

---

## North-South vs East-West Traffic

```text
North-South
= Traffic entering or leaving the cloud environment

East-West
= Traffic between cloud workloads/services
```

Both require explicit design.

A strong Internet perimeter does not protect poorly segmented east-west traffic between internal workloads.

---

# Connectivity to Cloud

## Internet Access

```text
Enterprise
    |
Internet
    |
Cloud
```

Advantages:

```text
Widely available
Fast to provision
Low transport complexity
```

Considerations:

```text
Variable Internet path
Public exposure
Encryption requirements
Internet-edge security
```

---

## Site-to-Site VPN

```text
Enterprise Edge
      ║
      ║ IPsec
      ║
Cloud VPN Gateway
```

Useful for:

```text
Encrypted connectivity
Rapid deployment
Backup connectivity
Moderate bandwidth requirements
```

The IPsec tunnel still depends on Internet or underlay reachability.

---

## Private Interconnect

A provider/private circuit connects the enterprise WAN directly to cloud-provider infrastructure.

Typical benefits:

```text
More predictable routing
Higher bandwidth options
Reduced dependence on public Internet paths
```

It does not inherently mean:

```text
Encrypted
Secure by itself
Internet-free end to end
```

Encryption may still be required by policy.

---

## BGP and Dynamic Routing

BGP is commonly used on:

```text
Cloud VPNs
Private interconnects
Transit gateways / hubs
Virtual routers
Hybrid-cloud WANs
```

Conceptually:

```text
Enterprise prefixes
      ↓
BGP
      ↓
Cloud edge

Cloud prefixes
      ↓
BGP
      ↓
Enterprise edge
```

Route filtering, summarization, default-route behavior, and failover must be designed explicitly.

---

# Control Plane vs Data Plane

Cloud environments separate resource management from packet/application forwarding.

## Control Plane

Used to:

```text
Create/delete resources
Change route tables
Modify security policy
Attach interfaces
Scale workloads
Manage identity
Deploy infrastructure
```

Accessed through:

```text
Web portal
CLI
API
SDK
Infrastructure as Code
```

---

## Data Plane

Carries actual workload traffic:

```text
Client → Application
VM → Database
Container → API
On-premises → Cloud workload
```

A control-plane configuration can be correct while the data plane still fails because of:

```text
Routing
DNS
Security policy
NAT
MTU
Application state
Return path
```

---

# Automation and Infrastructure as Code

Cloud resources are API-driven, making automated provisioning a normal operating model.

Example lifecycle:

```text
Desired configuration
      ↓
Terraform / provider-native IaC / API
      ↓
Cloud control plane
      ↓
Create/update resources
      ↓
Validate deployed state
```

Benefits include:

```text
Repeatability
Version control
Reviewable changes
Consistent environments
Faster recovery/rebuild
```

> Manual portal changes outside the managed configuration can create **configuration drift**.

---

# Availability and Failure Domains

Cloud availability is not automatic.

Providers expose multiple failure-domain scopes, commonly concepts such as:

```text
Availability zone
Region
Service-specific redundancy domain
```

A resilient design distributes dependencies so that one failure does not remove the entire application.

Example:

```text
Region
├── Zone A → App instance
├── Zone B → App instance
└── Load balancer
```

Multi-zone deployment protects against some localized failures.

Multi-region designs can tolerate a larger failure domain but require additional work:

```text
Data replication
DNS/global load balancing
Routing
Failover logic
Consistency model
Higher cost
```

> Creating two VMs in the same failure domain is not equivalent to designing high availability.

---

# Shared Responsibility

Cloud security and operations are divided between the provider and the customer.

The exact boundary depends on the service model.

```text
More customer control
IaaS ──────────────── PaaS ──────────────── SaaS
More provider responsibility →
```

Typical IaaS customer responsibilities include:

```text
Guest OS patching
Application security
IAM
Data protection
Virtual network policy
Logging/monitoring configuration
```

The provider normally operates:

```text
Physical facilities
Physical network
Physical compute/storage
Hypervisor/platform layers as applicable
```

In SaaS, the provider manages much more of the technical stack, but the customer still owns decisions such as user access, data use, and service configuration.

---

# Identity and Security

In cloud environments, **identity is a primary security boundary**.

Control-plane access should use:

```text
Least privilege
Role-based access
MFA where supported
Short-lived credentials where practical
Service identities for workloads
Central logging
```

Avoid long-lived shared administrative credentials.

Security architecture should also address:

```text
Encryption in transit
Encryption at rest
Key management
Network segmentation
Secrets management
Logging
Backup/recovery
Vulnerability management
Compliance
```

---

# Observability

Cloud systems are highly dynamic, so static diagrams alone are insufficient.

Operational visibility normally requires:

```text
Control-plane audit logs
Flow/network telemetry
Application logs
Metrics
Traces
Security events
Cost/usage telemetry
```

The ability to create resources rapidly must be paired with the ability to determine:

```text
Who changed what?
When?
Where?
Why?
What traffic was affected?
```

---

## 3. Why and When It Is Used

Cloud computing is appropriate when an organization needs:

- Rapid infrastructure or application provisioning.
- Elastic capacity for variable demand.
- API-driven automation.
- Globally distributed services.
- Managed platforms that reduce infrastructure operations.
- Faster development/test environments.
- Temporary or burst workloads.
- Hybrid or cloud-native application architectures.

It is especially useful when resource demand changes faster than traditional procurement and deployment cycles can accommodate.

Cloud computing is **not automatically the best fit** when:

- Regulatory or data-sovereignty requirements prohibit the intended deployment.
- Latency requires compute to remain physically close to users/devices.
- Specialized hardware is unavailable or cost-prohibitive in cloud.
- A stable, heavily utilized workload is materially more economical on dedicated infrastructure.
- Provider dependency or service limits violate business requirements.

---

## 4. Key Configuration, Parameters, or CLI

> **Platform:** Cloud computing has no universal CLI. AWS, Microsoft Azure, Google Cloud, private-cloud platforms, and Cisco-connected cloud designs use different APIs and configuration models. The parameters below are the common engineering controls that materially affect a cloud deployment.

---

## Network Parameters

Verify and document:

```text
Virtual-network CIDR
Subnet CIDRs
Route tables
Default routes
NAT behavior
Internet gateways
VPN/private-interconnect routing
BGP ASN and advertised prefixes
DNS resolution
MTU
Security-policy attachment points
```

Example address plan:

```text
Cloud VPC/VNet: 10.20.0.0/16

Application: 10.20.10.0/24
Database:    10.20.20.0/24
Management:  10.20.30.0/24
```

Avoid overlap with:

```text
On-premises networks
Other cloud networks
Partner networks
Future expansion ranges
```

---

## Security Parameters

Validate:

```text
IAM roles and permissions
MFA requirements
Security groups / firewall rules
Public IP exposure
Encryption settings
Key ownership/rotation
Logging and audit configuration
Secrets storage
Backup/retention policy
```

---

## Availability Parameters

Validate:

```text
Zone placement
Region placement
Load-balancer health checks
Autoscaling thresholds
Data replication
Backup restore testing
Failover dependencies
```

---

## Connectivity Verification

From the enterprise/Cisco edge, normal network verification still applies.

> **Platform: Cisco IOS / IOS XE**

```cisco
show ip route <cloud-prefix>
show bgp ipv4 unicast summary
show bgp ipv4 unicast <cloud-prefix>
show crypto ikev2 sa
show crypto ipsec sa
ping <cloud-address>
traceroute <cloud-address>
```

Use only the commands relevant to the actual design:

```text
BGP
→ Private interconnect / dynamic VPN routing

IKEv2/IPsec
→ Site-to-site VPN

Normal routing
→ Reachability validation
```

Cloud-provider-side verification must use the specific provider's route-table, flow-log, VPN, firewall, and health-status tools.

---

## Practical Troubleshooting Sequence

```text
1. Is the workload/service actually running?
        ↓
2. Is DNS resolving to the intended address?
        ↓
3. Are source and destination prefixes correct?
        ↓
4. Does the cloud route table have the required route?
        ↓
5. Does the enterprise/hybrid edge have the return route?
        ↓
6. Are security groups/firewalls permitting the flow?
        ↓
7. Is NAT changing the address unexpectedly?
        ↓
8. Is VPN/BGP/private interconnect healthy?
        ↓
9. Is MTU causing large-packet failure?
        ↓
10. Is the application listening and healthy?
```

Cloud troubleshooting is still:

```text
DNS
→ Routing
→ Policy
→ NAT
→ Transport
→ Application
```

but the control points are software-defined and may exist across multiple provider services.

---

## 5. Common Gotchas and Misconceptions

### Virtualization and Cloud Computing Are the Same Thing

**Incorrect.**

Virtualization abstracts hardware.

Cloud computing adds:

```text
Self-service
Automation
Resource pooling
Elasticity
Broad network access
Measured consumption
```

---

### Moving to Cloud Removes Infrastructure Responsibility

**Incorrect.**

Responsibility changes according to the service model.

```text
IaaS
= Customer manages more

SaaS
= Provider manages more
```

The customer still owns security, identity, data, and configuration responsibilities appropriate to the service.

---

### The Cloud Is Automatically Highly Available

**Incorrect.**

A single workload in one zone or region can still fail.

High availability must be explicitly designed across appropriate failure domains.

---

### Public Cloud Means Everything Has a Public IP Address

**Incorrect.**

Cloud workloads can be deployed entirely on private address space and reached through:

```text
VPN
Private interconnect
Transit network
Private service endpoint
Load balancer
```

Public-cloud ownership and public-IP exposure are separate concepts.

---

### A Private Interconnect Automatically Encrypts Traffic

**Incorrect.**

A private circuit provides a private connectivity path, not necessarily cryptographic confidentiality.

Use encryption when security policy requires it.

---

### Cloud Security Groups Are Just Cisco ACLs

**Incorrect.**

Cloud security constructs can differ in:

```text
Statefulness
Directionality
Attachment scope
Default behavior
Evaluation model
```

Verify the provider's documented behavior.

---

### Autoscaling Fixes Every Capacity Problem

**Incorrect.**

Autoscaling depends on:

```text
Application architecture
Startup time
Quotas
Database/storage scalability
Correct health metrics
Dependency capacity
```

A bottleneck that cannot scale may still limit the entire service.

---

### Multicloud Automatically Improves Availability

**Incorrect.**

Multicloud can increase resilience only if applications, identity, networking, data, and failover are deliberately designed across providers.

Otherwise it mainly increases complexity.

---

### Cloud Eliminates Network Design

**Incorrect.**

Cloud environments still require:

```text
IP addressing
Routing
DNS
Segmentation
NAT
Load balancing
Firewalls
VPN/BGP
MTU design
Observability
```

The implementation is virtualized, not eliminated.

---

### Portal Changes Are Harmless Because They Are Easy to Make

**Incorrect or Unsafe.**

Manual changes can create configuration drift and bypass review/automation controls.

Use Infrastructure as Code or another controlled change mechanism where practical.

---

## 6. Trade-Offs

### Best Practice

- Design cloud networks with **non-overlapping, summarizable IP space**.
- Treat identity and IAM as a primary security boundary.
- Use least privilege and strong administrative authentication.
- Use Infrastructure as Code for repeatable environments.
- Design availability around explicit failure domains.
- Enable audit, flow, application, and security logging before production use.
- Test backup restoration and failover rather than assuming provider redundancy is sufficient.
- Control public exposure explicitly.

---

### Context-Dependent Trade-Off — IaaS vs PaaS vs SaaS

```text
IaaS
+ Maximum infrastructure control
+ Broad workload compatibility
- Highest customer operational responsibility

PaaS
+ Less OS/platform management
+ Faster application delivery
- More provider/platform dependency

SaaS
+ Lowest infrastructure-management burden
+ Fastest consumption
- Least infrastructure/application control
```

Choose the highest-level managed service that still satisfies technical, security, compliance, and portability requirements.

---

### Context-Dependent Trade-Off — Public vs Private Cloud

**Public Cloud**

```text
+ Rapid elasticity
+ Large managed-service portfolio
+ Global footprint
- Provider dependency
- Consumption-cost variability
- Less physical infrastructure control
```

**Private Cloud**

```text
+ Greater infrastructure control
+ Can satisfy specific locality/security requirements
- Organization owns capacity planning
- Higher operational burden
- Elasticity limited by owned capacity
```

---

### Context-Dependent Trade-Off — VPN vs Private Interconnect

**Internet VPN**

```text
+ Fast deployment
+ Encryption
+ Lower entry cost
- Internet path variability
- Tunnel throughput limits may apply
```

**Private Interconnect**

```text
+ Predictable private connectivity
+ Higher bandwidth options
+ Direct cloud-provider access
- Higher cost
- Longer provisioning
- Encryption may still be required
```

Many production designs use both:

```text
Private interconnect
= Primary

VPN
= Backup
```

when business requirements justify the cost.

---

### Context-Dependent Trade-Off — Single Region vs Multi-Region

**Single Region**

```text
+ Simpler
+ Lower cost
+ Easier data consistency
- Region-level dependency
```

**Multi-Region**

```text
+ Larger failure-domain protection
+ Geographic service placement
- Higher cost
- More routing/DNS complexity
- Harder data replication and consistency
```

Use multi-region architecture only when the availability or geographic requirement justifies the operational complexity.

---

### Incorrect or Unsafe

- Assuming the provider secures customer identities, data, and application configuration automatically.
- Connecting cloud networks without checking for overlapping IP address space.
- Giving workloads or users broad administrative IAM permissions for convenience.
- Exposing management interfaces directly to the Internet without an explicit security design.
- Building critical services in one failure domain while claiming provider-level high availability.
- Treating backups as valid without testing restoration.
- Deploying cloud resources manually at scale without controls for configuration drift, auditability, and repeatability.

---

## Quick Reference

```text
Cloud Computing
= On-demand IT services from pooled, programmable resources

Essential Characteristics
= On-demand self-service
= Broad network access
= Resource pooling
= Rapid elasticity
= Measured service

IaaS
= Provider manages infrastructure
= Customer manages OS/apps/data

PaaS
= Provider manages infrastructure + platform
= Customer manages apps/data

SaaS
= Provider operates complete application
= Customer manages usage/access/data responsibilities

Public Cloud
= Provider-operated shared infrastructure

Private Cloud
= Cloud model dedicated to one organization

Hybrid Cloud
= On-prem/private + public cloud integration

Multicloud
= Multiple cloud providers

Compute
= VMs
= Containers
= Serverless

Cloud Networking
= Virtual networks
= Subnets
= Route tables
= NAT
= Firewalls/security groups
= Load balancers
= VPN/private interconnect

Control Plane
= API / portal / IaC configuration

Data Plane
= Actual workload traffic

Shared Responsibility
= Provider + customer responsibilities vary by service model

High Availability
= Must be designed across failure domains

Core Rule
= Cloud changes how infrastructure is consumed and controlled;
  it does not remove routing, security, identity, availability,
  or operational engineering requirements.
```

</div>
