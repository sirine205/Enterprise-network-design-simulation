# Project Requirements — TechCorp Inc.

## Scenario

TechCorp Inc. is a company with two physical sites — a headquarters (HQ) and a branch office — each housing three departments: IT, HR, and Sales. The network must reflect real-world enterprise design constraints: departments must not freely reach each other, sites must communicate over a WAN link, and the network must be manageable and scalable.

---

## Requirements

### 1. Department isolation
Each department must be segmented into its own VLAN so that broadcast traffic stays contained and access between departments can be controlled explicitly.

**Implementation:** VLANs per department at each site, with Layer 3 SVIs providing routed boundaries between them.

---

### 2. Inter-site communication
HQ and Branch must be able to route traffic to each other across a simulated WAN link.

**Implementation:** Serial WAN link between HQ and Branch routers, with multi-area OSPF providing dynamic route exchange.

---

### 3. Automatic IP assignment
End devices must receive IP configuration automatically — no static assignment per host.

**Implementation:** DHCP server(s) per site, scoped per VLAN. *(Planned)*

---

### 4. Internal name resolution
Devices must be reachable by hostname within the network, not just by IP.

**Implementation:** Internal DNS server. *(Planned)*

---

### 5. Controlled access between departments
Not all inter-department traffic should be permitted. Access rules must reflect business logic (e.g. Sales cannot reach IT servers directly).

**Implementation:** ACLs applied at the Layer 3 switch or router level. *(Planned)*