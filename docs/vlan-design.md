# VLAN Design

## Why These VLANs Exist

The segmentation here wasn't arbitrary. Each VLAN maps to a real organizational boundary with a specific security motivation.

---

### VLAN 10 – HR (192.168.10.0/24)

**Devices:** Computers, printers in reception area; the one-story building.

**Why separated:** HR handles PII — employee records, compensation data, performance reviews. This is the kind of data that, if it leaks to Finance or IT, creates compliance problems even if the person accessing it wasn't malicious. Keeping HR in its own VLAN means a Finance analyst can't accidentally (or intentionally) browse to an HR file share.

**Traffic allowed:**
- Internal: Access to DHCP, DNS (via server VLAN), and internet via NAT
- Blocked: Direct access to Finance VLAN, IT Admin VLAN
- Print: Local printers only (same VLAN, no cross-VLAN printing by default)

**Notes:** The conference room is physically in the same building. Right now conference room devices fall under VLAN 10. That's a gap I flagged — ideally conference/guest devices should be on a separate VLAN (call it VLAN 50 – Guest) with internet-only access and no reachability to any internal VLAN.

---

### VLAN 20 – Finance & Marketing (192.168.20.0/24)

**Devices:** Computers, printers, IP phones — floors 1–5 in Buildings B and C assigned to Finance and Marketing departments.

**Why combined:** Finance and Marketing need some level of collaboration (budget reviews, campaign spend approvals). Splitting them further would add ACL complexity with limited security gain at this org size. If this were a larger org with stricter financial controls (SOX compliance for example), I'd split Finance into its own VLAN.

**Why separated from everything else:** Finance data is obvious — transaction records, client billing, budget figures. Marketing strategies are arguably sensitive too (upcoming campaigns, pricing structures). The real concern is cross-contamination with IT Admin — you don't want a Finance workstation to reach network management interfaces.

**Traffic allowed:**
- Internet access via NAT
- Access to shared services (DNS, DHCP, potentially a file server in VLAN 40)
- IP phone traffic needs to reach voice infrastructure — in a real deployment VLAN 20 would need a separate voice VLAN (VLAN 21 or similar) for QoS tagging

**Traffic blocked:**
- No access to VLAN 30 (IT Admin)
- No access to VLAN 99 (Management)
- No direct access to VLAN 10 (HR)

---

### VLAN 30 – IT & Admin (192.168.30.0/24)

**Devices:** IT team workstations, admin computers, network management systems.

**Why separated:** This is your highest-trust user VLAN. IT admins need access to management interfaces, server infrastructure, potentially all other VLANs for troubleshooting. The segmentation here is about *limiting blast radius* — if an IT workstation gets compromised, you want the ACLs to contain what an attacker can reach from it. Doesn't fully solve the problem since IT has wide access by design, but it does mean a compromised Finance or HR workstation can't reach the management VLAN.

**Traffic allowed:**
- Access to VLAN 99 (Management) — IT only
- Access to VLAN 40 (Servers) for administration
- Internet access
- Can reach other VLANs for troubleshooting (this is the contentious ACL decision — see `acl-strategy.md`)

---

### VLAN 40 – Servers (192.168.40.0/24)

**Devices:** DHCP server, DNS (if local), file servers, application servers.

**Why a dedicated server VLAN:** Servers should not be on the same VLAN as any user group. A user VLAN getting flooded (broadcast storm, misconfigured device, malware) shouldn't affect server availability. More importantly, access to servers should be explicitly controlled — only specific VLANs should be able to reach specific servers.

**The DHCP server placement:** The DHCP server lives here but serves all VLANs. This works via `ip helper-address` on each distribution switch SVI — DHCP requests from clients are relayed to the centralized server. It means one server, one config to maintain, but the ACLs have to explicitly allow UDP 67/68 (DHCP) traffic from each user VLAN to reach VLAN 40.

---

### VLAN 99 – Management (192.168.99.0/24)

**Devices:** Switch management interfaces (SVIs), router management interface.

**Why separated:** Network device management should never share a VLAN with end users. If VLAN 99 is reachable from VLAN 20, a Finance user (or anything running on a Finance workstation) can attempt SSH or Telnet to switch management interfaces. Native VLAN best practice would also say don't use VLAN 1 for anything.

**Traffic allowed:** SSH from VLAN 30 (IT Admin) only. Nothing else.

---

## VLAN Assignment by Location

| Location | Floor | Department | VLAN |
|----------|-------|------------|------|
| Building A | 1 | HR / Reception | 10 |
| Building A | 1 | Conference (guest) | 10 (gap — should be 50) |
| Building B | 1–3 | Finance | 20 |
| Building B | 4–5 | Marketing | 20 |
| Building C | 1–3 | Finance | 20 |
| Building C | 4–5 | IT / Admin | 30 |
| Server Room | – | Infrastructure | 40 |
| All switches | – | Management | 99 |

---

## Trunk Configuration

All uplinks between access → distribution → core → router are configured as 802.1Q trunks allowing specific VLANs.

**Allowed VLANs on trunk links:**

| Link | Allowed VLANs |
|------|--------------|
| Access switch → Distribution | VLAN specific to that floor + 99 |
| Distribution → Core Switch | 10, 20, 30, 40, 99 |
| Core Switch → Core Router | 10, 20, 30, 40, 99 |

The access switch trunks only carry the VLANs they need. A floor that has only Finance workstations doesn't need VLAN 10 or 30 on its uplink trunk. This is the principle of least VLAN — don't allow VLANs across a trunk unless there's a device on the other end that needs them. Limits VLAN hopping surface area slightly.

---

## What This Design Doesn't Solve

- **Intra-VLAN traffic** is not controlled by these VLANs. Two Finance workstations on the same VLAN can still reach each other unrestricted. Micro-segmentation (host-based firewall, 802.1X, private VLANs) would be needed to address that.
- **VLAN hopping** via double-tagging attacks — mitigated somewhat by trunk pruning and setting native VLAN to 99 (not 1), but not fully eliminated without dedicated VLAN hopping prevention config.
- **Rogue DHCP** — a device brought onto any VLAN could theoretically run a DHCP server and start handing out addresses. DHCP snooping on access switches would fix this.
