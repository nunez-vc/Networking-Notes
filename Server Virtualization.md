<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# Server Virtualization

> **Core idea:** Server virtualization uses a **hypervisor** to abstract physical server resources so multiple independent virtual machines (VMs) can run on one physical host. Each VM receives virtual CPU, memory, storage, and network interfaces while remaining logically isolated from other VMs.

---

## 1. What It Is

**Server virtualization** decouples operating-system instances from specific physical server hardware. A hypervisor presents virtual hardware to each VM and schedules the underlying CPU, memory, storage, and network resources among the VMs running on the host.

```text
Physical Server
      ↓
Hypervisor
      ↓
+---------+---------+---------+
|   VM1   |   VM2   |   VM3   |
| Guest OS| Guest OS| Guest OS|
+---------+---------+---------+
```

A physical server running workloads directly without a virtualization layer is commonly called **bare metal**.

---

## 2. How It Works

## Hypervisor

The **hypervisor** is the virtualization layer that creates and operates VMs.

It performs functions such as:

```text
CPU scheduling
Memory allocation
Virtual device presentation
Storage access
Virtual networking
VM lifecycle control
Isolation between VMs
```

Each VM sees virtual hardware rather than directly controlling most physical hardware.

---

## Type 1 vs Type 2 Hypervisors

### Type 1 — Bare-Metal Hypervisor

Runs directly on the server hardware.

```text
Hardware
   ↓
Hypervisor
   ↓
VMs
```

Typical production server virtualization uses Type 1 hypervisors.

---

### Type 2 — Hosted Hypervisor

Runs on top of a conventional host operating system.

```text
Hardware
   ↓
Host OS
   ↓
Hypervisor
   ↓
VMs
```

Type 2 hypervisors are more common for desktop labs, development, and end-user systems.

---

# Virtual Machine Resource Model

A VM normally receives virtualized versions of physical resources.

```text
Physical CPU     → vCPU
Physical RAM     → Virtual memory allocation
Physical disk    → Virtual disk
Physical NIC     → vNIC
Physical switch  → vSwitch / virtual network
```

The guest operating system behaves as though these resources belong to a physical server.

---

## vCPU and CPU Scheduling

A VM is assigned one or more **virtual CPUs (vCPUs)**.

```text
VM1 = 4 vCPU
VM2 = 2 vCPU
VM3 = 8 vCPU
```

The hypervisor schedules those vCPUs onto physical CPU cores or hardware threads.

Multiple VMs can therefore share the same physical CPU resources over time.

```text
VM vCPUs
    ↓
Hypervisor scheduler
    ↓
Physical CPU cores / threads
```

### CPU Overcommit

The total configured vCPU count can exceed the number of physical execution resources.

Example:

```text
Physical host = 32 logical CPU threads

Configured:
VM1 = 8 vCPU
VM2 = 8 vCPU
VM3 = 8 vCPU
VM4 = 8 vCPU
VM5 = 8 vCPU

Total = 40 vCPU
```

This can work when VMs are not all busy simultaneously.

If sustained demand exceeds physical CPU capacity:

```text
CPU contention
      ↓
Scheduling delay
      ↓
Higher VM latency
```

> More assigned vCPUs do not automatically make a VM faster. Oversized VMs can increase scheduling complexity and reduce consolidation efficiency.

---

## Memory Virtualization

Each VM is configured with virtual RAM.

```text
VM1 = 8 GB
VM2 = 16 GB
VM3 = 32 GB
```

The hypervisor maps VM memory to physical host memory.

Some platforms support memory overcommitment using mechanisms such as:

```text
Ballooning
Compression
Swapping
Deduplication
```

Exact behavior is hypervisor-specific.

When physical memory becomes constrained, VM performance can degrade sharply, especially if the platform must swap memory to storage.

---

## Virtual Storage

A VM usually sees one or more virtual disks.

```text
VM
 |
 +-- Virtual Disk 1
 +-- Virtual Disk 2
```

The virtual disk may reside on:

```text
Local host storage
SAN
NAS
Distributed storage
Hyperconverged storage
Cloud-backed storage
```

The guest OS sees a logical disk while the virtualization platform maps that disk to the underlying storage system.

---

## Thin vs Thick Provisioning

### Thin Provisioning

Logical disk capacity can be larger than the physical storage currently consumed.

```text
VM sees 500 GB
Actual used = 80 GB
```

Benefits:

```text
Higher storage utilization
Faster provisioning
```

Risk:

```text
Logical allocations can exceed real capacity
```

If physical storage is exhausted, many VMs can be affected simultaneously.

---

### Thick Provisioning

Physical storage capacity is reserved or allocated more fully in advance.

```text
+ More predictable capacity reservation
- Lower storage utilization
```

Implementation details vary by storage and hypervisor platform.

---

# Virtual Networking

## vNIC

Each VM normally has one or more **virtual NICs (vNICs)**.

From the guest OS perspective:

```text
vNIC
= Normal network adapter
```

The VM can configure:

```text
MAC address
IPv4 / IPv6
Default gateway
VLAN behavior when supported
```

just as it would on a physical server.

---

## Virtual Switch

A **virtual switch (vSwitch)** is a software-based Layer 2 forwarding function inside the virtualization host.

```text
          VM1
         vNIC
           |
VM2 ---- vSwitch ---- VM3
           |
          pNIC
           |
     Physical Switch
```

The vSwitch connects:

```text
VM vNICs
Physical NIC uplinks
Virtual network segments / port groups
```

---

## Same-Host Traffic

If two VMs are connected to the same applicable virtual Layer 2 network on the same host:

```text
VM1
 |
 vSwitch
 |
VM2
```

their traffic can be switched locally inside the host and may never reach the physical switch.

This is operationally important because:

```text
Physical switch counters may not see the flow
Physical SPAN may not capture the flow
External firewall policy may not see the flow
```

unless the architecture deliberately steers that traffic externally.

---

## Off-Host Traffic

If the destination is outside the local virtual switching domain:

```text
VM
 ↓
vNIC
 ↓
vSwitch
 ↓
pNIC
 ↓
Physical switch
 ↓
Network
```

The physical NIC is commonly called a **pNIC** or uplink.

A host often has multiple pNICs for:

```text
Redundancy
Load distribution
Management
Storage
VM data traffic
Migration traffic
```

Exact teaming and failover behavior depend on the hypervisor and physical-switch design.

---

## VLANs and Trunks

Virtual switches commonly support VLAN segmentation.

Example:

```text
VM-A → VLAN 10
VM-B → VLAN 20
VM-C → VLAN 30
```

The physical host uplink may carry several VLANs using 802.1Q:

```text
vSwitch
   |
pNIC
   |
802.1Q trunk
   |
Physical Switch
```

A VLAN mismatch between the virtual and physical switching layers can produce normal-looking VM configuration with no end-to-end connectivity.

---

# Control Plane vs Data Plane

## Management / Control Plane

The virtualization management system controls:

```text
VM creation
VM deletion
Power state
Resource allocation
Host placement
Migration
Virtual networking
Snapshots
High-availability policy
```

Management may be performed through:

```text
GUI
CLI
API
Automation platform
```

---

## Data Plane

The data plane carries actual workload traffic:

```text
VM ↔ VM
VM ↔ Physical network
VM ↔ Storage
```

A healthy virtualization management system does not prove the VM data plane is functioning.

---

# VM Lifecycle

A VM normally moves through operational states such as:

```text
Created
  ↓
Powered On
  ↓
Running
  ↓
Suspended / Paused
  ↓
Powered Off
```

The platform can also:

```text
Clone
Snapshot
Migrate
Restart
Delete
```

Exact state names and behavior vary by hypervisor.

---

# VM Migration

Virtualization allows a VM to move from one physical host to another.

```text
Host A                       Host B

+------+                     +------+
| VM1  |  ─── migration ───> | VM1  |
+------+                     +------+
```

### Live Migration

A **live migration** moves a running VM while minimizing service interruption.

The platform must coordinate:

```text
VM execution state
Memory contents
Storage accessibility or storage migration
Virtual networking
Destination-host capacity
```

Networking must remain valid after the move.

Depending on the platform/design, this may require:

```text
Equivalent VLAN/network availability
Distributed virtual networking
Overlay networking
Routing-based mobility
Correct security policy on the destination host
```

> Live migration does **not universally require extending the same Layer 2 VLAN between hosts**; the requirement depends on the virtualization and network architecture.

---

## MAC Movement During Migration

If a VM retains the same MAC address while moving between hosts, the physical network may observe:

```text
Before:
VM MAC learned behind Host A

After migration:
VM MAC appears behind Host B
```

The switching fabric must update its forwarding information accordingly.

Gratuitous ARP, Neighbor Discovery behavior, or platform-specific mobility mechanisms may also help update surrounding network state.

---

# High Availability

Virtualization platforms can restart VMs on another host after physical-host failure.

```text
Host A fails
    ↓
Cluster detects failure
    ↓
VM restarted on Host B
```

This improves infrastructure availability, but:

```text
VM restart
≠
Application-level high availability
```

There may still be:

```text
Failure-detection time
VM boot time
Application startup time
Session loss
Database recovery
```

Applications requiring near-zero interruption usually need their own clustering, replication, or load-balancing architecture.

---

# Snapshots

A VM snapshot captures selected VM state so the VM can be reverted later.

Depending on the platform, a snapshot may include:

```text
Virtual disk state
Configuration state
Memory state
```

Snapshots are useful for short-term rollback and testing.

> **A snapshot is not a backup.** Snapshots normally depend on the original VM/storage chain and should not replace independent backup and restore procedures.

---

# Resource Contention

Virtualization consolidates many workloads onto shared hardware.

The shared physical resources can become contention points:

```text
CPU
Memory
Storage IOPS
Storage latency
Network bandwidth
NIC queues
PCIe resources
```

Example:

```text
20 VMs
   ↓
2 × 25-Gbps host uplinks
   ↓
All VMs share available network capacity
```

Per-VM virtual interfaces do not create additional physical bandwidth.

---

# Virtualization I/O Overhead

A normal virtual network path can involve:

```text
Physical NIC
    ↓
Host driver / virtualization stack
    ↓
vSwitch
    ↓
vNIC
    ↓
Guest OS
```

This introduces processing overhead compared with direct physical-device access.

For ordinary enterprise workloads, standard virtual I/O is usually desirable because it preserves:

```text
Mobility
Central policy
Resource sharing
Operational flexibility
```

High-performance workloads may use acceleration technologies such as:

```text
SR-IOV
PCI passthrough
DPDK-based forwarding
```

Support depends on the NIC, hypervisor, guest OS, and platform.

---

## SR-IOV

**Single Root I/O Virtualization (SR-IOV)** allows one compatible physical PCIe device to expose multiple virtual functions that can be assigned to VMs.

Conceptually:

```text
Physical NIC
   |
   +-- VF1 → VM1
   +-- VF2 → VM2
   +-- VF3 → VM3
```

Benefits:

```text
Lower virtualization overhead
Higher packet throughput
Lower latency
```

Trade-off:

```text
Some virtual-switch visibility/features and migration flexibility may be reduced
```

Exact limitations are platform-specific.

---

## PCI Passthrough

PCI passthrough assigns a physical PCI device directly to a VM.

```text
Physical NIC
     ↓
Direct assignment
     ↓
VM
```

Benefits:

```text
High performance
Low latency
Direct device access
```

Trade-offs:

```text
Device dedicated to one VM
Reduced resource sharing
Migration/HA flexibility may be limited
```

---

# VM vs Container

A VM virtualizes a complete machine environment:

```text
VM
= Guest OS
+ Virtual hardware
+ Applications
```

A container normally shares the host OS kernel:

```text
Container
= Application
+ Dependencies
+ Isolated process environment
```

Therefore:

```text
VM
= Separate guest OS

Container
= Shared kernel
```

> A container is not simply a "small VM." The isolation boundary and operating model are different.

---

## 3. Why and When It Is Used

Server virtualization is useful when multiple workloads can safely and efficiently share physical infrastructure.

Common use cases include:

```text
Enterprise data centers
Private clouds
Public-cloud IaaS
Development/test environments
Virtual network appliances
Disaster recovery
Legacy workload consolidation
Application isolation
```

It provides practical benefits such as:

```text
Higher hardware utilization
Rapid VM provisioning
Workload portability
Hardware consolidation
Simplified maintenance
Snapshot/clone capability
Cluster-based HA
```

Server virtualization may be less appropriate when a workload requires:

```text
Dedicated specialized hardware
Strict physical isolation
Extremely deterministic latency
Direct hardware access
Vendor support only on bare metal
Performance that cannot tolerate virtualization overhead
```

Even in those cases, technologies such as passthrough or SR-IOV may allow virtualization while preserving more direct hardware access.

---

## 4. Key Configuration, Parameters, or CLI

> **Hypervisor platforms:** VMware vSphere/ESXi, Microsoft Hyper-V, KVM, and other virtualization systems use different configuration models and CLIs. There is no universal server-virtualization CLI.

The most important parameters to validate are:

### VM Resources

```text
vCPU count
Memory allocation
Virtual-disk size/type
vNIC count/type
Boot/firmware settings
CPU/memory reservations or limits
```

---

### Virtual Networking

```text
vSwitch / distributed switch
Port group / virtual network
VLAN ID
vNIC attachment
pNIC/uplink assignment
NIC teaming/failover policy
MTU
Security policy
```

---

### Storage

```text
Datastore / storage pool
Available capacity
Thin/thick allocation
IOPS/latency
Replication
Snapshot state
```

---

### Cluster / Availability

```text
Host membership
Migration network
Shared or replicated storage
HA policy
Resource scheduler
Admission control
Destination-host compatibility
```

---

## Network-Side Verification

> **Platform: Cisco IOS / IOS XE or NX-OS physical switching.**

Useful network commands depend on the physical design.

### MAC Learning

```cisco
show mac address-table
```

Use it to verify where VM MAC addresses are learned.

---

### VLAN / Trunk Verification

```cisco
show interfaces trunk
show vlan brief
```

On NX-OS, equivalent VLAN/trunk verification uses NX-OS syntax appropriate to the interface and release.

---

### EtherChannel / LACP

```cisco
show etherchannel summary
show lacp neighbor
```

Useful when hypervisor pNICs connect through a supported Port-Channel/LACP design.

> Do not configure an EtherChannel merely because a host has multiple uplinks. Hypervisor teaming modes and physical-switch configuration must be explicitly compatible.

---

### Interface Health

```cisco
show interfaces <interface>
```

Check:

```text
Errors
Drops
Utilization
Speed/duplex
MTU
Link state
```

---

## Practical Troubleshooting Sequence

```text
1. Is the VM actually powered on and healthy?
        ↓
2. Does it have the expected vCPU/memory/storage resources?
        ↓
3. Is the vNIC connected?
        ↓
4. Is it attached to the correct virtual network/VLAN?
        ↓
5. Is the vSwitch/uplink operational?
        ↓
6. Is the VLAN allowed on the physical trunk?
        ↓
7. Is the VM MAC learned on the expected physical port?
        ↓
8. Is Layer 3 addressing/routing correct?
        ↓
9. Is a virtual or physical security policy blocking traffic?
        ↓
10. Is host CPU/memory/storage/network contention causing performance loss?
```

For a migrated VM, also verify:

```text
MAC moved to expected host uplink
Destination host has equivalent network policy
VLAN/overlay exists on destination host
Security policy followed the VM
Return path remains valid
```

---

## 5. Common Gotchas and Misconceptions

### A VM Has Dedicated Physical Hardware

**Usually incorrect.**

A VM normally has dedicated **virtual** resources backed by shared physical resources.

```text
4 vCPU
≠
4 exclusively dedicated physical CPU cores
```

unless explicit resource reservation/pinning is configured.

---

### More vCPUs Always Improve Performance

**Incorrect.**

Oversized VMs can suffer scheduling delays and waste resources.

Allocate CPU based on measured workload demand.

---

### Virtual NIC Speed Equals Guaranteed Physical Bandwidth

**Incorrect.**

Multiple VM vNICs ultimately share:

```text
Host pNIC capacity
Switch uplinks
Storage/network paths
```

A VM may display a high virtual-link speed without having that amount of dedicated physical bandwidth.

---

### Same-Host VM Traffic Must Cross the Physical Switch

**Incorrect.**

A vSwitch can forward same-host VM traffic internally.

This can bypass physical:

```text
Switch counters
SPAN sessions
Firewalls
IDS/IPS sensors
```

unless the design deliberately inserts them.

---

### A VM Snapshot Is a Backup

**Incorrect or Unsafe.**

Snapshots are normally dependent on the original virtualization/storage environment.

Use independent backups with tested restoration.

---

### HA Means No Application Downtime

**Incorrect.**

Hypervisor HA can restart a VM after host failure, but the guest OS and application may still require recovery time.

Application-level HA is a separate design problem.

---

### Live Migration Always Requires Layer 2 Stretch

**Incorrect.**

Some architectures use L2 adjacency preservation, while others use overlays, routed mobility, or platform-specific mechanisms.

Verify the virtualization/network architecture rather than assuming VLAN stretch is mandatory.

---

### Adding Multiple pNICs Automatically Creates Redundancy

**Incorrect.**

Redundancy depends on:

```text
Hypervisor teaming policy
Physical-switch topology
LACP/static configuration
VLAN consistency
Failure detection
```

An incompatible teaming/Port-Channel design can create packet loss or loops.

---

### Virtualization Automatically Provides Security Isolation

**Incorrect.**

VM isolation reduces some cross-workload risk, but the hypervisor, management plane, virtual network, shared storage, and credentials become high-value security boundaries.

---

### Containers and VMs Are Equivalent

**Incorrect.**

VMs have separate guest operating systems. Containers normally share a host kernel.

Their isolation, startup, resource use, and security models differ.

---

## 6. Trade-Offs

### Best Practice

- Size VMs from measured CPU, memory, storage, and network requirements.
- Keep hypervisor management networks isolated from ordinary workload access.
- Provide redundant physical network paths for production hosts.
- Keep VLAN, MTU, security, and virtual-network policy consistent across migration targets.
- Monitor both **VM metrics and physical-host contention**.
- Use tested backups independently of VM snapshots.
- Design application HA separately from hypervisor HA.
- Use distributed/centrally managed virtual networking where it materially improves policy consistency.

---

### Context-Dependent Trade-Off — Consolidation Density

```text
More VMs per host
+ Better hardware utilization
+ Lower hardware footprint
- Larger host-failure blast radius
- More resource contention
```

The correct density depends on workload behavior, cluster capacity, and failure-domain requirements.

---

### Context-Dependent Trade-Off — Resource Overcommit

```text
CPU/memory overcommit
+ Higher consolidation efficiency
+ Better utilization for bursty workloads
- Performance risk during simultaneous demand
```

Overcommitment is appropriate only when monitoring and capacity planning show that shared demand remains within acceptable limits.

---

### Context-Dependent Trade-Off — Standard vSwitch vs Direct I/O

**Virtual switching**

```text
+ Flexible
+ Easy migration
+ Centralized policy
+ Resource sharing
- More processing overhead
```

**SR-IOV / PCI passthrough**

```text
+ Higher throughput
+ Lower latency
- Reduced mobility/flexibility
- Hardware dependency
- May bypass some virtual-switch services
```

Use direct I/O only when the performance requirement justifies the operational trade-off.

---

### Context-Dependent Trade-Off — Virtualized vs Bare Metal

**Virtualized**

```text
+ Consolidation
+ Fast provisioning
+ Mobility
+ Snapshot/clone/HA capabilities
- Shared-resource contention
- Virtualization overhead
- Larger hypervisor failure domain
```

**Bare Metal**

```text
+ Direct hardware access
+ Predictable performance
+ Strong physical workload separation
- Lower consolidation
- Slower hardware lifecycle/provisioning
```

---

### Incorrect or Unsafe

- Heavily overcommitting hosts without capacity monitoring.
- Treating snapshots as the only recovery mechanism.
- Running production virtualization management interfaces on untrusted user networks.
- Migrating VMs without validating destination VLAN, MTU, security, and storage dependencies.
- Assuming virtual switching traffic is visible to physical security tools.
- Building all critical VMs on one host or one physical failure domain without recovery capacity.

---

## Quick Reference

```text
Server Virtualization
= Multiple independent virtual servers on shared physical hardware

VM
= Guest OS + virtual hardware

Hypervisor
= Creates/manages VMs and allocates physical resources

Type 1
= Hypervisor directly on hardware

Type 2
= Hypervisor runs on a host OS

vCPU
= Virtual CPU scheduled on physical CPU resources

vNIC
= VM's virtual network interface

vSwitch
= Software Layer 2 switch inside virtualization host

pNIC
= Physical host network interface

Same-Host VM Traffic
= Can remain entirely inside vSwitch

Off-Host Traffic
= vNIC → vSwitch → pNIC → physical network

Live Migration
= Move running VM between hosts

Hypervisor HA
= Restart/recover VM after host failure
≠ Application-level HA

Snapshot
= Short-term state/rollback mechanism
≠ Backup

Overcommit
= Allocate more virtual capacity than physical dedicated capacity

SR-IOV
= Hardware-assisted virtual functions for lower-overhead I/O

PCI Passthrough
= Direct physical PCI device assignment to one VM

Core Rule
= Virtualization abstracts hardware;
  physical CPU, memory, storage, and network capacity still define the real limits.
```

</div>
