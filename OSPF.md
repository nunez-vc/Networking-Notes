**Open Short Path First (OSPF)**

**OSPF Fundamentals**
- OSPF is an open standard.
- OSPF is a Link State routing Protocol.
- Every router has a "map" of the network called a Link State Database.
- OSPF runs the Dijkstra Algorithm to determine the shortest path to a network.
- OSPF administrative distance is 110.

**Link State Routing Protocol**
- It is a dynamic network protocol where every router builds a complete map of the entire network topology.
- OSPF and IS-IS use Link State Routing Protocol.

**OSPF Characteristics**
- Establishes adjacencies with other routers.
- Sends Link State Advertisements (LSA) to other routers in an area.
- Constructs a Link State Database from received LSAs.
- Runs the Dijkstra Shortest Path First Algorithm to determine the shortest path to a network.
- Attempts to inject the best path for each network into a router's IP routing table.

**OSPF Terminlogy**
- **Hello** - A protocol use to discover OSPF neighbors and confirm reachability to those neighbors (also used in the election of a Designated Router).
- **Link State Advertisement (LSA)** - Information a router sends and receives about network reachability (used to construct a router's Link State Database).
- **Link State Update (LSU)** - A packet that carries LSAs.
- **Link State Request (LSR)** - Used by a router to specific LSA information from a neighbor.

**Neighborship vs Adjacencies**

**Neighbor**
- Reside on the same network link.
- Exchange Hello messages.

**Adjacencies**
- Are Neighbors
- Have exchanged Link State Updates (LSUs) and Database Description (DD) packets.

**Neighborship Requirements**
- Matching Area
- Matching Authentication
- Matching Subnet
- Matching Timer
- Matching Stub Flags
- Matching MTU (EXSTART/EXCHANGE State)

**OSPF Cost**
- The default reference bandwidth is 100,000,000 bits per second (100Mbps).
- Formula: Cost = Reference BW / Interface BW

<Insert Image>

**The need of Designated Routers**
- Adjacencies only need to be formed with the DR and BDR.
- 224.0.0.5 or FF02::5 - All OSPF routers
- 224.0.0.6 or FF02::6 - All designated routers

**DR and BDR Election**
- Highest Priority Wins
- To configure OSPF priority: ip ospf priority [priority #]
- A priority of 0 prevents a router from participating in the election.
- Tie Breaker:
  1. Highest Router ID wins
  2. If there's no configured Router ID, the highest IP address on a Loopback interface wins.
  3. If there's no Loopback interface, the highest IP address on an interface that's up wins.

**OSPF Areas**
- Multi-Area OSPF networks must have a Backbone Area numbered 0 or 0.0.0.0

**Area Border Routers (ABR)**
- OSPF router with interfaces connected simultaneously to the backbone area (Area 0) and at least one other non-backbone area.
- Responsible for generating and advertising Type 3 summary LSAs for every single network in area 0.

**LSA Types**
1. **Type 1 LSA** - A **Router LSA** is created by each router and contains information about that router's directly attached networks.
2. **Type 2 LSA** - A **Network LSA** is created for each transit network within an area on which a DR is elected. In order for a router to generate a type 2 LSA, that link has to match 2 criteria. First, it has to be a transit area meaning an OSPF router to an OSPF router (router to router), and second it has to be a link on which you would elect a link a DR.
3. **Type 3 LSA** - A **Summary LSA** is sent from one area to another and is used to advertise a network in the source area.
4. **Type 4 LSA** - A **Summary ASBR LSA** is created by an ABR to tell members of an area how to reach an ASBR.
5. **Type 5 LSA** - A **AS External LSA** is created by and ASBR to advertise networks in a different AS.

**Autonomous System Boundary Router**
- Sitting in the boundary of 2 autonomous systems.
