# DHCP Design — Centralized DHCP with Relay

This document explains the DHCP design used across both sites. For raw verification output (bindings, match counters), see [`doc/verifications/dhcp.md`](verifications/dhcp.md).

---

## Purpose

Every VLAN needs automatic IP address assignment for end hosts, rather than manual static configuration on every PC. This phase implements DHCP for all six VLANs across HQ and Branch.

---

## Design decision: centralized DHCP vs. per-VLAN or per-site servers

Three approaches were considered:

1. **A DHCP server per VLAN** — six separate services to configure and maintain. Simple conceptually, but doesn't reflect how this is usually done in practice, and adds unnecessary device count for a simulation of this scale.
2. **A DHCP server per site** — two services instead of six, but still duplicates configuration and doesn't demonstrate DHCP relay across router boundaries, which is a core interview-relevant skill.
3. **A single centralized DHCP service, on the HQ router, serving all six VLANs at both sites** — chosen. One set of pools to maintain, and it forces (and demonstrates) proper use of DHCP relay across both VLAN boundaries at HQ and the WAN link to Branch.

Centralized DHCP on the HQ router was selected. This mirrors common real-world enterprise design (centralize infrastructure services, relay to reach them) and gives the strongest portfolio/interview value, since it requires correctly solving the broadcast-domain problem below rather than sidestepping it.

---

## The broadcast-domain problem, and why relay is needed

DHCP clients discover a server by broadcasting a `DHCPDISCOVER` message. Broadcasts are, by definition, confined to the local broadcast domain (VLAN) — they do not cross VLAN boundaries or get routed. With the DHCP server centralized on the HQ router, every VLAN except the one directly attached to that router would never have its broadcast reach the server.

`ip helper-address`, configured on each VLAN's SVI, solves this: the L3 switch listens for DHCP broadcasts on that VLAN, and converts each one into a normal unicast IP packet addressed to the configured helper IP. This unicast packet is then routed normally — via OSPF, across VLAN boundaries and the WAN link — to the HQ router, which processes it and replies through the same relay path in reverse.

---

## Design decision: what IP to target with `ip helper-address`

Two candidate targets were tested for the helper-address on each SVI:

1. **A nearby router's directly-connected interface** (e.g. pointing Branch's helper-address at the Branch router's own LAN-facing interface) — **tried and rejected**. The Branch router doesn't run any DHCP pools itself (all pools live on the HQ router), so relayed packets arriving there had nowhere to go. `ip helper-address` relays to the configured IP only — it does not automatically re-relay across additional router hops on its own; each hop would need its own relay configuration, which defeats the point of centralizing DHCP in the first place.
2. **The actual DHCP server's real, OSPF-reachable interface IP** (`10.0.0.1`, the HQ router's LAN-facing interface) — **adopted**. Since OSPF already provides full routing reachability from every VLAN subnet to this address, the relay packet is treated as an ordinary routed unicast packet from the point it leaves the local switch, needing no further relay configuration at any intermediate hop.

**A related early mistake:** the OSPF router-id (`1.1.1.1`) was briefly tried as the helper-address target, on the assumption it referred to a loopback interface. It does not — no `Loopback0` interface exists on the HQ router, so `1.1.1.1` is a router-id used only for OSPF process identification, not a routable address. Helper-address must point at a real, configured, OSPF-advertised interface.

---

## Design decision: ACL interaction with DHCP relay

Once relay was working, DHCP requests began arriving at each router's `FastEthernet0/0` inbound — the same interface where `HQ_TO_BRANCH` and `BRANCH_TO_HQ` are already applied (see [`doc/acl-design.md`](acl-design.md)). This surfaced two issues, both resolved by adding and correctly ordering a DHCP permit rule:

1. Neither ACL had any rule permitting DHCP traffic (UDP 67/68) at all, so every VLAN's requests were silently dropped by the final deny-all until an explicit permit was added.
2. Once added, the permit rule was initially placed near the bottom of the ACL — after the Sales deny-all rule. Since Sales' relayed DHCP request is sourced from the Sales subnet, it matched the deny-all rule first and never reached the DHCP permit. The fix was reordering the DHCP permit to sit near the top of both ACLs, ahead of the Sales deny-all, on the reasoning that DHCP relay is a local infrastructure service request, not the cross-site department traffic that rule was designed to block.

Full detail on this interaction is in `acl-design.md`'s "Design decision: DHCP relay permit placement" section, since it's fundamentally an ACL design decision, not a DHCP one — DHCP relay behaved correctly throughout; it was the ACL that needed updating to account for a new traffic type it hadn't previously needed to classify.

---

## DHCP pool configuration (on HQ Router)

One pool per VLAN, all hosted on the HQ router regardless of which site the VLAN belongs to:

```
ip dhcp excluded-address 10.0.10.1
ip dhcp excluded-address 10.0.20.1
ip dhcp excluded-address 10.0.30.1
ip dhcp excluded-address 10.1.10.1
ip dhcp excluded-address 10.1.20.1
ip dhcp excluded-address 10.1.30.1

ip dhcp pool HQ-IT
 network 10.0.10.0 255.255.255.0
 default-router 10.0.10.1
 dns-server 8.8.8.8

ip dhcp pool HQ-HR
 network 10.0.20.0 255.255.255.0
 default-router 10.0.20.1
 dns-server 8.8.8.8

ip dhcp pool HQ-SALES
 network 10.0.30.0 255.255.255.0
 default-router 10.0.30.1
 dns-server 8.8.8.8

ip dhcp pool BRANCH-IT
 network 10.1.10.0 255.255.255.0
 default-router 10.1.10.1
 dns-server 8.8.8.8

ip dhcp pool BRANCH-HR
 network 10.1.20.0 255.255.255.0
 default-router 10.1.20.1
 dns-server 8.8.8.8

ip dhcp pool BRANCH-SALES
 network 10.1.30.0 255.255.255.0
 default-router 10.1.30.1
 dns-server 8.8.8.8
```

Each `excluded-address` protects that VLAN's gateway (`.1`, the SVI address) from being handed out to a client — consistent with the project's [gateway convention](ip-plan.md#gateway-convention).

**Note on `dns-server 8.8.8.8`:** this is a placeholder. DNS is a separate, not-yet-implemented phase (see project README). Once an internal DNS server is deployed, every pool's `dns-server` line will be updated to point at that server's internal address instead of Google's public DNS, which is not reachable in this simulated topology anyway.

---

## Relay configuration (per site, on each L3 switch)

**HQ L3 switch** — all three SVIs relay to the HQ router's own LAN-facing interface, since the server is local to this site:
```
interface vlan 10
 ip helper-address 10.0.0.1
interface vlan 20
 ip helper-address 10.0.0.1
interface vlan 30
 ip helper-address 10.0.0.1
```

**Branch L3 switch** — all three SVIs relay directly to the HQ router's interface as well, crossing the WAN link via existing OSPF routes:
```
interface vlan 110
 ip helper-address 10.0.0.1
interface vlan 120
 ip helper-address 10.0.0.1
interface vlan 130
 ip helper-address 10.0.0.1
```

---

## Testing approach

Verification (documented in `doc/verifications/dhcp.md`) covers:

1. **Per-VLAN address assignment** — a test PC on each of the six VLANs set to DHCP, confirming it receives an address in the correct subnet with the correct gateway.
2. **`show ip dhcp binding`** on the HQ router — confirms all six VLANs are represented with correctly-scoped leases.
3. **ACL match counters** — confirms DHCP relay traffic is hitting the expected permit rule on both ACLs, including for Sales specifically.