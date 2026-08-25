# Address Resolution Protocol (ARP)

## 1. What Is ARP?

**ARP (Address Resolution Protocol)** is the mechanism IPv4 devices use to discover the **Layer 2 MAC address associated with an IPv4 address on the local network segment**.

The fundamental mapping is:

```text
IPv4 address
     ↓ ARP
MAC address
```

A router, multilayer switch, or host needs this mapping because IP and Ethernet perform different jobs:

```text
Layer 3 — IPv4
Identifies the logical source and destination.

Layer 2 — Ethernet
Actually delivers the frame across the local Ethernet segment.
```

Cisco describes the ARP table as the mechanism used to map Layer 3 IP addresses to Layer 2 MAC addresses so the device can construct the correct Layer 2 header before forwarding traffic.

A useful way to remember ARP is:

> **Routing determines the next-hop IP address. ARP determines the MAC address needed to reach that next hop on Ethernet.**

---

# 2. Why ARP Is Necessary

Suppose PC1 wants to communicate with PC2:

```text
PC1
IP:  172.16.10.10
MAC: AAAA.AAAA.AAAA

        |
        | Ethernet LAN
        |

PC2
IP:  172.16.10.20
MAC: BBBB.BBBB.BBBB
```

PC1 knows:

```text
Destination IP = 172.16.10.20
```

However, to transmit the packet through Ethernet, PC1 also needs:

```text
Destination MAC = ?
```

An Ethernet frame requires both:

```text
Source MAC
Destination MAC
```

Therefore PC1 needs some way to ask:

> "Who owns 172.16.10.20, and what is your MAC address?"

That is ARP.

Cisco Press defines ARP as the method by which a host or router dynamically learns the MAC address associated with another IP device on the same LAN.

---

# 3. ARP Operates Only on the Local Layer 2 Segment

This is one of the most important ARP concepts.

An ARP request is an Ethernet broadcast:

```text
FFFF.FFFF.FFFF
```

Routers do not normally forward Layer 2 broadcasts between networks.

Therefore:

```text
ARP does NOT cross routers.
```

If you have:

```text
VLAN 10                   VLAN 20

PC1 ---- SW1 ---- Router ---- SW2 ---- PC2

172.16.10.10             172.16.20.20
```

PC1 cannot ARP directly for:

```text
172.16.20.20
```

because PC2 exists in another broadcast domain.

Instead, PC1 resolves the MAC address of its **default gateway**.

This distinction becomes extremely important when troubleshooting.

---

# 4. ARP Request and ARP Reply

Normal ARP consists primarily of two messages:

```text
ARP Request
ARP Reply
```

## ARP Request

Suppose:

```text
PC1
IP  = 172.16.10.10
MAC = AAAA.AAAA.AAAA

PC2
IP  = 172.16.10.20
MAC = BBBB.BBBB.BBBB
```

PC1 does not know PC2's MAC.

It sends approximately:

```text
ARP Request:

Sender IP:     172.16.10.10
Sender MAC:    AAAA.AAAA.AAAA
Target IP:     172.16.10.20
Target MAC:    Unknown
```

The Ethernet destination is:

```text
FFFF.FFFF.FFFF
```

So conceptually:

```text
PC1:

"Who has 172.16.10.20?
 Tell 172.16.10.10."
```

Because it is a broadcast, every device in that VLAN receives the frame.

But only the device configured with:

```text
172.16.10.20
```

should respond.

---

## ARP Reply

PC2 replies:

```text
ARP Reply:

Sender IP:     172.16.10.20
Sender MAC:    BBBB.BBBB.BBBB
Target IP:     172.16.10.10
Target MAC:    AAAA.AAAA.AAAA
```

The reply is normally sent as a **unicast** directly to PC1.

Cisco Press describes this normal sequence as:

```text
ARP Request = broadcast
ARP Reply   = unicast
```

After receiving the reply, PC1 can store:

```text
172.16.10.20 → BBBB.BBBB.BBBB
```

in its ARP cache.

---

# 5. Both Devices Can Learn During ARP

An important detail is that ARP learning does not have to be completely one-directional.

The original ARP request contains:

```text
Sender IP
Sender MAC
```

Therefore, when PC2 receives PC1's ARP request, PC2 can also learn:

```text
172.16.10.10 → AAAA.AAAA.AAAA
```

PC1 then learns PC2's mapping from the ARP reply.

Cisco Press specifically points out that the receiver can learn the sender's IP-to-MAC mapping from the sender/origin fields contained in the ARP message.

---

# 6. Complete Local-Subnet Packet Flow

Consider:

```text
PC1
172.16.10.10/24
MAC AAAA.AAAA.AAAA

PC2
172.16.10.20/24
MAC BBBB.BBBB.BBBB
```

PC1 wants to ping:

```text
172.16.10.20
```

### Step 1 — Determine whether the destination is local

PC1 applies its subnet mask:

```text
PC1:
172.16.10.10/24

Destination:
172.16.10.20
```

Both belong to:

```text
172.16.10.0/24
```

Therefore:

```text
Destination = local subnet
```

PC1 knows that it should send the Ethernet frame directly to PC2.

---

### Step 2 — Check the ARP cache

PC1 checks:

```text
Do I already know the MAC for 172.16.10.20?
```

If yes:

```text
Use cached MAC.
```

If no:

```text
Perform ARP.
```

---

### Step 3 — Broadcast ARP request

```text
Ethernet destination:
FFFF.FFFF.FFFF

ARP target:
172.16.10.20
```

The switch floods the broadcast to the other ports in VLAN 10.

---

### Step 4 — PC2 responds

PC2 recognizes:

```text
Target IP = my IP address
```

and replies:

```text
172.16.10.20 = BBBB.BBBB.BBBB
```

---

### Step 5 — PC1 updates ARP cache

PC1 now has:

```text
IP Address       MAC Address
172.16.10.20     BBBB.BBBB.BBBB
```

---

### Step 6 — PC1 transmits the original IP packet

Now the Ethernet frame can be constructed:

```text
Ethernet Header
--------------------------------
Source MAC:      AAAA.AAAA.AAAA
Destination MAC: BBBB.BBBB.BBBB

IPv4 Header
--------------------------------
Source IP:       172.16.10.10
Destination IP:  172.16.10.20
```

The packet can finally be sent.

---

# 7. ARP for a Remote Destination

This is where many networking students initially get ARP wrong.

Suppose:

```text
PC1
172.16.10.10/24

Default gateway:
172.16.10.1

Destination:
8.8.8.8
```

PC1 does **not** ARP for:

```text
8.8.8.8
```

Instead, it determines:

```text
8.8.8.8 is not in 172.16.10.0/24.
```

Therefore:

```text
Send packet to default gateway.
```

PC1 needs the MAC address corresponding to:

```text
172.16.10.1
```

So it sends:

```text
Who has 172.16.10.1?
```

Cisco Press explicitly notes that an ARP table normally does not contain entries for every remote destination. Instead, it contains the Layer 2 information for the **next hop used to reach those remote destinations**.

The resulting frame might contain:

```text
Ethernet:
Source MAC      = PC1 MAC
Destination MAC = Router MAC

IP:
Source IP       = 172.16.10.10
Destination IP  = 8.8.8.8
```

Notice something very important:

```text
Destination MAC = Router

Destination IP  = 8.8.8.8
```

Those are intentionally different.

---

# 8. MAC Addresses Change Hop by Hop — IP Addresses Usually Do Not

Consider:

```text
PC1
 |
R1
 |
R2
 |
Server
```

Traffic:

```text
PC1 → Server
```

The IP addresses remain approximately:

```text
Source IP      = PC1
Destination IP = Server
```

throughout the routed path.

But the Ethernet MAC addresses change at every Layer 3 hop.

### PC1 → R1

```text
SRC MAC = PC1
DST MAC = R1
```

### R1 → R2

R1 removes the original Ethernet header and builds a new one:

```text
SRC MAC = R1
DST MAC = R2
```

### R2 → Server

```text
SRC MAC = R2
DST MAC = Server
```

ARP supplies the Layer 2 destination needed for each directly reachable Ethernet next hop.

This is why ARP and routing work together but perform different functions.

---

# 9. ARP vs Routing Table

Do not confuse the two.

## Routing Table

Answers:

> **Where should the IP packet go?**

Example:

```text
Destination: 8.8.8.8
Next hop:    172.16.100.1
```

Verification:

```ios
show ip route 8.8.8.8
```

---

## ARP Table

Answers:

> **What Layer 2 MAC address corresponds to that directly reachable next hop?**

Example conceptually:

```text
172.16.100.1 → AAAA.BBBB.CCCC
```

Verification on IOS/IOS-XE:

```ios
show ip arp
```

Cisco ENCOR describes the forwarding relationship this way: the route provides the next-hop IP information, after which the device looks for that next hop in the ARP table to obtain the destination MAC address.

Therefore:

```text
Routing Table
     ↓
Next-Hop IP
     ↓
ARP Table
     ↓
Next-Hop MAC
     ↓
Ethernet Frame
```

---

# 10. ARP Table vs MAC Address Table

This distinction is critical.

## ARP Table

Maps:

```text
IP → MAC
```

Example command:

```ios
show ip arp
```

Used primarily by:

```text
Hosts
Routers
Layer 3 switches
```

---

## MAC Address Table / CAM Table

Maps:

```text
MAC → Switch Port
```

Example:

```text
AAAA.BBBB.CCCC → Gi1/0/10
```

Command:

```ios
show mac address-table
```

The switch uses this table to decide which Layer 2 port should receive an Ethernet frame.

So:

```text
ARP Table
IP → MAC

MAC Table
MAC → Port
```

Together:

```text
IP
 ↓ ARP
MAC
 ↓ CAM table
Switch port
```

That is a very useful troubleshooting chain.

---

# 11. What the Switch Does During ARP

Consider:

```text
PC1 ---- SW1 ---- PC2
```

PC1 sends an ARP request.

### Step 1

SW1 receives:

```text
Source MAC = PC1
Destination MAC = FFFF.FFFF.FFFF
```

The switch first learns:

```text
PC1 MAC → incoming switch port
```

---

### Step 2

Because the destination is broadcast:

```text
FFFF.FFFF.FFFF
```

SW1 floods the frame throughout the VLAN, except back out the ingress port.

---

### Step 3

PC2 replies with a unicast ARP reply.

SW1 receives that frame and learns:

```text
PC2 MAC → PC2's port
```

---

### Step 4

The switch can now unicast frames between the two devices using its MAC address table.

Therefore ARP and MAC learning frequently happen together, but they are **different processes**.

---

# 12. ARP Is VLAN-Specific

Consider:

```text
VLAN 10
PC1
 |
SW1
 |
Trunk
 |
SW2
 |
PC2
VLAN 10
```

An ARP request from PC1 can traverse the trunk because VLAN 10 exists on both switches.

However:

```text
VLAN 10 ARP broadcast
```

does not automatically enter:

```text
VLAN 20
```

A VLAN defines a Layer 2 broadcast domain.

Therefore:

```text
ARP scope = broadcast domain / VLAN
```

Routers and Layer 3 SVIs form the boundary.

---

# 13. ARP Packet Structure

ARP is not carried inside a normal IPv4 packet.

On Ethernet, ARP has its own Ethernet EtherType:

```text
0x0806
```

Typical Ethernet/IPv4 ARP fields include:

| Field                   | Purpose                                  |
| ----------------------- | ---------------------------------------- |
| Hardware Type           | Identifies underlying Layer 2 technology |
| Protocol Type           | Identifies protocol being resolved       |
| Hardware Length         | Length of MAC address                    |
| Protocol Length         | Length of protocol address               |
| Operation               | Request or Reply                         |
| Sender Hardware Address | Sender MAC                               |
| Sender Protocol Address | Sender IPv4                              |
| Target Hardware Address | Target MAC                               |
| Target Protocol Address | Target IPv4                              |

For normal Ethernet and IPv4:

```text
Hardware Type = Ethernet
Protocol Type = IPv4
Hardware Length = 6 bytes
Protocol Length = 4 bytes
Opcode 1 = Request
Opcode 2 = Reply
```

In an ARP request:

```text
Target IP  = known
Target MAC = unknown
```

That is the entire problem ARP is trying to solve.

---

# 14. ARP Cache

Devices do not want to broadcast an ARP request for every single packet.

Instead, they maintain an **ARP cache/table**.

Conceptually:

```text
172.16.10.1  → MAC-A
172.16.10.20 → MAC-B
172.16.10.50 → MAC-C
```

Once the mapping has been learned, subsequent traffic can use the cached information.

Cisco IOS/IOS-XE verification:

```ios
show ip arp
```

The ENCOR guide also documents filtering forms of the command such as:

```ios
show ip arp <ip-address>
```

and platform-supported filters for MAC, VLAN, and interface.

---

# 15. Dynamic ARP Aging

Dynamically learned ARP entries do not normally remain forever.

They eventually age out so that network changes can be relearned.

Cisco IOS commonly uses a default ARP timeout of **four hours (240 minutes)**, although timers and behavior should always be verified for the specific platform and software release. Cisco's CCNA material documents the 240-minute IOS behavior and the `clear ip arp` command for manually clearing entries.

Useful lab command:

```ios
clear ip arp
```

Or for a specific entry on supported IOS/IOS-XE releases:

```ios
clear ip arp <ip-address>
```

Be careful with this command in production: clearing ARP entries forces the router to relearn neighbors and can briefly affect forwarding.

---

# 16. ARP and Cisco Express Forwarding

At CCNP/CCIE level, ARP also needs to be understood in relation to **CEF**.

Conceptually:

```text
Routing protocols / static routes
            ↓
           RIB
            ↓
           FIB
            ↓
     Next-hop resolution
            ↓
      ARP / adjacency
            ↓
Layer 2 rewrite information
            ↓
     Hardware forwarding
```

CEF's FIB determines where the packet should go.

ARP provides the MAC information required to construct the Layer 2 rewrite toward an Ethernet next hop.

Useful verification:

```ios
show ip route <destination>
show ip cef <destination> detail
show ip arp <next-hop>
```

This is a powerful troubleshooting sequence:

```text
1. Do I have a route?
2. What next hop does CEF select?
3. Can I resolve that next hop to a MAC?
```

---

# 17. Gratuitous ARP

A **Gratuitous ARP (GARP)** is an ARP message sent without first receiving a normal ARP request.

Its purpose is essentially:

> "Everyone on this LAN should know that this IP address is associated with this MAC address."

Cisco Press presents gratuitous ARP as an unsolicited ARP reply sent to the broadcast MAC address so other devices can update their ARP information.

Legitimate uses include:

```text
IP/MAC changes
High-availability failover
Duplicate-address detection mechanisms
Updating stale neighbor information
Virtual machine movement
Gateway redundancy
```

A GARP can quickly tell the rest of the broadcast domain:

```text
172.16.10.1 is now reachable using MAC AAAA.BBBB.CCCC
```

---

# 18. ARP and HSRP

ARP is particularly important when working with HSRP.

Suppose:

```text
DIST-SW1 = 172.16.10.2
DIST-SW2 = 172.16.10.3

HSRP VIP = 172.16.10.1
```

Clients use:

```text
Default gateway = 172.16.10.1
```

The client sends:

```text
Who has 172.16.10.1?
```

The HSRP Active router responds using the **HSRP virtual MAC**, not simply its physical interface MAC.

For HSRPv1 group 10, for example:

```text
0000.0c07.ac0a
```

Therefore the client learns conceptually:

```text
172.16.10.1 → 0000.0c07.ac0a
```

The major advantage is that after HSRP failover, the new Active device can take ownership of the same virtual IP and virtual MAC.

Conceptually:

```text
Before:

172.16.10.1
      ↓
HSRP Virtual MAC
      ↓
DIST-SW1


After failover:

172.16.10.1
      ↓
Same HSRP Virtual MAC
      ↓
DIST-SW2
```

This minimizes the need for hosts to learn a completely different gateway MAC after failover.

---

# 19. Proxy ARP

**Proxy ARP** occurs when a router answers an ARP request **on behalf of another IP address**.

Consider:

```text
Host ---- R1 ---- Remote Host
```

The host asks:

```text
Who has 172.16.20.20?
```

Even though `172.16.20.20` is not actually located on that Ethernet segment, R1 may respond:

```text
172.16.20.20 is reachable through my MAC address.
```

The host then sends the frame to R1, and R1 routes the packet normally.

Cisco Press describes Proxy ARP as a router answering an ARP request for a destination that is not on the receiving interface's local subnet when the router has a useful route toward that destination.

Proxy ARP can be useful, but it can also hide addressing or routing design mistakes.

### Best practice

When building Ethernet static routes, prefer a proper **next-hop IP address** rather than relying unnecessarily on an outgoing-interface-only static route that may depend on Proxy ARP.

---

# 20. ARP Failure Example

Suppose:

```text
PC1
172.16.10.10/24

Gateway
172.16.10.1
```

PC1 attempts:

```text
ping 8.8.8.8
```

Routing logic says:

```text
8.8.8.8 = remote
Use gateway 172.16.10.1
```

But suppose ARP resolution fails:

```text
172.16.10.1 → ???
```

Then PC1 cannot even construct the Ethernet frame needed to reach the router.

Therefore:

```text
Valid IP configuration
+
Valid default gateway
+
No ARP resolution
=
No Layer 3 forwarding
```

This is why an ARP problem can look like a routing problem.

---

# 21. Why the First Ping Sometimes Fails

You may occasionally see something similar to:

```text
.!!!!
```

during a Cisco ping test.

One possible reason is that the device has not yet resolved the next-hop MAC.

Conceptually:

```text
First packet
   ↓
No ARP entry
   ↓
ARP resolution
   ↓
MAC learned
   ↓
Following packets forwarded normally
```

This behavior is not guaranteed on every platform and should not automatically be blamed on ARP, but ARP resolution is a common explanation in a newly established forwarding path.

---

# 22. ARP Spoofing / ARP Poisoning

ARP has an important weakness:

**Traditional ARP does not authenticate the sender.**

An attacker can potentially send false information such as:

```text
Default gateway IP
172.16.10.1

actually maps to

ATTACKER MAC
```

Victim's corrupted ARP cache:

```text
172.16.10.1 → Attacker-MAC
```

Traffic then becomes:

```text
Victim
  |
  v
Attacker
  |
  v
Real Gateway
```

The attacker may inspect, modify, or forward the traffic.

This is an **ARP poisoning / man-in-the-middle** scenario. Cisco's security material describes ARP spoofing as poisoning IP-to-MAC bindings so traffic intended for another device is instead sent through the attacker.

---

# 23. Dynamic ARP Inspection — DAI

Cisco switches can protect against many ARP spoofing attacks using:

**Dynamic ARP Inspection (DAI).**

DAI examines ARP packets entering untrusted switchports.

Conceptually:

```text
ARP packet arrives
       |
       v
Is this port trusted?
     /     \
   Yes      No
   |         |
Forward    Validate
              |
              v
      Is IP ↔ MAC valid?
         /        \
       Yes         No
        |           |
     Forward       Drop
```

DAI commonly validates ARP information against the **DHCP Snooping binding database**. Cisco documents that invalid IP-to-MAC bindings can be intercepted, logged, and discarded.

---

# 24. DHCP Snooping + DAI Relationship

DHCP Snooping learns legitimate bindings such as:

```text
IP             MAC                 VLAN    Interface
172.16.10.50   AAAA.BBBB.CCCC      10      Gi1/0/10
```

If an ARP arrives from Gi1/0/10 claiming:

```text
172.16.10.50
MAC = DDDD.EEEE.FFFF
```

DAI sees:

```text
Expected:
AAAA.BBBB.CCCC

Received:
DDDD.EEEE.FFFF
```

and can drop the ARP.

Cisco Press describes this exact logic: on untrusted interfaces, DAI compares ARP sender/origin IP and MAC information to known DHCP Snooping bindings and discards invalid messages.

---

# 25. Basic Cisco DAI Example

### IOS/IOS-XE

Enable DAI on VLAN 10:

```ios
configure terminal
ip arp inspection vlan 10
```

Trusted infrastructure uplink:

```ios
interface GigabitEthernet1/0/24
 ip arp inspection trust
```

Verification:

```ios
show ip arp inspection vlan 10
show ip arp inspection interfaces
```

Cisco's SCOR guide documents these commands when implementing DAI.

### Production Risk

**Risk level:** Medium to High
**Blast radius:** The entire protected VLAN if incorrectly configured.

If DHCP Snooping bindings or static ARP validation information are missing, DAI can discard legitimate ARP and effectively break connectivity.

For production deployment:

```text
Lab validate
→ confirm DHCP Snooping
→ confirm trust boundaries
→ confirm static-IP handling
→ enable DAI
→ monitor drops
```

A rollback is straightforward:

```ios
no ip arp inspection vlan 10
```

but the implementation should still be maintenance-window validated when applied to critical networks.

---

# 26. Static-IP Devices and DAI

DHCP-based endpoints are straightforward because the switch learns their bindings through DHCP Snooping.

But servers, printers, infrastructure devices, or appliances may use static IP addresses.

For those environments, Cisco DAI can use mechanisms such as **ARP ACLs** in addition to DHCP Snooping bindings. Cisco Press explicitly notes ARP ACLs as an option for devices that do not obtain their addresses using DHCP.

Do not blindly enable DAI on a server VLAN without planning how static hosts will be validated.

---

# 27. ARP and IPv6

IPv6 does **not use ARP**.

IPv6 replaces ARP functionality with:

**Neighbor Discovery Protocol (NDP)**

which uses:

```text
ICMPv6
```

Conceptually:

```text
IPv4:
ARP Request / ARP Reply

IPv6:
Neighbor Solicitation / Neighbor Advertisement
```

So:

```text
IPv4 → ARP
IPv6 → NDP
```

This is an important CCNA/CCNP distinction.

---

# 28. Common ARP Troubleshooting Scenarios

## Scenario 1 — Local Hosts Cannot Communicate

Check:

```ios
show ip arp
show mac address-table
```

Questions:

```text
Is the target ARP entry present?
Is it associated with the expected MAC?
Does the switch learn that MAC on the expected port?
Are both endpoints in the same VLAN?
```

---

## Scenario 2 — Local Traffic Works but Remote Traffic Fails

Check the gateway:

```ios
show ip arp <gateway-ip>
```

The host/router needs to resolve its **next hop**, not the remote Internet destination.

---

## Scenario 3 — ARP Entry Is Missing

Verify:

```text
Correct VLAN
Correct subnet mask
Correct IP address
Interface up/up
Layer 2 connectivity
No DAI drops
No VLAN pruning
No STP blocking problem affecting the path
```

---

## Scenario 4 — ARP Maps to the Wrong MAC

Potential causes include:

```text
Duplicate IP address
ARP poisoning
Stale ARP entry
High-availability event
Proxy ARP
Incorrect static ARP
Virtualization movement
```

Correlate:

```ios
show ip arp <ip-address>
show mac address-table address <mac-address>
```

That gives you:

```text
IP
 ↓
MAC
 ↓
Physical switch port
```

---

# 29. The Troubleshooting Chain I Use

When troubleshooting an IPv4 forwarding problem on Ethernet, think in this order:

```text
1. ROUTING
   Do I have a route?

        ↓

2. NEXT HOP
   Which next hop/interface did the route select?

        ↓

3. ARP
   Can I resolve that next-hop IP to a MAC?

        ↓

4. CAM TABLE
   Where does the switch think that MAC lives?

        ↓

5. PHYSICAL/L2 PATH
   Is that port/trunk/VLAN actually forwarding?
```

Cisco commands:

```ios
show ip route <destination>
show ip cef <destination> detail
show ip arp <next-hop>
show mac address-table address <mac-address>
show interfaces status
show interfaces trunk
show spanning-tree vlan <vlan-id>
```

This separates routing problems from ARP problems from Layer 2 switching problems.

---

# 30. Common ARP Misconceptions

### Wrong

> ARP finds the MAC address of the final destination anywhere on the Internet.

### Correct

ARP resolves a **directly reachable Layer 2 neighbor**, typically either:

```text
Local destination
```

or:

```text
Next-hop router
```

---

### Wrong

> A switch needs ARP to switch every frame.

### Correct

A normal Layer 2 switch forwards transit Ethernet frames using its:

```text
MAC address table
```

not its ARP table.

---

### Wrong

> ARP requests cross routers.

### Correct

Normal ARP requests are Layer 2 broadcasts and stay inside the broadcast domain.

---

### Wrong

> The destination MAC and destination IP always belong to the same device.

### Correct

For remote traffic:

```text
Destination IP  = final remote host
Destination MAC = local next-hop router
```

---

### Wrong

> ARP and DNS perform similar functions.

They solve completely different mappings:

```text
DNS:
Name → IP

ARP:
IPv4 → MAC

CAM table:
MAC → Port
```

That gives a useful complete chain:

```text
www.example.com
      ↓ DNS
203.0.113.10
      ↓ Routing
Next hop 172.16.10.1
      ↓ ARP
AAAA.BBBB.CCCC
      ↓ MAC table
Gi1/0/24
```

---

# 31. ARP in Plain English

Imagine IP addressing as knowing someone's **street address**, while Ethernet needs to know which **door in the local building** to deliver the message to.

ARP asks:

> "I know the IPv4 address. Which MAC address should I actually send the Ethernet frame to?"

For a local device:

```text
ARP for the destination itself.
```

For a remote device:

```text
ARP for the default gateway / next-hop router.
```

The shortest accurate summary is:

```text
ARP
=
IPv4-to-MAC resolution
for directly reachable Layer 2 neighbors.
```

---

# 32. CCNA / CCNP / CCIE Memory Map

```text
ARP Request
→ Broadcast

ARP Reply
→ Normally unicast

ARP maps
→ IPv4 to MAC

ARP scope
→ Local broadcast domain / VLAN

Local destination
→ ARP for destination

Remote destination
→ ARP for gateway/next hop

Routing table
→ Destination IP → Next hop

ARP table
→ Next-hop IP → MAC

MAC table
→ MAC → Switch port

Gratuitous ARP
→ Unsolicited announcement/update

Proxy ARP
→ Router answers on behalf of another IP

ARP poisoning
→ False IP-to-MAC bindings

DAI
→ Validates ARP

DHCP Snooping
→ Provides bindings used by DAI

IPv6
→ Uses NDP, not ARP
```

## Final Mental Model

When an IPv4 device needs to transmit traffic:

<div align="center">
  <img
    src="Images/ARP/ARP Flowchart.png"
    alt="ARP Flowchart"
    width="450"
  />
</div>

That is ARP's real position in packet forwarding: **routing tells the device where to go; ARP gives it the Layer 2 information required to actually get to the next Ethernet hop.**

### Sources

Cisco Press, *CCNA 200-301 Official Cert Guide, Volume 1, Second Edition* — Fundamentals of WANs and IP Routing; IPv4 routing and ARP.

Cisco Press, *CCNP and CCIE Enterprise Core ENCOR 350-401 Official Cert Guide, Second Edition* — Packet Forwarding and ARP table operation.

Cisco Press, *CCNA 200-301 Official Cert Guide, Volume 2* — Gratuitous ARP, ARP poisoning, DHCP Snooping, and Dynamic ARP Inspection.

Cisco Press, *CCNP and CCIE Security Core SCOR 350-701 Official Cert Guide* — ARP spoofing and Dynamic ARP Inspection.

RFC 826 — *An Ethernet Address Resolution Protocol*.
