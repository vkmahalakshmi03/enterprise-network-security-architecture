# Enterprise Network Security Architecture

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
