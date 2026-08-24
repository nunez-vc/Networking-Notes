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
- To configure OSPF priority: ip ospf priority [priority 3]
- A priority of 0 prevents a router from participating in the election.
- Tie Breaker:
  1. Highest Router ID wins
  2. If there's no configured Router ID, the highest IP address on a Loopback interface wins.
  3. If there's no Loopback interface, the highest IP address on an interface that's up wins.
  
