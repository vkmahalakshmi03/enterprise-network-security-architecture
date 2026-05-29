# Enterprise Network Security Architecture

Built a multi-zone enterprise network from scratch — VLANs, ACLs, DHCP, NAT, the whole stack — with internal segmentation as the primary design constraint, not an afterthought. Simulated in Cisco Packet Tracer across a three-building campus topology.

The question driving the design: if one endpoint gets compromised, how far can an attacker actually move? Most of the time the answer in enterprise environments is "pretty far" — flat networks, permissive ACLs, or VLANs that exist on paper but aren't enforced. This was built to address that.

---

## The Problem I Was Solving

Perimeter security stops things at the edge. It does not stop lateral movement once someone is inside. An HR workstation, a Finance file server, and a network management interface sitting on the same routable network is a segmentation failure — not a firewall gap.

The design here treats every department boundary as a security boundary. Default-deny between VLANs, explicit permits only where there is a real business reason for the traffic to flow.

---

## Objectives

- Isolate departments at the network layer so a compromised endpoint in one VLAN has no path to another
- Apply ACL enforcement at the distribution layer, inbound on SVIs, before traffic enters the routing process
- Centralize DHCP with per-VLAN relay rather than spinning up a server per VLAN
- NAT at the perimeter — internal addressing stays private, all outbound traffic maps to a single WAN IP
- Validate every security boundary with actual traffic tests, not just config review

---

## How I Approached It

Started with the org structure and worked backwards to the network design. Three departments with meaningfully different data sensitivity levels — HR handles PII, Finance handles transaction data, IT needs privileged access to infrastructure. Those differences should be reflected in the network, not ignored.

**Physical topology:** Hybrid star-bus. Star within each floor (every device to an access switch), bus backbone between floors within a building (access switches up to a distribution switch). Three distribution switches, one per building, all converging at a core L3 switch. Keeps cabling manageable and isolates floor-level failures.

**Logical topology:** Three-tier hierarchy — core, distribution, access. Inter-VLAN routing lives at the distribution layer, not the core. ACLs applied inbound on SVIs at the distribution layer. The core just moves traffic between buildings fast — policy enforcement happens closer to where traffic originates.

**VLAN design:** One VLAN per organizational boundary. Each VLAN got its own /24 subnet, its own DHCP pool, and its own ACL. VLAN 99 as a dedicated management VLAN for switch SVIs — completely separate from user traffic, SSH from IT only.

**ACL strategy:** Named extended ACLs. Default-deny with explicit permits. Every permit has a justification. Specific service permits (DHCP, DNS, SMB to specific servers) come before broad denies, denies between internal VLANs come before the internet permit. That ordering matters more than it sounds — more on that below.

---

## Architecture

**Campus:** 3 buildings — 1 reception (1 floor) + 2 office buildings (5 floors each)

```
                    [ ISP ]
                       |
                  [ Firewall ]
                       |
               [ Core Router ]
               NAT/PAT · Edge
                       |
              [ Core Switch L3 ]
           Inter-VLAN Routing · ACL
          /            |            \
    [ Dist-A ]    [ Dist-B ]    [ Dist-C ]
    Building A    Building B    Building C
         |        |  |  |  |    |  |  |  |  |
        F1        F1 F2 F3 F4 F5  F1 F2 F3 F4 F5
      VLAN 10        VLAN 20          VLAN 20/30
```

**Device count:** 1 core router · 1 L3 switch · 3 distribution switches · 8 access switches · 3 servers

| Layer | Device | Role |
|-------|--------|------|
| Edge | Firewall + Core Router 2911 | Perimeter, NAT/PAT |
| Core | L3 Switch Cisco 3560 | Inter-building routing |
| Distribution | Cisco 2960 × 3 | VLAN aggregation, ACL enforcement |
| Access | Cisco 2960 × 8 | End device connectivity, port security |
| Services | DHCP + DNS + File Server | Centralized in VLAN 40 |

---

## Segmentation Design

| VLAN | Subnet | Department | Why Separated |
|------|--------|------------|---------------|
| 10 | 192.168.10.0/24 | HR | PII — no path to Finance or IT |
| 20 | 192.168.20.0/24 | Finance & Marketing | Financial data — isolated from HR and IT |
| 30 | 192.168.30.0/24 | IT & Admin | Privileged access — widest reach by design, still isolated from user VLANs |
| 40 | 192.168.40.0/24 | Servers | Services only — DHCP, DNS, File. No unsolicited outbound to users |
| 99 | 192.168.99.0/24 | Management | Switch SVIs — SSH from IT only, nothing else has a path here |

Trunk pruning applied on all uplinks — each trunk only carries the VLANs that have devices on the other end. Building B distribution switch never carries VLAN 30 because there are no IT devices in that building. Limits VLAN hopping surface slightly and keeps the trunk config honest.

---

## Access Control Matrix

| Source | Internet | VLAN 10 HR | VLAN 20 Finance | VLAN 30 IT | VLAN 40 Servers | VLAN 99 Mgmt |
|--------|----------|------------|-----------------|------------|-----------------|--------------|
| VLAN 10 HR | NAT | — | DENY | DENY | DHCP + DNS only | DENY |
| VLAN 20 Finance | NAT | DENY | — | DENY | DHCP + DNS + SMB + HTTP | DENY |
| VLAN 30 IT | NAT | troubleshooting only | troubleshooting only | — | full access | SSH only |
| VLAN 40 Servers | NAT | DHCP responses | DHCP + DNS + return traffic | DHCP responses | — | DENY |
| VLAN 99 Mgmt | NAT | DENY | DENY | full access | DENY | — |

---

## What's in This Repo

| Folder | Contents |
|--------|----------|
| [configs/](configs/) | Cisco IOS configuration — VLANs, ACLs, router, DHCP/NAT with inline comments |
| [docs/](docs/) | Architecture decisions, VLAN rationale, ACL logic, security analysis |
| [diagrams/](diagrams/) | Mermaid diagrams — topology, traffic flow, security zones |
| [validation/](validation/) | Test cases, expected results, and what actually happened |

Good places to start:
- [configs/acl-rules.txt](configs/acl-rules.txt) — full ACL ruleset, every decision has a comment
- [docs/acl-strategy.md](docs/acl-strategy.md) — reasoning behind each ACL including a mistake caught during testing
- [validation/findings.md](validation/findings.md) — what failed, why, and how it was fixed
- [diagrams/security-zones.md](diagrams/security-zones.md) — full threat model and zone boundary map

---

## What I Observed

**ACL ordering is a silent failure mode.**
The broad internet permit — `permit ip ... any` — was initially placed above the inter-VLAN deny rules. HR traffic to Finance was going through. The deny rule showed 0 hits. Nothing broke visibly because internet still worked. Only caught it by checking ACL hit counters after running test traffic.

This is exactly why this misconfiguration exists in production networks for months without anyone noticing. The fix was reordering — deny internal VLANs first, then permit internet. After the reorder, deny rule counters incremented correctly on cross-VLAN traffic. Full breakdown in [validation/findings.md](validation/findings.md).

**Return traffic needs its own permit.**
Extended ACLs without stateful inspection require explicit permits in both directions. Finance could send a TCP SYN to the file server but the SYN-ACK was dropped on the way back — no return permit on the server-side ACL. Added the `established` keyword to ACL-VLAN40-IN. Easy to miss when you're only thinking about the outbound direction.

**DHCP relay failure is invisible.**
Missing `ip helper-address` on the VLAN 20 SVI meant Finance devices silently fell back to APIPA (169.254.x.x). No error message, no alert — devices just didn't get addresses. Caught only during DHCP test cases, not during initial build.

**Trunk pruning is an actual control, not just cleanup.**
Building B distribution switch not carrying VLAN 30 means a device on Building B floors has no path to reach a VLAN 30 segment in Building C — that VLAN simply doesn't exist on the trunk between them. Worth verifying explicitly rather than assuming the pruning config is correct.

---

## Results

| Test | Result | Notes |
|------|--------|-------|
| Intra-VLAN connectivity | pass | Conference room port sitting on VLAN 1 default — fixed |
| Internet access via NAT | pass | NAT inside/outside were swapped on interfaces — corrected |
| DHCP assignment all VLANs | pass | VLAN 20 missing ip helper-address — added |
| HR → Finance blocked | pass | ACL reorder required — ordering bug, details in findings.md |
| Finance → IT blocked | pass | — |
| All VLANs → Mgmt blocked | pass | — |
| IT SSH to management switches | pass | — |
| Finance → File Server SMB | pass | Return traffic permit missing in server-side ACL — added |
| Trunk VLAN pruning | pass | Verified via PDU testing across buildings |

4 issues found and resolved during testing. None were caught by config review alone — all required generating actual traffic and checking hit counters or DHCP bindings.

---

## Known Gaps

Documenting these because they're real, not because the scope ran out:

- **No guest VLAN** — conference room visitor devices currently share VLAN 10 with HR. Should be a completely isolated VLAN with internet-only access and a hard deny to all internal VLANs.
- **No DHCP snooping** — a rogue DHCP server on any VLAN goes undetected. One config block on access switches fixes this.
- **No 802.1X** — any device plugged into an access port gets network access. Port security limits MAC count but doesn't authenticate the device.
- **No monitoring** — ACL deny logging is configured but there is no syslog destination. In production this feeds a SIEM. Without it, denied traffic is invisible.
- **Single core switch** — no redundancy at the core layer. HSRP or a dual-core design would address this.

---

## Conclusion

The segmentation holds up for what it was designed to solve. A compromised HR workstation has no routable path to Finance or IT. Finance can reach its file server and DNS but nothing else in the server VLAN. The management VLAN is locked to SSH from IT only.

The more useful takeaway was about validation methodology. Config review alone missed the ACL ordering bug, the missing DHCP relay, and the swapped NAT interfaces. Generating traffic and verifying hit counters caught all of them. In a production environment without that active validation, all three would have gone unnoticed — the network would have looked like it was working while the segmentation was silently broken.

The gaps listed above are the next layer. This design solves the segmentation problem. DHCP snooping, 802.1X, and monitoring solve the endpoint trust and visibility problems that segmentation alone cannot address.

---

## Stack

`Cisco IOS` `VLANs` `802.1Q Trunking` `Named Extended ACLs` `Inter-VLAN Routing (SVI)` `DHCP Relay` `NAT/PAT` `Port Security` `Cisco Packet Tracer 8.2` `Mermaid`
