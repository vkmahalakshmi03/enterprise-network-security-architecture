# Enterprise Network Security Architecture

A multi-zone network security implementation covering VLAN segmentation, ACL-based traffic control, DHCP, and NAT across a simulated enterprise campus environment. Built and tested in Cisco Packet Tracer.

The focus was on internal segmentation — not just getting the network up, but making sure that a compromised endpoint in one department has no lateral path to another. Perimeter firewall alone does not cover that.

## Architecture Overview

Three-building campus. Two 5-floor office buildings, one reception/conference building. Hybrid physical topology (star per floor, bus backbone between floors) with a three-tier logical hierarchy — core, distribution, access.

Inter-VLAN routing happens at the distribution layer. ACLs are applied inbound on SVIs, so traffic is evaluated before it enters the routing process.

## Segmentation Design

| VLAN | Subnet | Purpose |
|------|--------|---------|
| 10 | 192.168.10.0/24 | HR |
| 20 | 192.168.20.0/24 | Finance and Marketing |
| 30 | 192.168.30.0/24 | IT and Admin |
| 40 | 192.168.40.0/24 | Servers (DHCP, DNS, File) |
| 99 | 192.168.99.0/24 | Management (switches) |

Each VLAN maps to a real organizational boundary. HR carries PII. Finance carries transaction data. IT needs wide access but should still be isolated from user VLANs at the routing layer. Servers are their own zone — only specific ports from specific VLANs reach specific servers.

Default-deny between VLANs. Explicit permits only where there is a legitimate business reason.

## What Is in This Repo

- configs/ — Cisco IOS configuration (VLANs, ACLs, router, DHCP/NAT)
- docs/ — Architecture decisions, VLAN rationale, ACL logic, security analysis
- diagrams/ — Mermaid diagrams for topology, traffic flow, security zones
- validation/ — Test cases, expected results, and actual findings

## Key Finding

The most significant issue found during testing: ACL rule ordering silently broke inter-VLAN blocking. The broad internet permit rule was placed above the inter-VLAN deny rules. HR traffic to Finance was passing — the deny rule showed 0 hits, the permit was matching everything first.

Fix was reordering: deny internal VLANs first, then permit internet. After correction, deny rule hit counters incremented correctly on cross-VLAN traffic.

This kind of misconfiguration exists in production networks longer than it should because nothing obviously breaks — internet still works, nobody checks whether the segmentation is actually enforced.

Full breakdown in validation/findings.md

## Known Gaps

- No guest VLAN — conference room devices share VLAN 10 with HR
- No DHCP snooping — rogue DHCP on any VLAN goes undetected
- No 802.1X — any device plugged in gets network access
- No monitoring — ACL deny logging configured but no syslog destination
- Single core switch — no redundancy at the core layer

## Stack

Cisco IOS · VLANs · 802.1Q Trunking · Extended ACLs · Inter-VLAN Routing (SVI) · DHCP · NAT/PAT · Cisco Packet Tracer 8.2# Enterprise Network Security Architecture

**Project Period:** Aug 2024 – Dec 2024 (George Mason University – CYSE 530)  
**Implementation Environment:** Cisco Packet Tracer 8.2  
**Last Updated:** May 2025  

> **Note on timeline:** The original network design memo was written in September 2024 as part of a university module assignment. The Packet Tracer implementation, ACL configuration, and validation testing documented here was completed between October–December 2024. Diagrams were updated and this repo was formalized in early 2025.

---

## What This Is

This repo documents a multi-zone enterprise network security architecture I designed and implemented for a simulated three-building campus. The goal wasn't just to get devices talking — it was to figure out where segmentation actually matters and what breaks when you get it wrong.

The scenario: a new campus with two 5-story office buildings and a 1-story reception/conference building. Around 8–12 network devices depending on how you count the topology (routers, L3 switches, access switches, firewall boundary, client segments). Four to six VLANs depending on how granular you go with the segmentation.

The bigger question I was working through: most enterprise breaches don't fail at the perimeter. They fail because once someone's inside, lateral movement is easy. This design tries to address that with deliberate VLAN segmentation and layered ACL control.

---

## Repository Layout

```
enterprise-network-security-architecture/
├── README.md
├── docs/
│   ├── architecture.md          # Physical + logical topology decisions
│   ├── vlan-design.md           # VLAN segmentation rationale
│   ├── acl-strategy.md          # ACL rules and the thinking behind them
│   └── security-analysis.md     # What I observed, what held up, what didn't
├── configs/
│   ├── vlan-config.txt          # VLAN configuration (Cisco IOS syntax)
│   ├── acl-rules.txt            # Full ACL ruleset
│   ├── router-config.txt        # Core router setup
│   └── dhcp-nat-config.txt      # DHCP pools and NAT configuration
├── diagrams/
│   ├── topology.md              # Mermaid: physical + logical network diagram
│   ├── traffic-flow.md          # Mermaid: inter-VLAN and inter-building flow
│   └── security-zones.md        # Mermaid: security boundary map
└── validation/
    ├── test-cases.md            # What I tested and why
    ├── expected-results.md      # What should happen
    └── findings.md              # What actually happened
```

---

## Quick Summary of the Architecture

| Layer | Device | Role |
|---|---|---|
| Internet/Edge | ISP → Firewall | Perimeter filtering |
| Core | Core Router + Core Switch | Inter-building routing |
| Distribution | Per-building distribution switch | VLAN aggregation + inter-VLAN routing |
| Access | Per-floor access switch | End device connectivity |
| Services | DHCP Server, NAT | IP management, address translation |

**VLANs:**
- VLAN 10 – HR (reception building)
- VLAN 20 – Finance & Marketing
- VLAN 30 – IT & Admin
- VLAN 40 – Server/Infrastructure
- VLAN 99 – Management (out-of-band)

**Key security controls:** ACLs between VLANs, DHCP snooping conceptually applied, port security on access switches, NAT at the perimeter.

---

## Why I Built This

Coming from a 3-year background doing L2/L3 troubleshooting on HPE-Aruba enterprise environments, I'd seen a lot of network incidents where the root cause traced back to poor segmentation — either no VLANs at all, or VLANs that existed on paper but had ACLs that basically permitted everything. This project was a chance to build something from scratch with security as a first-class design concern rather than an afterthought.

The Packet Tracer environment is obviously limited vs. a real deployment. There's no actual traffic generation, the firewall simulation is simplified, and you can't do real packet captures. But the configuration logic transfers — the Cisco IOS commands are the same, the VLAN tagging behavior is the same, and the ACL hit counts work the same way for validation.

---

## Related Skills / Tools Used

- Cisco IOS (Packet Tracer 8.2)
- VLANs, 802.1Q trunking, inter-VLAN routing (Router-on-a-Stick + L3 switch SVI)
- Extended ACLs, named ACLs
- DHCP server configuration (Cisco IOS)
- NAT overload (PAT)
- Mermaid for diagrams

---

## Disclaimer

This is a lab environment. IP addresses used are RFC 1918 private ranges. No real production credentials, hostnames, or sensitive data are included anywhere in this repo.
