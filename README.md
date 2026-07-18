# TechCorp Inc. — Enterprise Network Simulation

A fully documented, multi-site enterprise network built from scratch in Cisco Packet Tracer. Covers IP planning, VLAN segmentation, inter-VLAN routing, multi-area OSPF, DHCP/DNS, firewall ACLs, and traffic analysis.

---

## What this project is

TechCorp operates two sites — HQ and Branch — each with three departments: IT, HR, and Sales. The network enforces department isolation via VLANs, routes between sites using multi-area OSPF, and controls cross-department access via firewall ACLs.

Every design document explains the *why* behind each decision, not just what was configured.

---

## Repository structure

```
.
├── config/                       # Cisco IOS configuration files per device
├── diagrams/                     # Topology diagram (draw.io) — in progress
├── wireshark/                    # Packet captures — in progress
└── doc/
    ├── ip-plan.md                # Full IP addressing and VLAN table
    ├── requirements.md           # Project requirements
    ├── vlan-design.md            # VLAN & inter-VLAN routing design rationale
    ├── acl-design.md             # Firewall ACL design rationale
    ├── dhcp-design.md            # DHCP design rationale
    └── verifications/
        ├── ospf.md               # OSPF verification — show outputs + analysis
        ├── acl.md                # ACL verification — show outputs + analysis
        └── dhcp.md               # DHCP verification — show outputs + analysis
```

---

## Progress

| Phase                      | Status         | Reference                          |
|----------------------------|----------------|-------------------------------------|
| IP addressing              | ✅ Complete    | `doc/ip-plan.md`                   |
| VLAN design & SVIs         | ✅ Complete    | `doc/vlan-design.md`               |
| WAN serial link            | ✅ Complete    | `config/`                          |
| Multi-area OSPF            | ✅ Complete    | `doc/verifications/ospf.md`        |
| Firewall ACLs              | ✅ Complete    | `doc/acl-design.md` `doc/verifications/acl.md` |
| DHCP                       | ✅ Complete    | `doc/dhcp-design.md` `doc/verifications/dhcp.md` |
| DNS                        | 🔲 Planned     |                                     |
| Wireshark captures         | 🔲 Planned     | `wireshark/`                       |
| Topology diagram (draw.io) | 🔲 Planned     | `diagrams/`                        |

---

## Design summary

**Sites:** HQ (`10.0.x.x`) and Branch (`10.1.x.x`), connected via a WAN serial link (`10.255.255.0/30`).

**VLANs:** Three per site — IT, HR, Sales. VLAN IDs encode site and department so a single number tells you both (e.g. VLAN 120 = Branch HR). See [`doc/vlan-design.md`](doc/vlan-design.md) for the full rationale.

**Routing:** Each site uses a Layer 3 switch with SVIs for hardware-speed inter-VLAN routing. OSPF runs in three areas — backbone Area 0 on the WAN link, Area 1 for HQ, Area 2 for Branch. See [`doc/verifications/ospf.md`](doc/verifications/ospf.md) for confirmed adjacencies and routing table output.

**Access control:** Firewall ACLs enforce department-level access policy between sites — some departments can reach anything remote, others only their counterpart department, and some nothing at all. ACLs are applied inbound on each router's LAN-facing interface, closest to the traffic's source. See [`doc/acl-design.md`](doc/acl-design.md) for the design rationale and [`doc/verifications/acl.md`](doc/verifications/acl.md) for tested proof.

**DHCP:** Centralized DHCP service on the HQ router, with per-VLAN pools for all six VLANs across both sites. Since DHCP requests are broadcasts and can't cross VLAN or router boundaries on their own, each VLAN's SVI uses `ip helper-address` to relay requests to the HQ router. See [`doc/dhcp-design.md`](doc/dhcp-design.md) for the design rationale and [`doc/verifications/dhcp.md`](doc/verifications/dhcp.md) for tested proof.

---

## Tools

- Cisco Packet Tracer 8.x
- Cisco IOS (router and multilayer switch CLI)
- Wireshark (planned)
- draw.io (planned)