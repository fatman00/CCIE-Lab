# Lab design
- Four IOSv routers in AS 65000
- One external IOSv router in AS 65100
- Redundant OSPF underlay between the internal routers
- Loopback-sourced iBGP sessions
- Internal and external test prefixes
- No topology annotations
- No initial BGP configuration, allowing students to perform every BGP task themselves

# Workbook Topics

- Underlay and TCP/179 validation
- Loopback-sourced iBGP
- The iBGP advertisement rule
- Full-mesh session scaling
- Unchanged NEXT_HOP and next-hop-self
- LOCAL_PREF, MED, and Cisco weight behavior
- Route-reflector clients
- ORIGINATOR_ID and CLUSTER_LIST
- Route-reflection path hiding
- Peer groups and per-neighbor policy exceptions
- Physical-link and route-reflector failure drills
- Final verification and CCIE-style review questions
