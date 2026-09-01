<div style="font-family: Inter, 'Segoe UI', Arial, sans-serif; line-height: 1.6;">

# Dynamic Host Configuration Protocol (DHCP)

> **Core idea:** DHCP automatically gives hosts the IP configuration they need to communicate. In IPv4, a client normally obtains a leased address and options such as subnet mask, default gateway, and DNS servers through the **DORA** exchange; a relay is required when the server is on another subnet.

---

## 1. What It Is

**DHCP (Dynamic Host Configuration Protocol)** is a client/server protocol that dynamically assigns IP configuration to endpoints and tracks address leases.

For IPv4, DHCP commonly provides:

```text
IPv4 address
Subnet mask
Default gateway
DNS server(s)
Lease time
Other optional parameters
```

DHCPv4 uses:

```text
UDP 67 → Server
UDP 68 → Client
```

> DHCP automates host configuration; it does **not** provide routing, DNS resolution itself, or access-control policy.

---

## 2. How It Works

## DHCPv4 Lease Process — DORA

A new DHCPv4 client usually begins without a usable IPv4 address, so the initial exchange relies heavily on local broadcast traffic.

```text
Client                                  Server

DHCPDISCOVER  ------------------------->
             <------------------- DHCPOFFER
DHCPREQUEST   ------------------------->
             <--------------------- DHCPACK
```

Memory:

```text
D = Discover
O = Offer
R = Request
A = Acknowledge
```

---

### 1. DHCPDISCOVER

The client searches for DHCP servers.

A common initial packet uses:

```text
Source IP:      0.0.0.0
Destination IP: 255.255.255.255
Source UDP:     68
Destination UDP:67
```

```text
Client
  |
  | DHCPDISCOVER
  | Broadcast
  v
Local VLAN / Broadcast Domain
```

Because `255.255.255.255` is a local broadcast, routers do not forward it normally.

---

### 2. DHCPOFFER

A DHCP server can offer:

```text
Proposed IP address
Subnet mask
Default gateway
DNS server(s)
Lease duration
Other DHCP options
```

Multiple servers may send offers.

The server's response can be broadcast or unicast depending on the client's state, flags, and implementation behavior.

---

### 3. DHCPREQUEST

The client selects an offer and requests that lease.

During the initial allocation process, the DHCPREQUEST is commonly broadcast so that:

```text
Selected server
= Knows its offer was accepted

Other servers
= Know their offers were not selected
```

The message identifies the requested address and selected server.

---

### 4. DHCPACK

The selected server confirms the lease with a DHCPACK.

The client can then apply the assigned configuration.

```text
DHCPACK
   ↓
Client installs:
IP address
Subnet mask
Default gateway
DNS
Lease timers
```

A server can instead send **DHCPNAK** when the requested address is no longer valid for that client or subnet.

---

## DHCP Client State and Lease Renewal

DHCP addresses are normally leased, not permanently assigned.

Important client states:

```text
INIT
  ↓
SELECTING
  ↓
REQUESTING
  ↓
BOUND
  ↓
RENEWING
  ↓
REBINDING
```

### BOUND

The client has a valid lease and uses the address normally.

---

### T1 — Renewal

By default, T1 is typically:

```text
50% of the lease time
```

At T1, the client enters **RENEWING** and normally sends a unicast DHCPREQUEST to the server that granted the lease.

```text
Lease starts
     |
     |------ 50% ------|
                      T1
                      ↓
             Unicast renewal attempt
```

The server may override the default timer through DHCP options.

---

### T2 — Rebinding

If renewal fails, the client reaches T2, typically:

```text
87.5% of the lease time
```

The client enters **REBINDING** and attempts to renew with any available DHCP server, generally using broadcast.

```text
T1                    T2                Expiration
|---------------------|---------------------|
 Unicast attempts      Broadcast attempts
```

If the lease expires without successful renewal, the client must stop using the leased address.

> Existing clients can continue operating during a temporary DHCP-server outage as long as their current leases remain valid.

---

## Other Useful DHCPv4 Messages

| Message | Purpose |
|---|---|
| **DHCPNAK** | Server rejects an invalid requested lease |
| **DHCPDECLINE** | Client reports that the offered address appears to be in use |
| **DHCPRELEASE** | Client voluntarily returns its lease |
| **DHCPINFORM** | Client already has an address but requests other DHCP options |

These messages are operationally useful but are not part of the basic DORA sequence.

---

## DHCP Relay

A DHCP server is often centralized in another subnet.

Problem:

```text
Client DHCP broadcast
        ↓
Router
        X
Broadcast is not routed
```

A **DHCP relay agent** solves this.

```text
Client VLAN
   |
   | DHCP broadcast
   v
Relay Agent
   |
   | Unicast
   v
DHCP Server
```

The relay receives the client broadcast and forwards the DHCP message toward the server as routable unicast traffic.

The relay provides information identifying the client-facing subnet so the server can choose the correct address pool. In DHCPv4, the **giaddr** field is central to this behavior; relay implementations may also use **Option 82** to carry additional relay/circuit information.

Return traffic follows the reverse path through the relay back to the client.

---

## DHCP Server Decision Logic

A DHCP server maintains state for leases.

Conceptually:

```text
DHCP request arrives
       ↓
Identify client subnet
       ↓
Select matching scope / pool
       ↓
Find available address
       ↓
Apply reservation/policy if configured
       ↓
Offer address + options
       ↓
Record lease if accepted
```

The server must have an address pool that corresponds to the client's subnet.

Common scope information includes:

```text
Network / subnet
Available address range
Excluded addresses
Default gateway
DNS servers
Lease duration
Reservations
```

---

## Address Allocation Models

### Dynamic Allocation

```text
Address selected from pool
        ↓
Leased for a finite time
        ↓
Returned to pool after expiration/release
```

This is the normal enterprise client model.

---

### Reservation / Static Allocation

A specific address is consistently assigned to a specific client identifier or MAC-based reservation according to the DHCP server implementation.

Useful when an endpoint needs predictable addressing but centralized DHCP management is still desired.

---

## DHCPv6 — Important Differences

DHCPv6 is related to DHCPv4 but is **not the same protocol exchange**.

DHCPv6 uses:

```text
UDP 546 → Client
UDP 547 → Server
```

A normal stateful exchange is:

```text
Client                                  Server

SOLICIT     --------------------------->
            <------------------- ADVERTISE
REQUEST     --------------------------->
            <----------------------- REPLY
```

DHCPv6 uses IPv6 multicast rather than IPv4 broadcast. Clients commonly send Solicit messages to:

```text
FF02::1:2
```

Key difference:

> **DHCPv6 does not supply the IPv6 default gateway.**

IPv6 hosts learn default-router information through **ICMPv6 Router Advertisements (RA)**.

DHCPv6 may operate as:

```text
Stateful
= Server assigns IPv6 addressing information

Stateless
= SLAAC provides addressing; DHCPv6 supplies selected options such as DNS
```

A DHCPv6 relay is required when the DHCPv6 server is not reachable on the client's local link.

---

## 3. Why and When It Is Used

DHCP is appropriate when large numbers of endpoints require centrally managed and automatically assigned IP configuration.

Typical environments:

```text
User LANs
Wireless networks
Voice endpoints
Branch offices
Guest networks
Data-center or cloud workloads where dynamic addressing is appropriate
```

DHCP solves:

- Manual address-assignment overhead
- Duplicate-address risk caused by manual configuration
- Centralized distribution of gateway, DNS, and other options
- Reuse of addresses through leasing
- Consistent client configuration across many subnets

DHCP is usually unnecessary for infrastructure addresses that must remain predictably reachable regardless of DHCP availability, such as many routers, switches, firewalls, servers, or network-management interfaces. Whether those devices use static addressing or DHCP reservations is a design decision.

---

## 4. Key Configuration, Parameters, or CLI

> **Platform:** Cisco IOS / IOS XE.  
> Exact DHCP feature support can vary by platform and release.

---

## Cisco IOS / IOS XE — DHCP Relay

Configure the relay on the interface facing the **DHCP clients**:

```cisco
interface Vlan10
 ip address 172.16.10.1 255.255.255.0
 ip helper-address 172.16.100.50
```

Conceptually:

```text
VLAN 10 clients
      ↓
Vlan10 SVI
ip helper-address 172.16.100.50
      ↓
DHCP Server
172.16.100.50
```

Verify:

```cisco
show ip interface Vlan10
show running-config interface Vlan10
```

> `ip helper-address` is required only when the DHCPv4 server is on another subnet.

---

## Cisco IOS / IOS XE — DHCP Server

Exclude addresses that must not be leased:

```cisco
ip dhcp excluded-address 172.16.10.1 172.16.10.20
```

Create the pool:

```cisco
ip dhcp pool USERS
 network 172.16.10.0 255.255.255.0
 default-router 172.16.10.1
 dns-server 10.10.10.53
 lease 7
```

Verify:

```cisco
show ip dhcp pool
show ip dhcp binding
show ip dhcp conflict
```

The IOS/IOS XE DHCP server is useful for labs, branches, and some production designs, but large enterprises often use dedicated centralized DHCP services.

---

## Cisco IOS / IOS XE — DHCP Client

Configure an interface to obtain an IPv4 address through DHCP:

```cisco
interface GigabitEthernet0/0
 ip address dhcp
 no shutdown
```

Verify:

```cisco
show ip interface brief
show dhcp lease
```

Command availability/output can vary by IOS/IOS XE platform and release; verify with the platform command reference if needed.

---

## Cisco IOS / IOS XE — DHCPv6 Relay

Example:

```cisco
interface GigabitEthernet0/0
 ipv6 dhcp relay destination 2001:db8:100::50
```

Verify IPv6 interface behavior:

```cisco
show ipv6 interface GigabitEthernet0/0
```

---

## Cisco Catalyst IOS / IOS XE — DHCP Snooping

DHCP snooping protects the access layer against rogue DHCP servers and certain DHCP-based attacks.

Enable globally and for the required VLAN:

```cisco
ip dhcp snooping
ip dhcp snooping vlan 10
```

Trust only the path toward the legitimate DHCP server or relay:

```cisco
interface GigabitEthernet1/0/48
 ip dhcp snooping trust
```

Verify:

```cisco
show ip dhcp snooping
show ip dhcp snooping binding
```

> Access ports facing normal clients are typically left **untrusted**.

Option 82 behavior and DHCP-snooping defaults can vary by topology and platform. Validate server/relay compatibility before enabling DHCP snooping in production.

---

## Practical Troubleshooting Sequence

```text
1. Does the client reach the local VLAN correctly?
        ↓
2. Does DHCPDISCOVER leave the client?
        ↓
3. Is the server local?
       / \
     Yes  No
      |    |
      |   Is relay/helper configured?
      |            ↓
      +------------+
                   ↓
4. Can relay and server route to each other?
        ↓
5. Does the server have a pool for that client subnet?
        ↓
6. Is the pool exhausted?
        ↓
7. Does DHCPOFFER return?
        ↓
8. Does DHCPREQUEST / DHCPACK complete?
        ↓
9. Are gateway, mask, DNS, and lease options correct?
```

Useful IOS/IOS XE checks:

```cisco
show ip interface <client-facing-interface>
show ip dhcp pool
show ip dhcp binding
show ip dhcp conflict
show ip dhcp snooping
show ip dhcp snooping binding
```

Packet capture is often the fastest way to prove where DORA stops.

---

## 5. Common Gotchas and Misconceptions

### DHCPDISCOVER Crosses Routers Automatically

**Incorrect.**

The initial DHCPv4 client traffic is normally local broadcast traffic.

```text
Client broadcast
      ↓
Router
      X
```

Use a DHCP relay when the server is remote.

---

### `ip helper-address` Belongs on the Server-Facing Interface

**Incorrect.**

Configure it on the Layer 3 interface that receives the client broadcasts:

```text
Client VLAN
    ↓
Client-facing SVI/router interface
    ↓
ip helper-address
```

---

### DORA Is Used for Every DHCP Packet

**Incorrect.**

DORA describes the normal initial DHCPv4 lease process.

Lease renewals normally use DHCPREQUEST/DHCPACK directly rather than restarting with DHCPDISCOVER.

---

### A Client That Already Has a Lease Immediately Fails When the DHCP Server Goes Down

**Incorrect.**

The client can continue using its current address while the lease remains valid.

New clients and clients approaching lease expiration are affected first.

---

### A DHCP Lease Means the Client Has Correct Connectivity

**Incorrect.**

A server can lease a valid address while providing incorrect options.

Examples:

```text
Wrong default gateway
→ Local subnet works, remote networks fail

Wrong DNS
→ IP connectivity works, name resolution fails

Wrong subnet mask
→ Local/remote reachability behaves incorrectly
```

---

### DHCP Provides the IPv6 Default Gateway

**Incorrect.**

DHCPv6 does not provide the IPv6 default router.

```text
Default gateway
= Learned through Router Advertisement

DHCPv6
= Addressing and/or other options
```

---

### DHCP Snooping Trust Should Be Enabled on User Ports

**Incorrect or Unsafe.**

Client-facing access ports should normally remain untrusted.

Only infrastructure paths where legitimate DHCP server messages must enter should be trusted.

---

### DHCP Snooping Can Be Enabled Without Considering Option 82

**Incorrect or risky.**

Some Catalyst designs insert DHCP Option 82 when snooping is enabled. Servers, relay agents, and switches must agree on how that information is handled.

Verify behavior before production rollout.

---

### `ip helper-address` Is Only a DHCP Command

**Misconception.**

On IOS/IOS XE, `ip helper-address` is a UDP broadcast-relay mechanism and can forward multiple well-known UDP services by default, not only DHCP/BOOTP.

If the design requires DHCP relay only, review the platform's `ip forward-protocol` behavior and security implications.

---

## 6. Trade-Offs

### Best Practice

- Centralize DHCP where operationally appropriate and use relay agents for remote VLANs.
- Maintain redundant DHCP services for critical endpoint environments.
- Exclude infrastructure/static addresses from dynamic pools.
- Use DHCP snooping at the access layer when the switching platform and design support it.
- Trust only legitimate DHCP-server/relay paths.
- Validate lease capacity and expiration behavior before large migrations.

---

### Context-Dependent Trade-Off — Centralized vs Local DHCP

**Centralized DHCP**

```text
+ Centralized administration
+ Consistent options/policies
+ Easier address management
- Depends on routed reachability and relay
- Central service availability becomes important
```

**Local DHCP**

```text
+ Can continue serving clients during WAN isolation
+ Simpler local packet path
- More distributed configuration/state
- Harder to manage consistently at scale
```

The correct model depends on WAN resilience, site autonomy, operational tooling, and DHCP-server architecture.

---

### Context-Dependent Trade-Off — Static Address vs DHCP Reservation

**Static address**

```text
+ Independent of DHCP
+ Predictable local configuration
- Manual lifecycle management
```

**DHCP reservation**

```text
+ Predictable address
+ Centralized management
- Depends on DHCP service and correct client identity
```

Use reservations when centralized lifecycle management is valuable and DHCP dependency is acceptable.

---

### Incorrect or Unsafe

- Deploying overlapping DHCP pools without coordination.
- Trusting endpoint-facing ports for DHCP snooping.
- Enabling DHCP snooping without validating Option 82 and relay/server behavior.
- Using very short lease times without considering server load and outage tolerance.
- Treating DHCP success as proof that routing, DNS, or application connectivity is healthy.

---

## Quick Reference

```text
DHCP
= Dynamic host IP configuration

DHCPv4 UDP
= Server 67
= Client 68

DORA
= Discover
= Offer
= Request
= Acknowledge

Initial DHCPv4 Client
= 0.0.0.0 → 255.255.255.255

DHCP Relay
= Converts local client broadcast into routable server communication

IOS / IOS XE Relay
= ip helper-address <server-ip>

Lease
= Temporary right to use an address

T1
= Renewal
= Typically 50%

T2
= Rebinding
= Typically 87.5%

DHCP Snooping
= Filters unauthorized/invalid DHCP messages
= Builds IP/MAC/VLAN/interface bindings

DHCPv6 UDP
= Client 546
= Server 547

DHCPv6 Initial Flow
= Solicit → Advertise → Request → Reply

IPv6 Default Gateway
= Router Advertisement, not DHCPv6

DHCP
≠ Routing
≠ DNS itself
≠ Security policy
```

## CCNA Configuration

**CCNA 200-301 v2.0 — IOS-XE DHCPv4 Server**

| Command | Description |
|---|---|
| **Exclude one address:**<br>`(config)#ip dhcp excluded-address <ip-address>` | Excludes one IPv4 address from DHCP allocation. |
| **Exclude address range:**<br>`(config)#ip dhcp excluded-address <low-address> <high-address>` | Excludes an IPv4 address range from DHCP allocation. |
| **Create DHCP pool:**<br>`(config)#ip dhcp pool <pool-name>`<br>&nbsp;&nbsp;○ `(dhcp-config)#network <network-number> <subnet-mask>` | Creates a pool and defines its client subnet. |
| **Use prefix length:**<br>`(config)#ip dhcp pool <pool-name>`<br>&nbsp;&nbsp;○ `(dhcp-config)#network <network-number> /<prefix-length>` | Defines the pool subnet using CIDR prefix length. |
| **Set default gateway:**<br>`(config)#ip dhcp pool <pool-name>`<br>&nbsp;&nbsp;○ `(dhcp-config)#default-router <ip-address> [<ip-address2> ...]` | Supplies one or more default-router addresses to clients. |
| **Set DNS servers:**<br>`(config)#ip dhcp pool <pool-name>`<br>&nbsp;&nbsp;○ `(dhcp-config)#dns-server <ip-address> [<ip-address2> ...]` | Supplies DNS server addresses to DHCP clients. |
| **Set domain name:**<br>`(config)#ip dhcp pool <pool-name>`<br>&nbsp;&nbsp;○ `(dhcp-config)#domain-name <domain-name>` | Supplies the DNS domain name to clients. |
| **Set lease duration:**<br>`(config)#ip dhcp pool <pool-name>`<br>&nbsp;&nbsp;○ `(dhcp-config)#lease <days> [<hours> [<minutes>]]` | Sets the DHCP lease duration. |
| **Set infinite lease:**<br>`(config)#ip dhcp pool <pool-name>`<br>&nbsp;&nbsp;○ `(dhcp-config)#lease infinite` | Configures leases without expiration. |
| **Set next server:**<br>`(config)#ip dhcp pool <pool-name>`<br>&nbsp;&nbsp;○ `(dhcp-config)#next-server <ip-address> [<ip-address2> ...]` | Supplies next-server addresses to DHCP clients. |
| **Show DHCP bindings:**<br>`#show ip dhcp binding` | Displays active DHCP server bindings. |
| **Show DHCP pool:**<br>`#show ip dhcp pool [<pool-name>]` | Displays pool ranges, utilization, and lease counts. |
| **Show server statistics:**<br>`#show ip dhcp server statistics` | Displays DHCP server message and binding statistics. |
| **Show address conflicts:**<br>`#show ip dhcp conflict` | Displays addresses marked as DHCP conflicts. |

**CCNA 200-301 v2.0 — IOS-XE DHCPv4 Relay**

| Command | Description |
|---|---|
| **Configure DHCP relay:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip helper-address <dhcp-server-ip>` | Relays supported UDP broadcasts to the specified server. |
| **Add another DHCP server:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip helper-address <additional-server-ip>` | Adds another helper destination on the client-facing interface. |
| **Remove DHCP relay:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no ip helper-address <dhcp-server-ip>` | Removes the specified helper destination. |
| **Verify helper address:**<br>`#show ip interface <interface-id>` | Displays configured helper addresses on the interface. |
| **Verify relay configuration:**<br>`#show running-config interface <interface-id>` | Displays helper commands configured under the interface. |

**CCNA 200-301 v2.0 — IOS-XE DHCPv4 Client**

| Command | Description |
|---|---|
| **Configure routed-interface client:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip address dhcp` | Obtains the interface IPv4 configuration through DHCP. |
| **Configure SVI client:**<br>`(config)#interface vlan <vlan-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip address dhcp` | Obtains the SVI IPv4 configuration through DHCP. |
| **Show DHCP lease:**<br>`#show dhcp lease` | Displays DHCP-learned lease information. |
| **Verify interface address:**<br>`#show ip interface brief` | Displays assigned interface IPv4 addresses and status. |
| **Verify SVI address:**<br>`#show interfaces vlan <vlan-id>` | Displays the SVI address and operational state. |

**CCNA 200-301 v2.0 — IOS-XE Catalyst DHCP Snooping**

| Command | Description |
|---|---|
| **Enable DHCP snooping:**<br>`(config)#ip dhcp snooping` | Enables DHCP snooping globally. |
| **Enable snooping on VLANs:**<br>`(config)#ip dhcp snooping vlan <vlan-list>` | Enables DHCP snooping on specified VLANs. |
| **Trust infrastructure interface:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip dhcp snooping trust` | Marks the interface trusted for DHCP server messages. |
| **Restore untrusted state:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no ip dhcp snooping trust` | Restores the default untrusted snooping state. |
| **Disable Option 82 insertion:**<br>`(config)#no ip dhcp snooping information option` | Disables DHCP snooping Option 82 insertion. |
| **Enable Option 82 insertion:**<br>`(config)#ip dhcp snooping information option` | Enables DHCP snooping Option 82 insertion. |
| **Set DHCP rate limit:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip dhcp snooping limit rate <pps>` | Limits received DHCP messages per second. |
| **Remove DHCP rate limit:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#no ip dhcp snooping limit rate` | Removes the configured DHCP snooping rate limit. |
| **Enable errdisable recovery:**<br>`(config)#errdisable recovery cause dhcp-rate-limit` | Enables automatic recovery after DHCP rate-limit errdisable. |
| **Set recovery interval:**<br>`(config)#errdisable recovery interval <seconds>` | Sets the global errdisable recovery interval. |
| **Show snooping status:**<br>`#show ip dhcp snooping` | Displays DHCP snooping configuration and interface trust states. |
| **Show snooping bindings:**<br>`#show ip dhcp snooping binding` | Displays dynamically learned DHCP snooping bindings. |
| **Show snooping statistics:**<br>`#show ip dhcp snooping statistics` | Displays DHCP snooping packet counters. |
| **Show errdisable recovery:**<br>`#show errdisable recovery` | Displays errdisable recovery causes and timers. |

## CCNP Configuration

**CCNP Security — SCOR 350-701 v2.0 — IOS-XE Catalyst DHCP Snooping Persistence**

| Command | Description |
|---|---|
| **Configure TFTP binding database:**<br>`(config)#ip dhcp snooping database tftp://<server>/<filename>` | Stores DHCP snooping bindings on a TFTP destination. |
| **Configure flash binding database:**<br>`(config)#ip dhcp snooping database flash:/<filename>` | Stores DHCP snooping bindings in local flash. |
| **Set database timeout:**<br>`(config)#ip dhcp snooping database timeout <seconds>` | Sets the binding database transfer timeout. |
| **Set database write delay:**<br>`(config)#ip dhcp snooping database write-delay <seconds>` | Sets delay before writing changed snooping bindings. |
| **Verify binding database:**<br>`#show ip dhcp snooping database [detail]` | Displays binding database agent status and statistics. |
| **Verify source bindings:**<br>`#show ip source binding` | Displays dynamically and statically learned source bindings. |

**CCNP Security — SCOR 350-701 v2.0 — IOS-XE Catalyst DHCP Snooping Validation**

| Command | Description |
|---|---|
| **Enable MAC verification:**<br>`(config)#ip dhcp snooping verify mac-address` | Verifies client hardware address against source MAC address. |
| **Disable MAC verification:**<br>`(config)#no ip dhcp snooping verify mac-address` | Disables DHCP snooping source-MAC verification. |
| **Allow untrusted Option 82:**<br>`(config)#ip dhcp snooping information option allow-untrusted` | Accepts Option 82 packets arriving on untrusted interfaces. |
| **Verify snooping state:**<br>`#show ip dhcp snooping` | Displays Option 82, verification, trust, and VLAN state. |


</div>
