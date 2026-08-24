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
