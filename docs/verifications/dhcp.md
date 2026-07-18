# DHCP Verification

Proof that centralized DHCP with relay (see [`doc/dhcp-design.md`](../dhcp-design.md)) correctly assigns addresses to all six VLANs across both sites, including two issues found and fixed during testing.

---

## 1. Final DHCP bindings (HQ Router)

```
Router#show ip dhcp binding
IP address       Client-ID/          Lease expiration      Type
                  Hardware address
10.0.10.5         000C.CFD2.6A96      --                    Automatic
10.0.10.6         0001.638D.3A2D      --                    Automatic
10.0.10.7         0000.0C1B.A224      --                    Automatic
10.0.20.2         0002.1640.7552      --                    Automatic
10.0.20.3         000D.BD23.60D0      --                    Automatic
10.0.20.4         00D0.D30A.43D0      --                    Automatic
10.0.30.2         0060.70E0.5673      --                    Automatic
10.0.30.3         0001.9652.2989      --                    Automatic
10.0.30.4         0090.2BA6.126E      --                    Automatic
10.1.10.2         0090.0CCC.ED28      --                    Automatic
10.1.10.3         0000.0C25.D5D0      --                    Automatic
10.1.10.4         0050.0F41.9E3A      --                    Automatic
10.1.20.2         00D0.9788.638C      --                    Automatic
10.1.20.3         0001.96C6.E664      --                    Automatic
10.1.20.4         0000.0CD4.1C02      --                    Automatic
10.1.30.2         0060.3E4E.6BBE      --                    Automatic
10.1.30.3         0090.21CB.48A0      --                    Automatic
10.1.30.4         0060.3E85.0DE3      --                    Automatic
```

✅ All six VLANs across both sites — HQ IT/HR/Sales and Branch IT/HR/Sales — show correctly-scoped leases, confirming DHCP relay and the ACL fix (Finding 2) are both working end to end.

---

## 2. Test matrix

| VLAN | Department | Expected subnet | Result |
|------|------------|------------------|--------|
| 10   | HQ IT      | 10.0.10.0/24     | ✅ Success |
| 20   | HQ HR      | 10.0.20.0/24     | ✅ Success |
| 30   | HQ Sales   | 10.0.30.0/24     | ✅ Success (see Finding 2 — required an ACL fix) |
| 110  | Branch IT  | 10.1.10.0/24     | ✅ Success (see Finding 1 — required a relay-target fix) |
| 120  | Branch HR  | 10.1.20.0/24     | ✅ Success (see Finding 1 — required a relay-target fix) |
| 130  | Branch Sales | 10.1.30.0/24   | ✅ Success (see Findings 1 and 2 — required both fixes) |

---

## Finding 1 — Branch helper-address pointed at the wrong device (bug, fixed)

**Symptom:** no Branch VLAN (110/120/130) could obtain an address, even though the HQ VLANs worked correctly and the relevant DHCP pools existed on the HQ router.

**Cause:** the Branch L3 switch's `ip helper-address` was initially set to the Branch router's own LAN-facing interface (`10.1.0.1`), on the assumption that the Branch router would forward the request onward toward HQ. It does not — the Branch router has no DHCP pools configured (all pools live on the HQ router) and `ip helper-address` does not chain across multiple router hops automatically; each hop needs its own explicit relay configuration. The relayed packet was arriving at the Branch router and going nowhere.

Confirmed via:
```
Router#show ip dhcp pool BRANCH-IT
Pool BRANCH-IT :
 Leased addresses       : 0
 1 subnet is currently in the pool :
 Current index        IP address range                    Leased/Excluded/Total
 10.1.10.1             10.1.10.1     - 10.1.10.254          0    / 6        / 254
```
Zero leases despite a correctly-defined pool and a DHCP request already having been attempted — consistent with the request never reaching this pool at all.

**Fix:** changed the Branch L3 switch's helper-address on all three Branch SVIs to point directly at the HQ router's real interface (`10.0.0.1`), the same target used at HQ:
```
interface vlan 110
 no ip helper-address 10.1.0.1
 ip helper-address 10.0.0.1
interface vlan 120
 no ip helper-address 10.1.0.1
 ip helper-address 10.0.0.1
interface vlan 130
 no ip helper-address 10.1.0.1
 ip helper-address 10.0.0.1
```

**Confirmed fixed:** VLAN 110 and 120 test PCs obtained correctly-scoped addresses immediately after this change and a DHCP renew (Static → DHCP toggle).

---

## Finding 2 — Sales VLANs (both sites) still failed after Finding 1 fix (bug, fixed — ACL ordering)

**Symptom:** after fixing the helper-address target, IT and HR VLANs at both sites worked, but Sales VLANs (VLAN 30 at HQ, VLAN 130 at Branch) still could not obtain an address.

**Cause:** this was an ACL issue, not a DHCP or relay issue (see [`doc/acl-design.md`](../acl-design.md) and [`doc/verifications/acl.md`](acl.md) Finding 3 for the full ACL-side account). In short: the DHCP permit rule on both `HQ_TO_BRANCH` and `BRANCH_TO_HQ` was initially placed after the Sales deny-all rule, so Sales' relayed DHCP request — sourced from the Sales subnet — matched the deny-all first and never reached the DHCP permit.

**Fix:** reordered the DHCP permit rule to sit near the top of both ACLs, ahead of the Sales deny-all:
```
! On HQ Router
ip access-list extended HQ_TO_BRANCH
 no permit udp any eq 68 any eq 67
 15 permit udp any eq 68 any eq 67

! On Branch Router
ip access-list extended BRANCH_TO_HQ
 no permit udp any eq 68 any eq 67
 15 permit udp any eq 68 any eq 67
```

**Confirmed fixed:**
```
Router#show ip access-lists BRANCH_TO_HQ
Extended IP access list BRANCH_TO_HQ
 permit ospf any any (532 match(es))
 permit udp any eq bootpc any eq bootps (32 match(es))
 ...
 deny ip 10.1.30.0 0.0.0.255 any (10 match(es))
 ...
 deny ip any any (10 match(es))
```
VLAN 130's DHCP request now matches the relocated permit line (32 matches, growing with each request) instead of the Sales deny-all — confirmed by testing the VLAN 130 PC afterward and receiving a correctly-scoped `10.1.30.x` address. The same fix and result applied symmetrically to VLAN 30 on `HQ_TO_BRANCH`.

---

## 3. ACL match counters confirming DHCP enforcement

See [`doc/verifications/acl.md`](acl.md), section 4, for the full match counter listing on both ACLs — line 15 (`permit udp any eq 68 any eq 67`) on each confirms DHCP relay traffic from every VLAN, including both Sales VLANs, is being correctly permitted ahead of any deny rule.

---

## Conclusion

DHCP relay is functioning correctly across all six VLANs at both sites, using a single centralized DHCP service on the HQ router. Two issues were found and resolved during testing: an incorrect relay target at Branch (fixed by pointing directly at the DHCP server's real interface rather than an intermediate router), and an ACL ordering conflict that blocked Sales VLANs specifically (fixed by reordering the DHCP permit rule ahead of the Sales deny-all on both ACLs). `show ip dhcp binding` and ACL match counters together confirm all six VLANs are obtaining correctly-scoped addresses as designed.