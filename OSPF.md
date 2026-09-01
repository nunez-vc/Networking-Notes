# **Open Short Path First (OSPF)**

---

## **OSPF Fundamentals**
- OSPF is an open standard.
- OSPF is a Link State routing Protocol.
- Every router has a "map" of the network called a Link State Database.
- OSPF runs the Dijkstra Algorithm to determine the shortest path to a network.
- OSPF administrative distance is 110.

---

## **Link State Routing Protocol**
- It is a dynamic network protocol where every router builds a complete map of the entire network topology.
- OSPF and IS-IS use Link State Routing Protocol.

---

## **OSPF Characteristics**
- Establishes adjacencies with other routers.
- Sends Link State Advertisements (LSA) to other routers in an area.
- Constructs a Link State Database from received LSAs.
- Runs the Dijkstra Shortest Path First Algorithm to determine the shortest path to a network.
- Attempts to inject the best path for each network into a router's IP routing table.

---

## **OSPF Terminlogy**
- **Hello** - A protocol use to discover OSPF neighbors and confirm reachability to those neighbors (also used in the election of a Designated Router).
- **Link State Advertisement (LSA)** - Information a router sends and receives about network reachability (used to construct a router's Link State Database).
- **Link State Update (LSU)** - A packet that carries LSAs.
- **Link State Request (LSR)** - Used by a router to specific LSA information from a neighbor.

---

## **Neighborship vs Adjacencies**

### **Neighbor**
- Reside on the same network link.
- Exchange Hello messages.

### **Adjacencies**
- Are Neighbors
- Have exchanged Link State Updates (LSUs) and Database Description (DD) packets.

---

## **Neighborship Requirements**
- Matching Area
- Matching Authentication
- Matching Subnet
- Matching Timer
- Matching Stub Flags
- Matching MTU (EXSTART/EXCHANGE State)

---

## **OSPF Cost**
- The default reference bandwidth is 100,000,000 bits per second (100Mbps).
- Formula: Cost = Reference BW / Interface BW

<Insert Image>

---

## **The need of Designated Routers**
- Adjacencies only need to be formed with the DR and BDR.
- 224.0.0.5 or FF02::5 - All OSPF routers
- 224.0.0.6 or FF02::6 - All designated routers

---

## **DR and BDR Election**
- Highest Priority Wins
- To configure OSPF priority: ip ospf priority [priority #]
- A priority of 0 prevents a router from participating in the election.
- Tie Breaker:
  1. Highest Router ID wins
  2. If there's no configured Router ID, the highest IP address on a Loopback interface wins.
  3. If there's no Loopback interface, the highest IP address on an interface that's up wins.

---

## **OSPF Areas**
- Multi-Area OSPF networks must have a Backbone Area numbered 0 or 0.0.0.0

---

## **Area Border Routers (ABR)**
- OSPF router with interfaces connected simultaneously to the backbone area (Area 0) and at least one other non-backbone area.
- Responsible for generating and advertising Type 3 summary LSAs for every single network in area 0.

---

## **LSA Types**
1. **Type 1 LSA** - A **Router LSA** is created by each router and contains information about that router's directly attached networks.
2. **Type 2 LSA** - A **Network LSA** is created for each transit network within an area on which a DR is elected. In order for a router to generate a type 2 LSA, that link has to match 2 criteria. First, it has to be a transit area meaning an OSPF router to an OSPF router (router to router), and second it has to be a link on which you would elect a link a DR.
3. **Type 3 LSA** - A **Summary LSA** is sent from one area to another and is used to advertise a network in the source area.
4. **Type 4 LSA** - A **Summary ASBR LSA** is created by an ABR to tell members of an area how to reach an ASBR.
5. **Type 5 LSA** - A **AS External LSA** is created by and ASBR to advertise networks in a different AS.
6. **Type 6 LSA** - A **NSSA LSA** is sent from an ASBR into an NSSA to advertise networks from a different AS.

---

## **Autonomous System Boundary Router**
- Sitting in the boundary of 2 autonomous systems.

---

## **Stub Area**
- Blocks external routes (routes redistributed into OSPF from other protocols like EIGRP or static routes) while still learning routes from other OSPF areas. Instead of receiving external routes, the ABR provides a default route.
- “Tell me about other OSPF areas, but don't give me all the external routes.”
- No external routes

---

## **Totally Stubby Area**
- Blocks both external routes and inter-area routes. Routers inside this area only know how to reach devices in their own local area.
- "I only need to know my local area. Send everything else to the ABR.”
- No external + no detailed inter-area routes

---

## **Not So Stubby Area (NSSA)**
- Behaves like a Stub Area but it allows a router inside the area to redistribute external routes into OSPF.
- “Don't send me external routes from outside, but allow me to introduce my own external routes.”
-  Stub + redistribution allowed

---

## **Totally Not So Stubby Area**
- Combines Totally Stubby + NSSA behavior.
- It blocks detailed inter-area and external routes, but still allows redistribution from an ASBR inside the area.
- “Give me only local routes and a default route, but still allow me to redistribute my own external routes.”
- Totally Stubby + redistribution allowed

---

## CCNA Configuration

**IOS-XE — OSPFv2 Process and Interface Activation**

| Command | Description |
|---|---|
| **Start OSPF process:**<br>`(config)#router ospf <process-id>` | Enters the OSPFv2 routing process. |
| **Set router ID:**<br>`(config-router)#router-id <router-id>` | Statically assigns the OSPF router ID. |
| **Enable OSPF by network statement:**<br>`(config-router)#network <ip-address> <wildcard-mask> area <area-id>` | Enables OSPF on matching interfaces in the specified area. |
| **Enable OSPF directly on interface:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip ospf <process-id> area <area-id>` | Enables OSPF directly on the selected interface. |

**IOS-XE — OSPFv2 Passive Interfaces**

| Command | Description |
|---|---|
| **Make one interface passive:**<br>`(config-router)#passive-interface <interface-id>` | Suppresses OSPF neighbor formation on the selected interface. |
| **Make all interfaces passive:**<br>`(config-router)#passive-interface default` | Makes all OSPF-enabled interfaces passive by default. |
| **Re-enable one interface:**<br>`(config-router)#no passive-interface <interface-id>` | Restores OSPF neighbor formation on the selected interface. |

**IOS-XE — OSPFv2 Network Type and DR/BDR**

| Command | Description |
|---|---|
| **Set broadcast network type:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip ospf network broadcast` | Sets the interface OSPF network type to broadcast. |
| **Set point-to-point network type:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip ospf network point-to-point` | Sets the interface OSPF network type to point-to-point. |
| **Set interface priority:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip ospf priority <0-255>` | Sets the OSPF DR/BDR election priority. |

**IOS-XE — OSPFv2 Cost and Reference Bandwidth**

| Command | Description |
|---|---|
| **Set interface OSPF cost:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip ospf cost <1-65535>` | Statically sets the OSPF interface cost. |
| **Set interface bandwidth:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#bandwidth <kilobits>` | Sets interface bandwidth used by OSPF cost calculation. |
| **Set reference bandwidth:**<br>`(config-router)#auto-cost reference-bandwidth <mbps>` | Sets the OSPF reference bandwidth in Mbps. |

**IOS-XE — OSPFv2 Timers**

| Command | Description |
|---|---|
| **Set hello interval:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip ospf hello-interval <seconds>` | Sets the OSPF hello interval. |
| **Set dead interval:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip ospf dead-interval <seconds>` | Sets the OSPF dead interval. |

**IOS-XE — OSPFv2 Default Route Advertisement**

| Command | Description |
|---|---|
| **Originate existing default:**<br>`(config-router)#default-information originate` | Advertises an existing default route into OSPF. |
| **Always originate default:**<br>`(config-router)#default-information originate always` | Advertises an OSPF default regardless of local default presence. |

**IOS-XE — OSPFv2 Verification**

| Command | Description |
|---|---|
| **Show OSPF process:**<br>`#show ip ospf` | Displays OSPF process, router ID, areas, and statistics. |
| **Show OSPF interfaces:**<br>`#show ip ospf interface` | Displays detailed OSPF interface state and timers. |
| **Show interface summary:**<br>`#show ip ospf interface brief` | Displays summarized OSPF interfaces, areas, costs, and states. |
| **Show specific interface:**<br>`#show ip ospf interface <interface-id>` | Displays detailed OSPF state for one interface. |
| **Show OSPF neighbors:**<br>`#show ip ospf neighbor` | Displays OSPF neighbor adjacencies and states. |
| **Show interface neighbors:**<br>`#show ip ospf neighbor <interface-id>` | Displays OSPF neighbors learned on one interface. |
| **Show specific neighbor:**<br>`#show ip ospf neighbor <neighbor-router-id>` | Displays detailed information for one OSPF neighbor. |
| **Show OSPF database:**<br>`#show ip ospf database` | Displays summarized OSPF link-state database entries. |
| **Show OSPF routes:**<br>`#show ip route ospf` | Displays IPv4 routes installed by OSPF. |
| **Show protocol configuration:**<br>`#show ip protocols` | Displays OSPF process parameters and enabled networks. |

## CCNP Configuration

**CCNP Enterprise — IOS-XE — Multi-Area OSPFv2**

| Command | Description |
|---|---|
| **Enable interface in another area:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip ospf <process-id> area <area-id>` | Assigns the interface to the specified OSPF area. |
| **Enable network in another area:**<br>`(config-router)#network <ip-address> <wildcard-mask> area <area-id>` | Enables matching interfaces in the specified OSPF area. |
| **Show area information:**<br>`#show ip ospf` | Displays OSPF areas and process-level information. |
| **Show inter-area routes:**<br>`#show ip route ospf` | Displays installed intra-area and inter-area OSPF routes. |

**CCNP Enterprise — IOS-XE — OSPFv2 Inter-Area Summarization**

| Command | Description |
|---|---|
| **Summarize inter-area routes:**<br>`(config-router)#area <area-id> range <network> <subnet-mask>` | Summarizes routes crossing an OSPF area border router. |
| **Set summary metric:**<br>`(config-router)#area <area-id> range <network> <subnet-mask> cost <metric>` | Sets the advertised metric for the OSPF summary. |
| **Suppress summary range:**<br>`(config-router)#area <area-id> range <network> <subnet-mask> not-advertise` | Suppresses type-3 advertisement for the matched range. |
| **Explicitly advertise summary:**<br>`(config-router)#area <area-id> range <network> <subnet-mask> advertise` | Explicitly advertises the configured inter-area summary. |

**CCNP Enterprise — IOS-XE — OSPFv2 ABR Route Filtering**

| Command | Description |
|---|---|
| **Create prefix list:**<br>`(config)#ip prefix-list <name> [seq <sequence>] <permit|deny> <prefix>/<length> [ge <minimum>] [le <maximum>]` | Creates the prefix list used for OSPF area filtering. |
| **Filter routes entering area:**<br>`(config-router)#area <area-id> filter-list prefix <prefix-list-name> in` | Filters type-3 routes entering the specified OSPF area. |
| **Filter routes leaving area:**<br>`(config-router)#area <area-id> filter-list prefix <prefix-list-name> out` | Filters type-3 routes leaving the specified OSPF area. |
| **Show prefix list:**<br>`#show ip prefix-list [<name>]` | Displays configured prefix-list entries and match counters. |

**CCNP Enterprise — IOS-XE — OSPFv2 Advanced Default Origination**

| Command | Description |
|---|---|
| **Set default metric:**<br>`(config-router)#default-information originate [always] metric <metric-value>` | Originates the OSPF default with a configured metric. |
| **Set external metric type:**<br>`(config-router)#default-information originate [always] metric-type <1|2>` | Selects OSPF external metric type for the default route. |

**CCNP Enterprise — IOS-XE — OSPFv2 LSDB Verification**

| Command | Description |
|---|---|
| **Show router LSAs:**<br>`#show ip ospf database router` | Displays OSPF type-1 router LSAs. |
| **Show network LSAs:**<br>`#show ip ospf database network` | Displays OSPF type-2 network LSAs. |
| **Show summary LSAs:**<br>`#show ip ospf database summary` | Displays OSPF type-3 summary LSAs. |

**CCNP Enterprise — IOS-XE — OSPFv3 Process and IPv6 Activation**

| Command | Description |
|---|---|
| **Enable IPv6 forwarding:**<br>`(config)#ipv6 unicast-routing` | Enables IPv6 unicast routing for OSPFv3 operation. |
| **Start OSPFv3 process:**<br>`(config)#router ospfv3 <process-id>` | Enters the OSPFv3 routing process. |
| **Set OSPFv3 router ID:**<br>`(config-router)#router-id <router-id>` | Statically assigns the OSPFv3 router ID. |
| **Create IPv6 address family:**<br>`(config-router)#address-family ipv6 unicast` | Enters the OSPFv3 IPv6 unicast address family. |
| **Enable IPv6 OSPFv3 on interface:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ospfv3 <process-id> ipv6 area <area-id>` | Enables OSPFv3 IPv6 on the selected interface. |

**CCNP Enterprise — IOS-XE — OSPFv3 IPv4 Address Family**

| Command | Description |
|---|---|
| **Create IPv4 address family:**<br>`(config)#router ospfv3 <process-id>`<br>&nbsp;&nbsp;○ `(config-router)#address-family ipv4 unicast` | Enters the OSPFv3 IPv4 unicast address family. |
| **Enable IPv4 OSPFv3 on interface:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ospfv3 <process-id> ipv4 area <area-id>` | Enables OSPFv3 IPv4 on the selected interface. |

**CCNP Enterprise — IOS-XE — OSPFv3 Passive Interfaces**

| Command | Description |
|---|---|
| **Make one interface passive:**<br>`(config-router)#passive-interface <interface-id>` | Makes the selected OSPFv3 interface passive. |
| **Make all interfaces passive:**<br>`(config-router)#passive-interface default` | Makes all OSPFv3 interfaces passive by default. |
| **Re-enable one interface:**<br>`(config-router)#no passive-interface <interface-id>` | Restores OSPFv3 neighbor formation on one interface. |

**CCNP Enterprise — IOS-XE — OSPFv3 Network Type**

| Command | Description |
|---|---|
| **Set broadcast network type:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ospfv3 network broadcast` | Sets the OSPFv3 interface network type to broadcast. |
| **Set point-to-point network type:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ospfv3 network point-to-point` | Sets the OSPFv3 interface network type to point-to-point. |

**CCNP Enterprise — IOS-XE — OSPFv3 Inter-Area Summarization**

| Command | Description |
|---|---|
| **Summarize IPv6 routes:**<br>`(config)#router ospfv3 <process-id>`<br>&nbsp;&nbsp;○ `(config-router)#address-family ipv6 unicast`<br>&nbsp;&nbsp;○ `(config-router-af)#area <area-id> range <ipv6-prefix>/<prefix-length>` | Summarizes IPv6 routes crossing an OSPFv3 ABR. |

**CCNP Enterprise — IOS-XE — OSPFv3 Verification**

| Command | Description |
|---|---|
| **Show OSPFv3 interfaces:**<br>`#show ospfv3 interface` | Displays detailed OSPFv3 interface state. |
| **Show OSPFv3 interface summary:**<br>`#show ospfv3 interface brief` | Displays interface, process, area, AF, cost, and state. |
| **Show specific OSPFv3 interface:**<br>`#show ospfv3 interface <interface-id>` | Displays detailed OSPFv3 state for one interface. |
| **Show IPv6 neighbors:**<br>`#show ospfv3 ipv6 neighbor` | Displays OSPFv3 IPv6 neighbor adjacencies. |
| **Show all OSPFv3 neighbors:**<br>`#show ospfv3 neighbor` | Displays OSPFv3 neighbors across enabled address families. |
| **Show IPv6 OSPF routes:**<br>`#show ipv6 route ospf` | Displays IPv6 routes installed by OSPFv3. |
| **Show IPv4 OSPFv3 routes:**<br>`#show ip route ospfv3` | Displays IPv4 routes installed through OSPFv3. |

**CCNP Security — IOS-XE — OSPFv2 Plaintext Authentication**

| Command | Description |
|---|---|
| **Enable area authentication:**<br>`(config-router)#area <area-id> authentication` | Enables plaintext OSPF authentication for the specified area. |
| **Set interface authentication key:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip ospf authentication-key <password>` | Sets the plaintext OSPF authentication key on the interface. |
| **Verify authentication:**<br>`#show ip ospf interface <interface-id>` | Displays configured OSPF authentication state for the interface. |

**CCNP Security — IOS-XE — OSPFv2 MD5 Authentication**

| Command | Description |
|---|---|
| **Enable area MD5 authentication:**<br>`(config-router)#area <area-id> authentication message-digest` | Enables OSPF MD5 authentication for the specified area. |
| **Enable interface MD5 authentication:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip ospf authentication message-digest` | Enables OSPF MD5 authentication on the interface. |
| **Configure MD5 key:**<br>`(config)#interface <interface-id>`<br>&nbsp;&nbsp;○ `(config-if)#ip ospf message-digest-key <key-id> md5 <password>` | Configures the OSPF MD5 key and key identifier. |
| **Verify MD5 authentication:**<br>`#show ip ospf interface <interface-id>` | Displays OSPF authentication configuration and interface state. |
