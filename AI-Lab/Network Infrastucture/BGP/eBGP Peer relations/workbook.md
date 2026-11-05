# CCIE Workbook: BGP eBGP Peer Relations

## Lab purpose

This lab isolates the mechanics of an external BGP neighbor relationship. The topology is deliberately small, but the tasks force you to distinguish transport reachability, TCP source and destination addresses, BGP AS validation, TTL behavior, authentication, address-family activation, and control-plane verification.

Do not add an IGP. Use only the supplied connected routes and host routes unless a task explicitly instructs otherwise.

## Topology

```text
                      R2, AS 65002
                  Gi0/0       Gi0/1
                    /           \
          10.12.0.0/30       10.23.0.0/30
                  /               \
        Gi0/0 R1 ---------------- R3 Gi0/0
        AS 65001      10.13.0.0/30    AS 65003
               Gi0/1             Gi0/1
```

| Router | AS | Loopback0 | Link addresses |
|---|---:|---|---|
| R1 | 65001 | 10.255.1.1/32 | Gi0/0 10.12.0.1/30, Gi0/1 10.13.0.1/30 |
| R2 | 65002 | 10.255.2.2/32 | Gi0/0 10.12.0.2/30, Gi0/1 10.23.0.1/30 |
| R3 | 65003 | 10.255.3.3/32 | Gi0/0 10.23.0.2/30, Gi0/1 10.13.0.2/30 |

IPv6 uses the matching `2001:db8:<link>::/64` prefixes and `2001:db8:255:<router>::<router>/128` loopbacks.

## Rules of engagement

- Begin each section from the configuration produced by the preceding section unless told to remove it.
- Do not use peer-groups, templates, dynamic neighbors, or route reflectors.
- Record the BGP finite-state-machine state before and after every correction.
- Prefer targeted commands such as `show bgp ipv4 unicast neighbors x.x.x.x` over broad output.
- Use `clear bgp ...` only when the relevant change does not trigger a session reset automatically.

## Task 1: Establish a directly connected eBGP session

Configure an IPv4 unicast eBGP relationship between R1 and R2 using their connected 10.12.0.0/30 addresses.

Requirements:

1. Use the router AS numbers in the topology table.
2. Set the BGP router ID manually to each router's Loopback0 IPv4 address.
3. Ensure the neighbor is active in the IPv4 unicast address family.
4. Do not advertise any prefixes yet.
5. Verify that the session reaches `Established` and that the neighbor table shows zero received prefixes.

Answer these questions:

- Which endpoint initiates the TCP connection?
- Which local and foreign TCP ports are visible after establishment?
- What TTL is sent by default for this directly connected eBGP session?
- Why can the session be established without an IGP or static route?

Suggested verification:

```text
show bgp ipv4 unicast summary
show bgp ipv4 unicast neighbors 10.12.0.2
show tcp brief all | include 179
show ip route 10.12.0.2
```

## Task 2: Prove the neighbor's source-address requirement

On R2, temporarily change the neighbor address so that R2 expects R1's Loopback0 address, while R1 still sources BGP from 10.12.0.1.

1. Predict the BGP state on both routers before making the change.
2. Observe the TCP attempts and the BGP state transitions.
3. Restore the correct connected neighbor address.
4. Explain why ordinary IP reachability to 10.255.1.1 is not enough for this mismatched configuration.

Capture evidence with:

```text
show bgp ipv4 unicast summary
show bgp ipv4 unicast neighbors
show tcp brief all
show logging | include BGP|TCP
```

## Task 3: Diagnose remote-AS failures

Break the R1-R2 session by configuring the wrong remote AS on R2.

1. Use remote AS 65111 on R2.
2. Identify the BGP notification and subcode.
3. Determine whether TCP completes before BGP rejects the peer.
4. Correct the remote AS and verify recovery.

Your explanation must distinguish TCP establishment from BGP OPEN-message validation.

## Task 4: Configure a loopback-to-loopback eBGP session

Build a second eBGP relationship directly between R1 and R3, using only Loopback0 addresses as the BGP endpoints.

The supplied host routes provide bidirectional reachability across the R1-R3 physical link.

Requirements:

1. R1 must peer with 10.255.3.3 in AS 65003.
2. R3 must peer with 10.255.1.1 in AS 65001.
3. Both routers must source BGP from Loopback0.
4. Configure the minimum eBGP multihop TTL that allows the session to establish.
5. Do not change the host routes.

Before adding the multihop command, observe the failed state and explain why successful ICMP echo tests do not prove that eBGP's TTL requirement is satisfied.

Verification:

```text
ping 10.255.3.3 source 10.255.1.1
traceroute 10.255.3.3 source 10.255.1.1
show bgp ipv4 unicast summary
show bgp ipv4 unicast neighbors 10.255.3.3 | include Local host|Foreign host|TTL
show ip route 10.255.3.3
```

## Task 5: Calculate and test the eBGP multihop TTL

On the R1-R3 loopback session:

1. Test `ebgp-multihop 1`, then `ebgp-multihop 2`, and record the result.
2. Shut R1 Gi0/1 and replace the R1 and R3 host routes so the loopback traffic traverses R2.
3. Do not configure BGP on R2 for this task. Add only the minimum static routes needed on R2 to forward traffic between 10.255.1.1 and 10.255.3.3.
4. Determine the new minimum multihop value.
5. Explain the TTL calculation from the source loopback to the destination loopback.
6. Restore Gi0/1, the original host routes, and the minimum setting for the direct R1-R3 path.

Success requires both the ICMP test and the BGP session to pass. Document why these tests can produce different results.

## Task 6: Replace multihop with Generalized TTL Security Mechanism

Protect the R1-R3 loopback session with TTL security.

1. Remove `ebgp-multihop` from both ends.
2. Configure `ttl-security hops 1` on both ends while the direct R1-R3 physical path is in use.
3. Verify the received TTL threshold in the neighbor detail.
4. Prove that `ebgp-multihop` and `ttl-security` are not intended to remain configured together for the same neighbor.
5. Change one side to `ttl-security hops 2`, leave the other at 1, and determine whether the session remains established.

Explain why TTL security checks received packets near TTL 255, whereas ordinary eBGP multihop limits the sender's initial TTL.

## Task 7: Add TCP MD5 authentication and analyze failure behavior

Secure the R1-R2 connected session with the password `CCIE-EBGP-12`.

1. Configure the password on R1 only and observe the result.
2. Identify whether an unmatched password produces a BGP notification.
3. Configure the matching password on R2.
4. Replace R2's password with `CCIE-EBGP-21`, observe, then restore the correct value.
5. Confirm that the final session is established.

Do not expose the password in production-style command output saved for review.

## Task 8: Control session initiation with transport connection-mode

On the R1-R2 neighbor relationship:

1. Configure R1 as passive for TCP connection initiation.
2. Reset the session and prove that R2 initiates it.
3. Configure R2 as passive as well.
4. Predict and observe the resulting BGP state.
5. Restore R2 to the default active behavior and verify recovery.

Explain why `transport connection-mode passive` does not mean that a router refuses incoming BGP sessions.

## Task 9: Build and troubleshoot IPv6 eBGP

Create an IPv6 unicast eBGP session between R2 and R3 using their global connected addresses on `2001:db8:23::/64`.

Requirements:

1. Retain the same AS numbers.
2. Activate the neighbor only under the IPv6 unicast address family.
3. Do not advertise any IPv6 prefixes yet.
4. Verify that IPv4 and IPv6 neighbor activation are independent.
5. Deliberately omit IPv6 address-family activation on one side and record the state and messages.
6. Correct the configuration and verify zero received IPv6 prefixes.

Verification:

```text
show bgp ipv6 unicast summary
show bgp ipv6 unicast neighbors 2001:db8:23::2
show ipv6 route 2001:db8:23::2
show tcp brief all
```

## Task 10: Advertise test prefixes and interpret next hop

After all three required sessions are stable, originate only the following loopbacks:

- R1 originates 10.255.1.1/32 toward R2.
- R2 originates 10.255.2.2/32 toward R1 and 2001:db8:255:2::2/128 toward R3.
- R3 originates 10.255.3.3/32 toward R1 and 2001:db8:255:3::3/128 toward R2.

Requirements:

1. Use `network` statements. Do not redistribute connected or static routes.
2. Confirm that every network statement has an exact route in the local RIB.
3. Record the BGP next hop seen at each external neighbor.
4. Explain why an eBGP speaker normally rewrites the next hop when advertising to an eBGP peer.
5. Confirm that the AS_PATH begins with the advertising neighbor's AS.

## Task 11: Failure-injection matrix

For each fault below, predict the state, inject the fault, collect evidence, and restore service before continuing.

| Fault | Session | Evidence to collect |
|---|---|---|
| Shut the physical interface | R1-R2 | BGP state, TCP state, interface state |
| Remove the route to remote loopback | R1-R3 | RIB lookup, TCP attempts, BGP state |
| Wrong remote AS | R1-R2 | BGP notification, OPEN validation |
| Wrong MD5 password | R1-R2 | TCP behavior, logs, BGP state |
| Remove `update-source Loopback0` on one end | R1-R3 | Source address, TCP behavior |
| Set insufficient multihop TTL | R1-R3 | Packet TTL reasoning, BGP state |
| Deactivate IPv6 address family on one side | R2-R3 | IPv6 AF summary and neighbor detail |
| Make both peers connection-mode passive | R1-R2 | TCP initiation attempts and BGP state |

For each row, state whether the failure occurs at Layer 1/2, IP reachability, TCP, BGP OPEN validation, or address-family negotiation.

## Task 12: Final-state requirements

Leave the lab in this state:

- R1-R2 connected IPv4 eBGP is established, authenticated, and R1 is passive for initiation.
- R1-R3 loopback IPv4 eBGP is established with `update-source Loopback0` and TTL security for the direct R1-R3 path.
- R2-R3 connected IPv6 eBGP is established.
- Only the prefixes listed in Task 10 are originated.
- No IGP or redistribution is configured.
- No neighbor remains shutdown.

Collect this final command set from every applicable router:

```text
show ip interface brief
show ipv6 interface brief
show ip route static
show ipv6 route static
show bgp ipv4 unicast summary
show bgp ipv6 unicast summary
show bgp ipv4 unicast neighbors
show bgp ipv6 unicast neighbors
show tcp brief all
show running-config | section router bgp
```

## Mastery checks

A successful student can answer all of these without guessing:

1. What three conditions must be true before a BGP OPEN message can be exchanged?
2. Why does a directly connected eBGP neighbor normally not require `ebgp-multihop`?
3. Why does loopback peering normally require both `update-source` and a TTL adjustment?
4. What exactly does `remote-as` validate?
5. At what stage does an MD5 mismatch prevent progress?
6. How is GTSM different from simply increasing eBGP multihop?
7. Why can one address family be operational while another is inactive?
8. Which commands prove the configured neighbor address, actual TCP endpoints, negotiated AS, hold time, TTL behavior, and received prefixes?
