# Test Findings & Observations

What actually happened when I ran the tests. Documenting failures and how they were diagnosed matters more than just listing passes.

---

## TC-01: Intra-VLAN Connectivity

**Result: PASS (after one fix)**

HR workstations and Finance workstations communicated within their VLANs as expected. IT Admin VLAN same.

**Issue found:** Conference room device in Building A was initially showing no connectivity. Traced to the access port being left in VLAN 1 (default) instead of VLAN 10. Fixed by:
```
interface FastEthernet0/24
 switchport access vlan 10
```
After fix: ping worked. This is a config discipline thing — when you add a new device, the port needs to be explicitly assigned. Default VLAN 1 should have no devices.

---

## TC-02: Internet Access

**Result: PASS (after NAT interface direction fix)**

Initial config had `ip nat inside` and `ip nat outside` applied to the wrong interfaces. Gi0/0 (WAN) had `ip nat inside` and the LAN link had `ip nat outside`. Result: no translations were created, all traffic was dropped.

Corrected:
```
interface GigabitEthernet0/0
 ip nat outside
interface GigabitEthernet0/1
 ip nat inside
```

After correction, `show ip nat translations` showed entries correctly. All VLANs successfully NAT'd.

**Observation:** In Packet Tracer the NAT debug output is limited. You can see translations in the table but can't see the step-by-step process like you can with `debug ip nat` on real hardware. This made diagnosing the inside/outside swap harder than it needed to be — had to review the config manually.

---

## TC-03: DHCP Assignment

**Result: PASS (with one VLAN failing initially)**

VLAN 10 (HR), VLAN 30 (IT), and VLAN 40 devices received addresses correctly. VLAN 20 (Finance) was failing — devices showed APIPA (169.254.x.x).

**Root cause:** Missing `ip helper-address` on VLAN 20 SVI. Added:
```
interface Vlan20
 ip helper-address 192.168.40.10
```

After fix: Finance devices received addresses from the 192.168.20.0/24 pool correctly.

**Side observation on DHCP snooping:** Didn't configure it. If I had, I would have needed to mark the uplink port as trusted before enabling snooping — otherwise the DHCP server itself would've been on an untrusted port and its offers would've been dropped. That's a common DHCP snooping misconfiguration.

---

## TC-04: Inter-VLAN Blocking

**Result: PASS (after ACL ordering correction)**

**This is the most important finding in the whole project.**

Initial result: HR could reach Finance. The deny rule was in the ACL but traffic was passing. Checked ACL hit counts:

```
show ip access-lists ACL-VLAN10-IN
Extended IP access list ACL-VLAN10-IN
    50 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255 (0 matches)
    90 permit ip 192.168.10.0 0.0.0.255 any (8 matches)
```

The permit rule had 8 matches. The deny rule had 0. Traffic was hitting the `permit ip ... any` rule first because it appeared above the deny rules in the ACL. The `permit ip ... any` rule matched everything destined for any IP, including 192.168.20.x, before the more-specific deny could fire.

**Fix:** Reordered the ACL. Deny rules for internal VLANs go before the internet permit. Correct order:

```
50 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
60 deny ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255
70 deny ip 192.168.10.0 0.0.0.255 192.168.40.0 0.0.0.255
80 deny ip 192.168.10.0 0.0.0.255 192.168.99.0 0.0.0.255
90 permit ip 192.168.10.0 0.0.0.255 any       ← internet now
```

After reorder, HR → Finance ICMP was denied. Hit counter on rule 50 incremented.

**Why this matters:** This exact issue comes up in real environments. Someone adds a broad permit for internet access and it silently overrides the granular denies that came after it. Nobody notices because the internet works. The cross-VLAN access keeps working too, and nobody checks because nothing obviously broke. ACLs need to be reviewed as a complete ordered list, not rule-by-rule.

---

## TC-05: Permitted Services

**Result: PASS**

Finance → File Server (TCP 445) worked correctly after ACL-VLAN40-IN was configured with the return traffic permit (`established` keyword on rule 60). Without the return permit, the SYN got through but the SYN-ACK was dropped on the way back — connection never established.

This is a common mistake: writing ACLs for outbound direction only and forgetting that stateful inspection isn't happening here. Named extended ACLs don't have stateful inspection — you need to explicitly permit the return direction unless you're using something like CBAC or Zone-Based Firewall.

---

## TC-06: Management VLAN

**Result: PASS**

IT Admin SSH to switch SVIs worked. All other VLANs blocked at both the ACL-VLAN99-IN rule and the VTY access-class. Tested from HR, Finance, and Server VLAN — all failed to connect as expected.

---

## TC-07: Server VLAN Access

**Result: PASS**

HR was blocked from file server (SMB). Servers couldn't reach management VLAN. IT could admin servers.

One unexpected thing: ICMP from the server VLAN toward user VLANs was being permitted by the `permit ip ... any` rule in ACL-VLAN40-IN, which allowed ping from the server to Finance/HR. This isn't a critical issue since it's server-initiated, but I should have added explicit ICMP restricts for server-to-user directions. Added to the gap list.

---

## TC-09: Trunk Verification

**Result: PASS**

VLAN pruning working correctly. Building B distribution switch trunk to core shows only VLANs 20 and 99. Building C distribution switch shows 20, 30, 40, 99.

**Note:** Packet Tracer's `show interfaces trunk` doesn't always show the pruned VLAN list the same way real IOS does. Validated via PDU testing — confirmed that a packet tagged VLAN 30 originating from Building B couldn't reach a VLAN 30 device in Building C (because the Building B distribution switch doesn't carry VLAN 30 up to the core).

---

## Summary of Issues Found

| Issue | Severity | Root Cause | Resolution |
|-------|----------|------------|------------|
| VLAN 20 DHCP failure | High | Missing ip helper-address on Vlan20 SVI | Added helper-address |
| HR → Finance not blocked | **Critical** | ACL rule ordering: permit all before deny | Reordered ACL |
| NAT not working | High | ip nat inside/outside on wrong interfaces | Swapped interface NAT direction |
| Conference room on VLAN 1 | Medium | Default port VLAN not changed | Set access vlan 10 |
| Server VLAN ICMP to users | Low | Overly broad permit in VLAN 40 ACL | Documented as gap, not fixed in initial pass |

The ACL ordering issue was the most significant find. Everything else was configuration mistakes. The ACL ordering bug is the kind of thing that can exist in a production network for months or years without anyone noticing because nothing obviously stops working.
