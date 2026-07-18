# ACL Design — Inter-Site Access Control

This document explains the firewall ACL design that enforces department-level access policy between HQ and Branch. For raw verification output (show commands, ping test results), see [`doc/verifications/acl.md`](verifications/acl.md).

---

## Purpose

By default, OSPF makes every subnet in the network reachable from every other subnet — that's what routing is for. But TechCorp's security policy requires **department-level isolation** across sites: some departments should freely reach their counterpart site, others only their own department, and some not at all.

ACLs are applied on the WAN-facing interface (`Serial0/0/0`) of each router to enforce this policy at the point where inter-site traffic crosses from one site's network into the other's.

---

## Access policy

| Source | Can reach |
|--------|-----------|
| Branch IT (`10.1.10.0/24`) | Anything at HQ |
| HQ IT (`10.0.10.0/24`) | Anything at Branch |
| Branch HR (`10.1.20.0/24`) | HQ HR only (`10.0.20.0/24`) |
| HQ HR (`10.0.20.0/24`) | Branch HR only (`10.1.20.0/24`) |
| Branch Sales (`10.1.30.0/24`) | Nothing at HQ |
| HQ Sales (`10.0.30.0/24`) | Nothing at Branch |
| All data VLANs | Cannot Telnet (port 23) to any router, either site |

Two ACLs implement this — one per direction of travel, each applied **inbound**, as close as possible to where the traffic actually originates:

- **`BRANCH_TO_HQ`** — Branch Router, `FastEthernet0/0` (LAN-facing, toward the L3 switch), inbound
- **`HQ_TO_BRANCH`** — HQ Router, `FastEthernet0/0` (LAN-facing, toward the L3 switch), inbound

---

## Design decision: direction and placement

Standard practice for extended ACLs (ones matching both source and destination, like these) is to apply them **inbound, as close to the traffic's source as possible**. Two concrete reasons drive this:

1. **Router efficiency.** An inbound ACL is evaluated the instant a packet enters an interface — before any routing-table lookup or forwarding decision. If a packet is going to be denied, the router should find that out immediately, not after it's already done the work of routing the packet toward an outbound interface only to drop it there.
2. **Avoids wasted internal forwarding.** Filtering as close to the source as possible means denied traffic never travels any further into the router than necessary. Filtering it at the far side (e.g. the WAN interface) means it's already been forwarded internally for no reason.

**Why `FastEthernet0/0`, not `Serial0/0/0`?** "Closest to the source" depends on where each router actually sees the traffic first. Each site's L3 switch already handles all *intra-site* inter-VLAN routing internally via SVIs — so the only traffic that ever reaches the router through `FastEthernet0/0` (the link to the L3 switch) is traffic destined for the *remote* site. That makes `FastEthernet0/0` inbound the earliest point at which the router sees exactly the traffic this ACL needs to control, and nothing else.

**A placement that was tried and rejected:** applying the ACL inbound directly on `Serial0/0/0` (the WAN link) was the first attempt, but this interface faces the *remote* site — inbound there catches traffic *arriving from the other site*, not traffic leaving the local site. Since every rule in these ACLs uses the *local* site's subnets as the source address, an ACL applied this way would never match real traffic; it would sit there apparently active but not actually filtering anything meaningful. `Serial0/0/0` inbound was ruled out for this reason, and outbound on `Serial0/0/0` was tested as an intermediate fix — functionally correct, but not the most efficient placement, since it still lets denied traffic get routed internally before being dropped. `FastEthernet0/0` inbound was adopted as the final placement since it's both correct and efficient.

---

## Design decision: OSPF permit rule

OSPF hello packets are sent to the multicast address `224.0.0.5` (all OSPF routers), not to a specific unicast address. An early version of these ACLs used:

```
permit ospf host <local-WAN-IP> host <remote-WAN-IP>
```

This only matches unicast traffic between the two exact WAN interface addresses and never matches multicast hello traffic, so OSPF hellos fell through to the final `deny ip any any` — dropping the adjacency. The fix, used in both ACLs:

```
permit ospf any any
```

This permits all OSPF traffic without constraining source/destination, since OSPF's own protocol behavior (authentication, area matching, etc.) already governs which routers can legitimately form an adjacency — the ACL doesn't need to duplicate that job.

This rule is placed **first** in both ACLs so routing stability is never accidentally affected by rule ordering below it. With the ACL applied on `FastEthernet0/0` (see below), OSPF hello traffic between the two routers — which crosses `Serial0/0/0`, not `FastEthernet0/0` — isn't actually subject to this ACL at all anymore. The rule is kept anyway as a safeguard: if the ACL's placement is ever moved back onto a WAN-facing interface, OSPF adjacency won't silently break again.

---

## Design decision: DHCP relay permit placement (added during DHCP phase)

Once DHCP relay was configured (`ip helper-address` on every VLAN SVI, pointed at the HQ router's DHCP pools — see [`doc/dhcp-design.md`](dhcp-design.md)), relayed DHCP traffic began crossing the same `FastEthernet0/0 in` interface these ACLs already filter. This surfaced a real conflict between two existing design decisions:

- DHCP relay is unicast UDP (client port 68 → server port 67), sourced from each VLAN's SVI address.
- The Sales subnets (`10.0.30.0/24`, `10.1.30.0/24`) have a blanket `deny ip ... any` rule, by policy, positioned early in each ACL (rule 3 in the ordering below).

Since Sales' DHCP relay traffic is *also* sourced from a Sales subnet, it was being caught and dropped by that same deny rule — before ever reaching a DHCP permit line placed near the bottom of the list. Sales PCs could not obtain an address at all until this was fixed.

**Resolution:** the DHCP permit line was inserted **before** the Telnet denies and the Sales deny-all, near the very top of both ACLs (immediately after the OSPF permit). The reasoning: DHCP relay traffic is a request to the local network's own infrastructure service (address assignment), not the cross-site department traffic the Sales deny-all rule was written to block. Those are two different traffic classes that happen to share a source subnet, and the ACL needed to distinguish them explicitly rather than let the broader Sales rule shadow the narrower DHCP one.

```
15 permit udp any eq 68 any eq 67
```

This line is intentionally scoped to `any → any` on these specific ports rather than per-subnet, since every VLAN (not just Sales) needs its DHCP relay traffic to pass, and restricting it per-subnet would just recreate the same problem for each department individually.

---

## Rule ordering logic (updated)

Cisco ACLs evaluate top-to-bottom and stop at the first match, so order matters:

1. **`permit ospf any any`** — first, to guarantee routing is never impacted by anything below.
2. **`permit udp any eq 68 any eq 67`** (DHCP relay) — second, so every VLAN's address assignment works regardless of what department-level policy is enforced further down.
3. **Telnet denies per subnet** — before any broad permit, so management-plane access is blocked regardless of what data-plane access is later granted.
4. **Full subnet deny for blocked departments** (Sales) — before the general permits, so Sales traffic never reaches a later `permit ip` line.
5. **ICMP permits** — explicit, so ping tests can verify reachability independent of the broader IP permits.
6. **Scoped `permit ip`** — narrowest source/destination pairs allowed by policy (IT → any remote subnet, HR → remote HR only).
7. **`deny ip any any`** — explicit final catch-all. Functionally redundant (Cisco appends this implicitly), but written explicitly so `show access-lists` reports match counters for it, making unexpected/blocked traffic visible during testing.

---

## `BRANCH_TO_HQ` (applied on Branch Router, `FastEthernet0/0 in`)

```
ip access-list extended BRANCH_TO_HQ
 permit ospf any any
 permit udp any eq 68 any eq 67
 deny tcp 10.1.10.0 0.0.0.255 any eq 23
 deny tcp 10.1.20.0 0.0.0.255 any eq 23
 deny tcp 10.1.30.0 0.0.0.255 any eq 23
 deny ip 10.1.30.0 0.0.0.255 any
 permit icmp 10.1.10.0 0.0.0.255 any
 permit icmp 10.1.20.0 0.0.0.255 10.0.20.0 0.0.0.255
 permit ip 10.1.10.0 0.0.0.255 10.0.0.0 0.0.255.255
 permit ip 10.1.20.0 0.0.0.255 10.0.20.0 0.0.0.255
 deny ip any any
```

## `HQ_TO_BRANCH` (applied on HQ Router, `FastEthernet0/0 in`)

```
ip access-list extended HQ_TO_BRANCH
 permit ospf any any
 permit udp any eq 68 any eq 67
 deny tcp 10.0.10.0 0.0.0.255 any eq 23
 deny tcp 10.0.20.0 0.0.0.255 any eq 23
 deny tcp 10.0.30.0 0.0.0.255 any eq 23
 deny ip 10.0.30.0 0.0.0.255 any
 permit icmp 10.0.10.0 0.0.0.255 any
 permit icmp 10.0.20.0 0.0.0.255 10.1.20.0 0.0.0.255
 permit ip 10.0.10.0 0.0.0.255 10.1.0.0 0.0.255.255
 permit ip 10.0.20.0 0.0.0.255 10.1.20.0 0.0.0.255
 deny ip any any
```

---

## Testing approach

For each ACL, verification (documented separately in `doc/verifications/acl.md`) covers:

1. **OSPF stability** — `show ip ospf neighbor` on both routers confirms `FULL` state is unaffected by the ACL.
2. **Positive tests** — pings that should succeed per policy (e.g. Branch IT → HQ Sales host, HQ HR → Branch HR host).
3. **Negative tests** — pings that should fail per policy (e.g. Branch Sales → any HQ host, HQ Sales → any Branch host).
4. **Telnet block** — attempted Telnet from a data VLAN host to a router, confirming refusal.
5. **DHCP relay** — confirms DHCP requests from every VLAN, including Sales, are permitted through despite the Sales deny-all rule (see [`doc/verifications/dhcp.md`](verifications/dhcp.md)).
6. **Match counters** — `show access-lists` after testing, confirming traffic actually hit the expected rules rather than passing through unfiltered.