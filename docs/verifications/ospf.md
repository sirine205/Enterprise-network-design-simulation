# OSPF Verification — TechCorp Inc.

Verified on: HQ Router (Router ID 1.1.1.1)  
OSPF Process ID: 1  
Areas: 0 (backbone / WAN), 1 (HQ), 2 (Branch)

---

## 1. Neighbor adjacencies

```
Router# show ip ospf neighbor

Neighbor ID   Pri   State       Dead Time   Address        Interface
3.3.3.3       1     FULL/DR     00:00:38    10.0.0.2       FastEthernet0/0
2.2.2.2       —     FULL        —           10.255.255.2   Serial0/0/0
```

**What this confirms:**

- **3.3.3.3 (HQ L3 Switch) — FULL/DR on FastEthernet0/0**  
  The adjacency is fully established. The HQ L3 switch won the DR election on the FastEthernet segment — expected, since multi-access Ethernet segments elect a DR to reduce LSA flooding. The HQ router acts as BDR on this segment.

- **2.2.2.2 (Branch Router) — FULL on Serial0/0/0**  
  The WAN adjacency is also fully established. Serial point-to-point links skip the DR/BDR election entirely — there are only two routers on the link, so no election is needed. This is why the State column shows `FULL` without `/DR` or `/BDR`.

- **Log entry during verification:**  
  ```
  %OSPF-5-ADJCHG: Process 1, Nbr 2.2.2.2 on Serial0/0/0 from LOADING to FULL, Loading Done
  ```
  This confirms the Branch adjacency completed its final LSA exchange and reached FULL state while the verification was running.

---

## 2. OSPF routing table

```
Router# show ip route ospf

10.0.0.0/8 is variably subnetted, 11 subnets, 3 masks
O     10.0.10.0  [110/2]  via 10.0.0.2,     00:04:20, FastEthernet0/0
O     10.0.20.0  [110/2]  via 10.0.0.2,     00:04:20, FastEthernet0/0
O     10.0.30.0  [110/2]  via 10.0.0.2,     00:04:20, FastEthernet0/0
O IA  10.1.0.0   [110/65] via 10.255.255.2,  00:00:36, Serial0/0/0
O IA  10.1.10.0  [110/66] via 10.255.255.2,  00:00:24, Serial0/0/0
O IA  10.1.20.0  [110/66] via 10.255.255.2,  00:00:24, Serial0/0/0
O IA  10.1.30.0  [110/66] via 10.255.255.2,  00:00:24, Serial0/0/0
```

**What this confirms:**

- **HQ VLANs — intra-area (`O`), cost 2**  
  Routes to 10.0.10.0, 10.0.20.0, and 10.0.30.0 are learned within Area 1 via the HQ L3 switch (10.0.0.2). Cost 2 = FastEthernet reference bandwidth cost (1) from the HQ router to the L3 switch, plus the cost of the SVI itself. These are standard intra-area routes.

- **Branch VLANs — inter-area (`O IA`), cost 65–66**  
  Routes to Branch subnets are marked `O IA` because they cross an area boundary — they originate in Area 2 and are summarised by the Branch ABR into Area 0, then re-advertised into Area 1 as Summary LSAs. The higher cost (65/66) reflects the serial WAN link's default cost (64 for serial vs 1 for FastEthernet) plus the additional hop.

- **Cost difference between 10.1.0.0 (65) and 10.1.10.0–10.1.30.0 (66)**  
  The summary route for the entire Branch block has cost 65; the individual VLAN routes cost 66 because they include one more intra-area hop inside Area 2 (the Branch L3 switch).

---

## 3. OSPF database

```
Router# show ip ospf database

OSPF Router with ID (1.1.1.1) (Process ID 1)

        Router Link States (Area 0)

Link ID       ADV Router    Age    Seq#        Checksum    Link count
1.1.1.1       1.1.1.1       175    0x80000003  0x002a77    2
2.2.2.2       2.2.2.2       105    0x80000003  0x00c9d2    2

        Summary Net Link States (Area 0)

Link ID       ADV Router    Age    Seq#        Checksum
10.0.0.0      1.1.1.1       333    0x80000001  0x00e072
10.0.10.0     1.1.1.1       313    0x80000002  0x008cb7
10.0.20.0     1.1.1.1       313    0x80000003  0x001c1d
10.0.30.0     1.1.1.1       313    0x80000004  0x00ab82
10.1.0.0      2.2.2.2       95     0x80000001  0x00b697
10.1.10.0     2.2.2.2       83     0x80000002  0x0062dc
10.1.20.0     2.2.2.2       83     0x80000003  0x00f142
10.1.30.0     2.2.2.2       83     0x80000004  0x0081a7

        Router Link States (Area 1)

Link ID       ADV Router    Age    Seq#        Checksum    Link count
1.1.1.1       1.1.1.1       318    0x80000002  0x0077bc    1
3.3.3.3       3.3.3.3       318    0x80000016  0x00552d    4

        Net Link States (Area 1)

Link ID       ADV Router    Age    Seq#        Checksum
10.0.0.2      3.3.3.3       318    0x80000001  0x00270d

        Summary Net Link States (Area 1)

Link ID       ADV Router    Age    Seq#        Checksum
10.255.255.0  1.1.1.1       333    0x80000001  0x0059ba
0.0.0.0       1.1.1.1       328    0x80000002  0x0073e5
10.1.0.0      1.1.1.1       90     0x80000003  0x0053bc
10.1.10.0     1.1.1.1       78     0x80000004  0x00fe02
10.1.20.0     1.1.1.1       78     0x80000005  0x008f66
10.1.30.0     1.1.1.1       78     0x80000006  0x001ecc
```

**What this confirms:**

- **Area 0 (backbone)** contains Router LSAs for both ABRs (1.1.1.1 and 2.2.2.2) and Summary LSAs for all subnets being redistributed between areas. Each ABR advertises its own area's subnets into the backbone so the other side can learn them.

- **Area 1 (HQ)** contains Router LSAs for 1.1.1.1 (HQ router) and 3.3.3.3 (HQ L3 switch). The Network LSA advertised by 3.3.3.3 corresponds to the DR on the HQ multi-access segment. Summary LSAs injected by the ABR (1.1.1.1) bring in the WAN link, all Branch subnets, and a default route (0.0.0.0).

- **LSA sequence numbers** (0x80000002, 0x80000003, etc.) show these are early in the adjacency lifetime — the network was recently converged, consistent with the adjacency log message seen during verification.