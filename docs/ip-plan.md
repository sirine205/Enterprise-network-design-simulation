# IP Addressing Plan — TechCorp Inc.

## Address space

The network uses private RFC 1918 space, split by site:

| Block          | Assigned to       |
|----------------|-------------------|
| `10.0.0.0/8`   | Entire project    |
| `10.0.x.x`     | HQ                |
| `10.1.x.x`     | Branch            |
| `10.255.255.x` | WAN serial link   |

Each site-department combination gets a `/24`. This is deliberately generous for a simulation — it keeps subnets easy to read and leaves room to add hosts without redesigning the plan.

---

## Full addressing table

| Site              | VLAN | Department  | Network         | Gateway                                  |
|-------------------|------|-------------|-----------------|-------------------------------------------|
| HQ                | 10   | IT          | 10.0.10.0/24    | 10.0.10.1                                |
| HQ                | 20   | HR          | 10.0.20.0/24    | 10.0.20.1                                |
| HQ                | 30   | Sales       | 10.0.30.0/24    | 10.0.30.1                                |
| Branch            | 110  | IT          | 10.1.10.0/24    | 10.1.10.1                                |
| Branch            | 120  | HR          | 10.1.20.0/24    | 10.1.20.1                                |
| Branch            | 130  | Sales       | 10.1.30.0/24    | 10.1.30.1                                |
| WAN (HQ ↔ Branch) | —    | Router link | 10.255.255.0/30 | HQ: 10.255.255.1 / Branch: 10.255.255.2  |

---

## VLAN numbering convention

VLAN IDs are not arbitrary:

- The **hundreds digit** identifies the site: `0xx` = HQ, `1xx` = Branch
- The **tens digit** identifies the department: `x1x` = IT, `x2x` = HR, `x3x` = Sales

So VLAN 120 immediately reads as Branch HR, and VLAN 30 as HQ Sales. This makes `show vlan brief` output self-explanatory without needing a reference sheet.

---

## Gateway convention

Each gateway is the `.1` address of its subnet. This is a convention, not a technical requirement, but it's universal — any engineer reading the config will expect it.

---

## OSPF Router IDs

| Device          | Router ID |
|-----------------|-----------|
| HQ Router       | 1.1.1.1   |
| Branch Router   | 2.2.2.2   |
| HQ L3 Switch    | 3.3.3.3   |

Router IDs are assigned statically as loopback-style addresses to ensure stability — an OSPF router ID based on a physical interface IP can change if that interface goes down.