# CCIE Workbook: BGP iBGP Peer Relations

## Purpose

This lab isolates the rules that govern internal BGP peer relationships. The topology is intentionally small, but it exposes the behaviors that cause most production iBGP failures: transport reachability, loopback sourcing, the iBGP advertisement rule, unchanged next hop, full-mesh scaling, and route reflection.

## Topology

| Router | Role | AS | Router ID | Relevant prefixes |
|---|---|---:|---|---|
| EDGE | External origin | 65100 | 10.255.1.1 | 203.0.113.0/24 |
| R1 | Border router | 65000 | 10.255.0.1 | eBGP attachment to EDGE |
| R2 | Internal router | 65000 | 10.255.0.2 | None initially |
| R3 | Internal router | 65000 | 10.255.0.3 | None initially |
| R4 | Internal origin | 65000 | 10.255.0.4 | 10.44.44.44/32 |

Core physical links run OSPF area 0. Loopback0 addresses are advertised in OSPF and are intended as iBGP endpoints. The EDGE-facing subnet is deliberately excluded from OSPF.

## Addressing

| Link | Left address | Right address |
|---|---|---|
| EDGE to R1 | EDGE 192.0.2.2/30 | R1 192.0.2.1/30 |
| R1 to R2 | R1 10.0.12.1/30 | R2 10.0.12.2/30 |
| R1 to R3 | R1 10.0.13.1/30 | R3 10.0.13.2/30 |
| R2 to R4 | R2 10.0.24.1/30 | R4 10.0.24.2/30 |
| R3 to R4 | R3 10.0.34.1/30 | R4 10.0.34.2/30 |

## Ground Rules

- Do not redistribute BGP into OSPF or OSPF into BGP.
- Use Loopback0 for every iBGP session.
- Do not add static routes unless a task explicitly asks for one.
- Use soft policy refresh where possible. Do not clear all BGP sessions casually.
- Verify both the BGP table and the routing table. A valid BGP path is not automatically an installed route.

---

# Part 1: Baseline Validation

## Task 1.1: Validate the underlay

1. Start all nodes and wait for OSPF convergence.
2. On R1 through R4, verify that every core Loopback0 is present in the routing table as an OSPF route, except the router's own connected loopback.
3. From every core router, ping every other core Loopback0 using Loopback0 as the source.
4. Confirm that R2 and R3 do not have a route to 192.0.2.0/30.
5. Confirm that R1 can ping EDGE at 192.0.2.2.

**Commands to consider**

```text
show ip ospf neighbor
show ip route ospf
ping 10.255.0.4 source 10.255.0.2
show ip route 192.0.2.0
```

**Success criteria**

- OSPF adjacencies are FULL on all four core links.
- All core loopbacks are mutually reachable.
- The external transit subnet is not leaked into the IGP.

## Task 1.2: Record the control-plane baseline

On R1 through R4, record:

- OSPF neighbor count
- Number of OSPF routes
- Whether any BGP process exists
- Whether 203.0.113.0/24 is reachable

This snapshot will be used to distinguish underlay failure from BGP failure later.

---

# Part 2: Establish iBGP Correctly

## Task 2.1: Build one loopback-sourced iBGP session

Configure an iBGP session between R1 and R2 in AS 65000 using their Loopback0 addresses.

Requirements:

- Explicit router ID on both routers
- Neighbor address must be the remote Loopback0
- Source the TCP session from local Loopback0
- Do not use physical-interface addresses as BGP neighbors

Verify:

```text
show ip bgp summary
show tcp brief | include 179
show ip bgp neighbors 10.255.0.2
```

Answer these questions:

1. Why is `update-source Loopback0` required?
2. Why is `ebgp-multihop` not required for iBGP?
3. Which route in the RIB enables the TCP session to form?
4. Does iBGP require the peers to be directly connected?

## Task 2.2: Create a transport failure without breaking OSPF globally

On R2, deny only TCP destination port 179 sourced from R1's Loopback0 and destined to R2's Loopback0. Apply the filter in the direction that blocks session establishment while preserving ICMP.

Observe:

- Loopback-to-loopback ping still succeeds.
- The BGP session does not establish or drops.
- The neighbor state and connection retry behavior identify a transport problem rather than a routing problem.

Remove the filter and verify automatic recovery.

**Learning point:** IP reachability is necessary, but TCP/179 reachability is also required.

---

# Part 3: The iBGP Advertisement Rule

## Task 3.1: Create a partial iBGP chain

Build only these sessions in AS 65000:

- R1 to R2
- R2 to R4

Do not configure R1 to R4 yet. Do not configure any BGP session on R3.

On R4, originate 10.44.44.44/32 into BGP with an exact `network` statement.

Verify:

1. R2 learns 10.44.44.44/32 from R4.
2. R2 selects the path as best.
3. R1 does not learn 10.44.44.44/32 from R2.
4. R2's BGP table contains the prefix even though R2 does not advertise it to R1.

Use neighbor-specific advertised-route output to prove the result.

```text
show ip bgp 10.44.44.44
show ip bgp neighbors 10.255.0.1 advertised-routes
show ip bgp neighbors 10.255.0.4 routes
```

Explain why the word “split horizon” is useful shorthand but not the exact protocol mechanism. State the precise iBGP advertisement rule.

## Task 3.2: Repair reachability using a full mesh

Add the missing direct iBGP relationship between R1 and R4. Keep R2 to R4 and R1 to R2.

Verify that R1 now learns 10.44.44.44/32 directly from R4. Then add R3 to the iBGP domain and create a complete four-router iBGP full mesh.

For four routers, calculate:

- Required number of unique iBGP sessions
- Number of neighbors configured per router
- Formula for `n` routers
- Session count at 10, 50, and 100 routers

Expected session-count formula:

```text
n(n-1)/2
```

Do not continue until all six sessions for four routers are Established.

---

# Part 4: eBGP into iBGP and Next-Hop Reachability

## Task 4.1: Establish the external session

Configure eBGP between EDGE AS 65100 and R1 AS 65000 using the directly connected 192.0.2.0/30 addresses.

On EDGE:

- Originate 203.0.113.0/24 with an exact network statement.
- Ensure the prefix exists in the local routing table.

On R1:

- Accept the route without redistributing it into OSPF.

Verify that R1 installs 203.0.113.0/24 with next hop 192.0.2.2.

## Task 4.2: Observe unchanged next hop across iBGP

Allow R1 to advertise 203.0.113.0/24 to all iBGP peers without changing the next hop.

On R2, R3, and R4, determine:

- Whether the prefix is present in the BGP table
- Whether the path is valid
- Whether it is selected as best
- Whether it is installed in the IP routing table
- Which exact route is missing

Correlate the BGP status codes with recursive next-hop resolution.

**Constraint:** Do not add 192.0.2.0/30 to OSPF.

## Task 4.3: Repair the next hop at the boundary

Configure R1 so that external routes advertised to its iBGP peers use R1's Loopback0 as next hop.

Requirements:

- Apply the behavior only where needed.
- Verify that the next hop becomes 10.255.0.1.
- Verify that 10.255.0.1 resolves through OSPF.
- Verify that R2, R3, and R4 install 203.0.113.0/24.
- Prove end-to-end forwarding with a sourced ping where supported.

Explain why `next-hop-self` is commonly applied on an AS boundary and why blindly enabling it on every iBGP relationship can hide design errors.

---

# Part 5: Path Selection and iBGP Attributes

## Task 5.1: Compare local preference and MED scope

Create a second logical policy experiment without adding links:

1. On R1, set local preference 200 for 203.0.113.0/24 when receiving it from EDGE.
2. Confirm that local preference is carried to iBGP peers.
3. On EDGE, attach MED 50 to the same prefix toward R1.
4. Confirm which attributes are visible on R2.
5. Document which attributes are intended to influence outbound versus inbound path choice.

Remove the policy after verification.

## Task 5.2: Demonstrate the weight trap

On R2, apply Cisco weight 500 to 203.0.113.0/24 received from R1.

Verify:

- Weight affects only R2.
- R3 and R4 do not see the value.
- The attribute is not propagated in BGP updates.

Remove the weight policy after verification.

---

# Part 6: Replace the Full Mesh with Route Reflection

## Task 6.1: Baseline before migration

Record on every router:

```text
show ip bgp summary
show ip bgp 10.44.44.44
show ip bgp 203.0.113.0
```

Save the number of Established sessions and the best-path next hops.

## Task 6.2: Convert R2 into a route reflector

Redesign the iBGP topology as follows:

- R2 is the route reflector.
- R1, R3, and R4 are route-reflector clients.
- Each client has only one iBGP session, to R2.
- Remove all client-to-client iBGP sessions.
- Keep all sessions loopback sourced.

Do not use a peer group yet.

Verify:

1. R1 learns 10.44.44.44/32 through R2.
2. R4 learns 203.0.113.0/24 through R2.
3. R3 learns both prefixes.
4. Reflected routes include ORIGINATOR_ID and CLUSTER_LIST where applicable.
5. R2 does not reflect a route back to the client that originated it.

Answer:

- Which routes may be reflected from a client to another client?
- Which routes may be reflected from a client to a non-client?
- Which routes learned from a non-client may be sent to a client?
- Which routes learned from a non-client may be sent to another non-client?

## Task 6.3: Inspect loop prevention

On R2, explicitly set cluster ID 10.255.0.2. Observe reflected routes on R1, R3, and R4.

On one client, temporarily create a policy that attempts to return a reflected prefix toward R2. Determine which attribute prevents a reflection loop and where the update is rejected.

Restore the intended design.

## Task 6.4: Identify route-reflection path hiding

Using the existing topology, reason through this condition:

- R2 receives two paths for the same prefix from two clients.
- R2 selects only one best path.
- Another client may prefer the hidden path if it could see both.

Document:

1. Why a route reflector does not normally advertise every alternate path.
2. How this differs from a full mesh.
3. Two design mitigations, such as multiple route reflectors with diverse placement or BGP ADD-PATH where platform support and design permit it.

No configuration is required for ADD-PATH in this lab.

---

# Part 7: Peer Groups and Operational Consistency

## Task 7.1: Apply a peer group on R2

Replace repeated client configuration on R2 with an iBGP peer group for R1, R3, and R4.

The peer group must define:

- Remote AS 65000
- Update source Loopback0
- Route-reflector-client behavior
- A descriptive outbound policy name, even if the initial policy permits all routes

Verify that all three sessions remain Established and that no route disappears during the change.

## Task 7.2: Controlled policy failure

On R2, modify the outbound policy so that 10.44.44.44/32 is denied only toward R3, while 203.0.113.0/24 remains permitted.

Requirements:

- Preserve the common peer-group configuration.
- Add the exception at the individual-neighbor level if the IOS image permits inheritance override; otherwise create a second peer group for the exception.
- Do not use a blanket deny without an explicit permit for other routes.

Verify the effect from R3 and prove that R1 still receives 10.44.44.44/32.

Restore normal policy afterward.

---

# Part 8: Failure and Convergence Drills

## Task 8.1: Underlay failure with loopback peering

With route reflection operational, shut the R1 to R2 physical link.

Observe:

- OSPF reconvergence
- Continued reachability between 10.255.0.1 and 10.255.0.2 through R3 and R4
- Whether the R1 to R2 iBGP session survives
- Whether 203.0.113.0/24 remains available internally

Then restore the link.

Explain why loopback peering decouples BGP session fate from a single interface, but not from total underlay reachability.

## Task 8.2: Route-reflector failure

Shut down the BGP process on R2 or administratively disable all three client neighbors.

Measure:

- Time until clients lose reflected routes
- Which prefixes remain locally originated or directly learned
- Difference between BGP hold-time expiration and immediate administrative shutdown

Restore R2.

Propose the minimum production redesign that removes R2 as a single control-plane failure point. Include client peering to two route reflectors and discuss why the reflectors should have consistent policy and distinct cluster IDs unless they are intentionally modeled as one logical cluster.

---

# Final Verification

The final state must satisfy all of the following:

- EDGE and R1 have an Established eBGP session.
- R2 is the route reflector for R1, R3, and R4.
- No client-to-client iBGP sessions remain.
- All iBGP sessions use Loopback0 as the source.
- R1 applies next-hop-self toward R2 for the external route.
- R1, R2, R3, and R4 install 203.0.113.0/24.
- R1, R2, and R3 install 10.44.44.44/32 through BGP.
- OSPF carries infrastructure only, not 203.0.113.0/24 or 10.44.44.44/32.
- The BGP table demonstrates ORIGINATOR_ID and CLUSTER_LIST on reflected paths where applicable.

## Final command set

```text
show ip bgp summary
show ip bgp
show ip bgp 203.0.113.0/24
show ip bgp 10.44.44.44/32
show ip bgp neighbors
show ip route bgp
show ip route 10.255.0.0 255.255.255.0 longer-prefixes
show ip ospf neighbor
show tcp brief | include 179
```

# Review Questions

1. What exact rule prevents a normal iBGP speaker from advertising an iBGP-learned route to another iBGP peer?
2. Why does iBGP not change NEXT_HOP by default?
3. Why can a prefix appear in `show ip bgp` but not in `show ip route`?
4. Why is a full mesh required without route reflection or confederations?
5. For 20 iBGP speakers, how many unique sessions are required in a full mesh?
6. Which two attributes are added by route reflection for loop prevention?
7. What is the difference between ORIGINATOR_ID and CLUSTER_LIST?
8. Can a route reflector reflect a route learned from one non-client to another non-client?
9. Why can route reflection alter best-path decisions compared with a full mesh?
10. Which failures can loopback peering survive, and which can it not survive?
11. Why should infrastructure reachability remain in the IGP rather than be learned through the same iBGP session that depends on it?
12. When would two route reflectors share a cluster ID, and what is the trade-off?

# Instructor Checkpoints

| Checkpoint | Expected observation |
|---|---|
| Partial iBGP chain | R2 does not advertise R4's iBGP-learned prefix to R1 |
| Full mesh | Four routers require six unique iBGP sessions |
| External route before next-hop-self | Internal peers see an unresolved or unreachable next hop 192.0.2.2 |
| External route after next-hop-self | Internal peers resolve 10.255.0.1 through OSPF |
| Route reflection | Clients exchange routes through R2 without direct sessions |
| Reflected path | ORIGINATOR_ID identifies the originating client; CLUSTER_LIST identifies traversed reflector cluster(s) |
| Single-link failure | Loopback-sourced iBGP can survive if the IGP supplies an alternate path |
| Reflector failure | Client routes learned only through R2 are lost, exposing the single-reflector weakness |

# Cleanup State

At completion, save each running configuration. Export the final CML lab separately from the supplied starting topology so the original baseline remains reusable.
