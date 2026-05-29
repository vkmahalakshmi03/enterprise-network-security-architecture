# Test Cases

## Test Environment

- Platform: Cisco Packet Tracer 8.2
- Testing method: Ping, Traceroute, PDU (Packet Tracer's visual packet tool)
- ACL verification: `show ip access-lists` hit counters before and after each test
- Procedure: clear counters before each test batch → generate traffic → check hit counts

---

## TC-01: Basic Connectivity (Baseline)

**What it validates:** Devices in the same VLAN can communicate with each other before inter-VLAN controls are tested.

| Test ID | Source | Destination | Protocol | Expected |
|---------|--------|-------------|----------|----------|
| TC-01a | HR PC 192.168.10.50 | HR Printer 192.168.10.100 | ICMP | Pass |
| TC-01b | Finance PC 192.168.20.50 | Finance PC 192.168.20.60 | ICMP | Pass |
| TC-01c | IT PC 192.168.30.10 | IT PC 192.168.30.11 | ICMP | Pass |

**Why run this first:** If intra-VLAN traffic fails, the issue is at the access layer (wrong VLAN assignment, trunk config, etc.). Getting this working first isolates inter-VLAN ACL testing from basic connectivity issues.

---

## TC-02: Internet Access (NAT/PAT)

**What it validates:** All user VLANs can reach internet destinations via NAT overload.

| Test ID | Source | Destination | Protocol | Expected |
|---------|--------|-------------|----------|----------|
| TC-02a | HR PC 192.168.10.50 | 8.8.8.8 (simulated) | ICMP | Pass |
| TC-02b | Finance PC 192.168.20.50 | 8.8.8.8 | ICMP | Pass |
| TC-02c | IT PC 192.168.30.10 | 8.8.8.8 | ICMP | Pass |

**Verification:** `show ip nat translations` on RT-CORE should show entries mapping internal IPs to the WAN address.

---

## TC-03: DHCP Assignment

**What it validates:** Devices in each VLAN receive an IP address from the correct pool.

| Test ID | VLAN | Expected IP Range | Expected Gateway | Expected DNS |
|---------|------|-------------------|-----------------|-------------|
| TC-03a | VLAN 10 | 192.168.10.21 – .254 | 192.168.10.1 | 192.168.40.20 |
| TC-03b | VLAN 20 | 192.168.20.21 – .254 | 192.168.20.1 | 192.168.40.20 |
| TC-03c | VLAN 30 | 192.168.30.21 – .254 | 192.168.30.1 | 192.168.40.20 |

**Verification:** `show ip dhcp binding` on the DHCP server. Also `ipconfig` on client (Packet Tracer command prompt).

---

## TC-04: Inter-VLAN Blocking (Critical Tests)

**What it validates:** ACLs actually enforce the segmentation. These are the most important tests.

| Test ID | Source | Destination | Protocol | Expected | ACL Rule |
|---------|--------|-------------|----------|----------|----------|
| TC-04a | HR 10.50 | Finance 20.50 | ICMP | **DENY** | ACL-VLAN10-IN rule 50 |
| TC-04b | HR 10.50 | IT 30.10 | ICMP | **DENY** | ACL-VLAN10-IN rule 60 |
| TC-04c | Finance 20.50 | HR 10.50 | ICMP | **DENY** | ACL-VLAN20-IN rule 90 |
| TC-04d | Finance 20.50 | IT 30.10 | ICMP | **DENY** | ACL-VLAN20-IN rule 100 |
| TC-04e | HR 10.50 | Mgmt 99.21 | TCP/22 | **DENY** | ACL-VLAN10-IN rule 80 |
| TC-04f | Finance 20.50 | Mgmt 99.21 | TCP/22 | **DENY** | ACL-VLAN20-IN rule 120 |

**Verification method:** After each test, run `show ip access-lists ACL-VLAN10-IN` and confirm the hit counter on the expected deny rule incremented. If the permit counter increments instead, the ACL has an ordering issue.

---

## TC-05: Permitted Inter-VLAN Services

**What it validates:** Permitted services (DHCP, DNS, file access) work correctly across VLANs.

| Test ID | Source | Destination | Protocol/Port | Expected | ACL Rule |
|---------|--------|-------------|---------------|----------|----------|
| TC-05a | Finance 20.50 | File Server 40.30 | TCP/445 | **PERMIT** | ACL-VLAN20-IN rule 50 |
| TC-05b | Finance 20.50 | File Server 40.30 | TCP/80 | **PERMIT** | ACL-VLAN20-IN rule 70 |
| TC-05c | IT 30.10 | Server VLAN 40.x | TCP/22 | **PERMIT** | ACL-VLAN30-IN rule 20 |
| TC-05d | IT 30.10 | Mgmt 99.21 | TCP/22 | **PERMIT** | ACL-VLAN30-IN rule 10 |

---

## TC-06: Management VLAN Isolation

**What it validates:** VLAN 99 is only reachable via SSH from IT Admin VLAN.

| Test ID | Source | Destination | Protocol | Expected |
|---------|--------|-------------|----------|----------|
| TC-06a | IT 30.10 | Switch SVI 99.21 | TCP/22 | **PERMIT** |
| TC-06b | HR 10.50 | Switch SVI 99.21 | TCP/22 | **DENY** |
| TC-06c | Finance 20.50 | Switch SVI 99.21 | TCP/22 | **DENY** |
| TC-06d | Server 40.10 | Switch SVI 99.21 | TCP/22 | **DENY** |
| TC-06e | IT 30.10 | Switch SVI 99.21 | ICMP | **PERMIT** |
| TC-06f | Finance 20.50 | Switch SVI 99.21 | ICMP | **DENY** |

---

## TC-07: Server VLAN Access Controls

**What it validates:** Server VLAN allows service traffic but not unauthorized access.

| Test ID | Source | Destination | Protocol | Expected |
|---------|--------|-------------|----------|----------|
| TC-07a | HR 10.50 | File Server 40.30 | TCP/445 | **DENY** | HR has no file server permit |
| TC-07b | HR 10.50 | File Server 40.30 | TCP/80 | **DENY** |
| TC-07c | Server 40.10 | Switch SVI 99.21 | TCP/22 | **DENY** | Servers can't reach Mgmt |
| TC-07d | IT 30.10 | DHCP Server 40.10 | TCP/22 | **PERMIT** | IT admin of servers |

---

## TC-08: ACL Ordering Regression

**What it validates:** The ACL ordering bug found during initial testing (permit all before deny rules) doesn't reoccur after fix.

| Test ID | Description | Expected |
|---------|-------------|----------|
| TC-08a | HR → Finance after ACL correction | DENY (previously was passing) |
| TC-08b | Check hit counters — deny rule should show > 0 hits | Deny rule counter > 0 |
| TC-08c | Permit rule for internet should NOT show hits for internal traffic | Internet permit rule hits = 0 for internal dest |

---

## TC-09: Trunk and VLAN Propagation

**What it validates:** VLANs are propagated correctly on trunk links and pruned where not needed.

| Test ID | Check |
|---------|-------|
| TC-09a | `show interfaces trunk` on DIST-B — VLAN 30 should NOT be in allowed list |
| TC-09b | `show vlan brief` on ACC-B-F1 — should show VLAN 20 and 99 only |
| TC-09c | `show interfaces trunk` on DIST-C — VLAN 30 should be present (Building C has IT floors) |
