# VLAN & Inter-VLAN Routing Design — TechCorp Inc.

## 1. Why VLANs?

Without VLANs, every device on a switch shares the same broadcast domain. An ARP broadcast from a Sales laptop reaches every IT server — unnecessary traffic, and a security exposure before a single firewall rule has been written.

VLANs fix this by creating logically separate networks on the same physical switch. A broadcast in VLAN 10 stays in VLAN 10. Devices in other VLANs never see it.

For TechCorp, three departments require isolation for different reasons:

| Department | Why it needs isolation |
|------------|------------------------|
| IT         | Manages infrastructure — must not be reachable from Sales by default |
| HR         | Handles sensitive employee data — isolated from other departments |
| Sales      | Higher risk of phishing/malware — contained if compromised |

---

## 2. VLAN table

| Site   | VLAN ID | Department | Purpose                        |
|--------|---------|------------|--------------------------------|
| HQ     | 10      | IT         | Servers, workstations, mgmt    |
| HQ     | 20      | HR         | HR systems, employee data      |
| HQ     | 30      | Sales      | CRM, sales tools               |
| Branch | 110     | IT         | Local IT, remote mgmt          |
| Branch | 120     | HR         | HR terminals                   |
| Branch | 130     | Sales      | Sales floor devices            |

VLAN IDs follow a deliberate convention: the hundreds digit identifies the site (0 = HQ, 1 = Branch) and the tens digit identifies the department (1 = IT, 2 = HR, 3 = Sales). A VLAN ID alone tells you where and what — useful when reading `show vlan brief` output during troubleshooting.

---

## 3. Inter-VLAN routing — Layer 3 switch with SVIs

Two approaches were considered.

### Option A — Router-on-a-stick (rejected)

One physical link from switch to router, with logical sub-interfaces per VLAN. All inter-VLAN traffic funnels through that one cable.

**Problem:** A single bottleneck link. If IT and HR both generate significant traffic toward each other, you need double the bandwidth on one uplink. Routing also happens in software on the router CPU rather than in dedicated hardware.

### Option B — Layer 3 switch with SVIs (chosen)

A Layer 3 switch has routing capability built into dedicated hardware (ASICs). Each VLAN gets a Switched Virtual Interface (SVI) — a virtual Layer 3 interface that acts as the default gateway for that VLAN. Routing between VLANs happens at line speed inside the switch.

**Why it's better:**
- No single bottleneck link
- Hardware-speed routing (ASICs, not CPU)
- Cleaner topology — one device handles both switching and routing at the core
- Scales easily — adding a VLAN is just adding an SVI

---

## 4. SVI gateway assignments

Each gateway is the `.1` address of its subnet — a convention, not a technical requirement, but universal enough that any engineer reading the config will expect it.

### HQ Layer 3 switch

| VLAN | SVI IP        | Subnet mask     | Gateway for          |
|------|---------------|-----------------|----------------------|
| 10   | 10.0.10.1     | 255.255.255.0   | All HQ IT devices    |
| 20   | 10.0.20.1     | 255.255.255.0   | All HQ HR devices    |
| 30   | 10.0.30.1     | 255.255.255.0   | All HQ Sales devices |

### Branch Layer 3 switch

| VLAN | SVI IP        | Subnet mask     | Gateway for              |
|------|---------------|-----------------|--------------------------|
| 110  | 10.1.10.1     | 255.255.255.0   | All Branch IT devices    |
| 120  | 10.1.20.1     | 255.255.255.0   | All Branch HR devices    |
| 130  | 10.1.30.1     | 255.255.255.0   | All Branch Sales devices |

---

## 5. Port types — access vs trunk

Every port on every switch is one of two types.

### Access port
- Belongs to exactly one VLAN
- Connects to end devices: PCs, printers, IP phones
- Frames leaving the port are untagged — the end device sees a normal Ethernet frame and has no awareness of VLANs
- Inbound frames are tagged by the switch with the port's assigned VLAN

### Trunk port
- Carries multiple VLANs simultaneously
- Connects switch-to-switch or switch-to-router
- Uses 802.1Q tagging: a 4-byte tag inserted into each frame identifies the VLAN
- Required wherever traffic from more than one VLAN needs to cross the same cable

Rule of thumb: if more than one VLAN needs to cross a link, it's a trunk.

### Port assignments — HQ

| Connection                          | Port type | VLAN(s)    |
|-------------------------------------|-----------|------------|
| L3 Switch → L2 Switch (access wing) | Trunk     | 10, 20, 30 |
| L2 Switch → IT PC                   | Access    | 10         |
| L2 Switch → HR PC                   | Access    | 20         |
| L2 Switch → Sales PC                | Access    | 30         |
| L3 Switch → WAN Router              | Trunk     | all        |

---

## 6. Design decisions summary

| Decision           | Choice                   | Reason                                               |
|--------------------|--------------------------|------------------------------------------------------|
| Routing method     | L3 switch SVIs           | Hardware speed, no bottleneck, cleaner topology      |
| VLAN numbering     | Site-aware (x10/x20/x30) | Self-documenting, consistent, faster to troubleshoot |
| Gateway convention | .1 address of each subnet | Universal standard, expected by any engineer reading the config |
| Inter-site link    | Trunk                    | Carries all VLANs between HQ and Branch              |