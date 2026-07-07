# ACL Verification — HQ_TO_BRANCH & BRANCH_TO_HQ

Proof that both inter-site ACLs are correctly placed, do not interfere with OSPF, and enforce the access policy defined in [`doc/acl-design.md`](../acl-design.md). This document records `show` command output and ping test results gathered during testing, including two issues found and fixed along the way.

---

## 1. ACL placement

Both ACLs must be bound **inbound** on each router's `FastEthernet0/0` (LAN-facing, toward the local L3 switch) and **not bound at all** on `Serial0/0/0` (the WAN link) — per the design rationale in `acl-design.md`.

**Branch Router:**
```
Router#show ip interface fastEthernet0/0
...
Outgoing access list is not set
Inbound access list is BRANCH_TO_HQ
```
```
Router#show ip interface serial0/0/0
...
Outgoing access list is not set
Inbound access list is not set
```

**HQ Router:**
```
Router#show ip interface fastEthernet0/0
...
Outgoing access list is not set
Inbound access list is HQ_TO_BRANCH
```
```
Router#show ip interface serial0/0/0
...
Outgoing access list is not set
Inbound access list is not set
```

✅ Confirmed on both routers: each ACL is bound inbound on `FastEthernet0/0` only. `Serial0/0/0` carries no ACL on either side, as intended.

---

## 2. OSPF stability

Since neither ACL is applied on `Serial0/0/0`, OSPF adjacency should be unaffected by either ACL entirely.

**Branch Router:**
```
Router#show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
4.4.4.4         1     FULL/DR         00:00:36    10.1.0.2        FastEthernet0/0
1.1.1.1         0     FULL/ -         00:00:36    10.255.255.1    Serial0/0/0
```

**HQ Router:**
```
Router#show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
3.3.3.3         1     FULL/DR         00:00:38    10.0.0.2        FastEthernet0/0
2.2.2.2         0     FULL/ -         00:00:30    10.255.255.2    Serial0/0/0
```

✅ Both routers show `FULL` adjacency to their L3 switch and to each other. The `permit ospf any any` line in both ACLs is retained as a safeguard (see `acl-design.md`), but is not currently load-bearing given this placement.

---

## 3. Ping test matrix

| From | To | Expected | Result |
|------|-----|----------|--------|
| Branch IT (10.1.10.x) | HQ IT (10.0.10.x) | Success | ✅ Success |
| Branch IT (10.1.10.x) | HQ HR (10.0.20.x) | Success | ✅ Success |
| Branch IT (10.1.10.x) | HQ Sales (10.0.30.x) | Success | ✅ Success |
| Branch HR (10.1.20.x) | HQ HR (10.0.20.x) | Success | ✅ Success |
| Branch HR (10.1.20.x) | HQ IT (10.0.10.x) | Fail | ✅ Failed (as expected) |
| Branch HR (10.1.20.x) | HQ Sales (10.0.30.x) | Fail | ✅ Failed (as expected) |
| Branch Sales (10.1.30.x) | HQ IT / HR / Sales | Fail (all) | ✅ Failed (as expected) |
| HQ IT (10.0.10.x) | Branch IT (10.1.10.x) | Success | ✅ Success |
| HQ IT (10.0.10.x) | Branch HR (10.1.20.x) | Success | ✅ Success |
| HQ IT (10.0.10.x) | Branch Sales (10.1.30.x) | Success | ⚠️ Failed — see Finding 2 below |
| HQ HR (10.0.20.x) | Branch HR (10.1.20.x) | Success | ✅ Success |
| HQ HR (10.0.20.x) | Branch IT (10.1.10.x) | Fail | ⚠️ Initially succeeded — see Finding 1 below (fixed, now correctly fails) |
| HQ Sales (10.0.30.x) | Branch IT / HR / Sales | Fail (all) | ✅ Failed (as expected) |
| Any data VLAN host | Router WAN IP (Telnet, port 23) | Refused | ✅ Refused on all tested VLANs |

---

## Finding 1 — ICMP permit rules were too broad (bug, fixed)

**Symptom:** HQ HR → Branch IT ping *succeeded*, even though policy restricts HR to reaching only the remote HR subnet.

**Cause:** The original ICMP permit rules were unscoped:
```
permit icmp 10.0.20.0 0.0.0.255 any
```
This matched ICMP from HQ HR to **any** destination, including Branch IT — and since it appeared before the correctly-scoped `permit ip 10.0.20.0 ... 10.1.20.0 ...` rule, it matched first and let the ping through regardless of destination. The same issue existed symmetrically in `BRANCH_TO_HQ`.

**Fix:** Replaced the broad ICMP rule with one scoped to the correct destination, on both ACLs:
```
! HQ_TO_BRANCH
no permit icmp 10.0.20.0 0.0.0.255 any
permit icmp 10.0.20.0 0.0.0.255 10.1.20.0 0.0.0.255

! BRANCH_TO_HQ
no permit icmp 10.1.20.0 0.0.0.255 any
permit icmp 10.1.20.0 0.0.0.255 10.0.20.0 0.0.0.255
```

**Confirmed fixed:** Re-testing after the fix showed HQ HR → Branch HR still succeeds, and HQ HR → Branch IT now correctly fails (see match counters below — line 70 on `HQ_TO_BRANCH` now shows 4 matches from the successful HR→HR test, and line 100's catch-all deny shows 4 matches corresponding to the now-blocked HR→IT attempt).

---

## Finding 2 — HQ IT → Branch Sales fails (expected limitation, not a bug)

**Symptom:** HQ IT is permitted to reach anything at Branch, but a ping to a Branch Sales host fails.

**Cause:** Standard ACLs are stateless — they don't recognize a reply as belonging to a request they already permitted outbound. Tracing the ping:
- The **request**, HQ IT → Branch Sales, is permitted fine by `HQ_TO_BRANCH` (IT is allowed anywhere).
- The **reply**, sourced from the Branch Sales host back toward HQ IT, arrives at the Branch router's `FastEthernet0/0` inbound and is evaluated against `BRANCH_TO_HQ`, where it matches:
  ```
  deny ip 10.1.30.0 0.0.0.255 any
  ```
  This rule blocks all traffic sourced from Branch Sales — including legitimate replies to a session HQ IT was allowed to start — because the ACL has no concept of "this is a reply, not a new request."

**Resolution:** Not fixed — documented as a known limitation of stateless ACLs. This is an inherent conflict between two policy requirements ("IT can reach anything" vs. "Sales can reach nothing") that collide specifically at the Sales destination. A proper fix would require **reflexive ACLs**, which track outbound sessions and dynamically permit their return traffic. This is considered out of scope for the current project but is a natural next step if revisited.

---

## 4. Final match counters (proof of enforcement)

Captured after the full ping test matrix above, following the Finding 1 fix:

```
Extended IP access list HQ_TO_BRANCH
 10 permit ospf any any (338 match(es))
 20 deny tcp 10.0.10.0 0.0.0.255 any eq telnet (24 match(es))
 30 deny tcp 10.0.20.0 0.0.0.255 any eq telnet
 40 deny tcp 10.0.30.0 0.0.0.255 any eq telnet
 50 deny ip 10.0.30.0 0.0.0.255 any (21 match(es))
 60 permit icmp 10.0.10.0 0.0.0.255 any (34 match(es))
 70 permit icmp 10.0.20.0 0.0.0.255 10.1.20.0 0.0.0.255 (4 match(es))
 80 permit ip 10.0.10.0 0.0.0.255 10.1.0.0 0.0.255.255
 90 permit ip 10.0.20.0 0.0.0.255 10.1.20.0 0.0.0.255
 100 deny ip any any (4 match(es))

Extended IP access list BRANCH_TO_HQ
 10 permit ospf any any (369 match(es))
 20 deny tcp 10.1.10.0 0.0.0.255 any eq telnet
 30 deny tcp 10.1.20.0 0.0.0.255 any eq telnet
 40 deny tcp 10.1.30.0 0.0.0.255 any eq telnet
 50 deny ip 10.1.30.0 0.0.0.255 any (32 match(es))
 60 permit icmp 10.1.10.0 0.0.0.255 any (43 match(es))
 70 permit icmp 10.1.20.0 0.0.0.255 10.0.20.0 0.0.0.255 (4 match(es))
 80 permit ip 10.1.10.0 0.0.0.255 10.0.0.0 0.0.255.255
 90 permit ip 10.1.20.0 0.0.0.255 10.0.20.0 0.0.0.255
 100 deny ip any any
```

**Reading the counters:**
- Line 10 (OSPF) on both ACLs shows high match counts — expected, since OSPF hellos are frequent, though as noted this line no longer affects live OSPF traffic given the `FastEthernet0/0` placement (it's a retained safeguard).
- `HQ_TO_BRANCH` line 20 (24 matches) confirms Telnet attempts from HQ IT toward router WAN IPs were correctly denied.
- `HQ_TO_BRANCH` line 50 (21 matches) confirms HQ Sales traffic toward Branch was denied outright.
- `HQ_TO_BRANCH` line 60 (34 matches) and `BRANCH_TO_HQ` line 60 (43 matches) confirm IT's unrestricted ICMP reachability was exercised and permitted, as intended.
- Line 70 on both ACLs (4 matches each) confirms the Finding 1 fix is active and being hit by legitimate HR↔HR ping traffic.
- `HQ_TO_BRANCH` line 100 (4 matches) confirms the now-blocked HQ HR → Branch IT attempt was correctly caught by the final deny after the fix.

---

## Conclusion

Both ACLs are correctly placed, do not interfere with OSPF, and enforce the access policy as designed — with one bug found and fixed during testing (ICMP over-permission for HR), and one known, documented limitation inherent to stateless ACLs (HQ IT → Branch Sales reply traffic). Match counters confirm real traffic is being evaluated against the expected rules rather than passing through unfiltered.