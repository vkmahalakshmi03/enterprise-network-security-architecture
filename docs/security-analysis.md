# Security Analysis & Observations

This is the part that isn't usually in a clean writeup — what I actually learned building and testing this, including what broke, what surprised me, and what I'd do differently.

---

## What This Design Gets Right

### Segmentation Is the Right First Principle

The core insight that drove this design: perimeter-only security (firewall at the edge, everything inside trusted) doesn't hold. Inside the network, if segmentation is flat, lateral movement after a breach is trivial.

In three years of enterprise network support for HPE-Aruba environments, I saw incidents repeatedly where the breach itself was limited — a compromised endpoint, a phishing success — but the impact was amplified because internal segmentation either didn't exist or wasn't enforced. The ACLs existed in the config but were too permissive.

Building this from scratch forced the question: for every inter-VLAN permit, why? That discipline is harder to maintain when you're inheriting an existing configuration under pressure.

### DHCP Centralization Works Better Than Expected

Having a single DHCP server in VLAN 40 relay addresses to all VLANs via `ip helper-address` on each SVI is cleaner than I expected. One pool per VLAN, one place to look for lease issues. The alternative — putting a DHCP server in each VLAN — creates more moving parts without obvious security benefit.

The one thing this requires: the `ip helper-address` config has to be right on every SVI. If you miss it on one VLAN, devices on that VLAN silently fail to get addresses. This happened during testing with VLAN 20 — spent longer than I'd like to admit before checking the SVI config.

---

## What I'd Change

### 1. Guest/Conference VLAN — Significant Gap

The current design puts conference room devices in VLAN 10 (HR). This made sense as a physical grouping (same building) but is wrong from a security standpoint. Conference room devices include visitor laptops, personal phones, guest speakers — none of these should be on the same VLAN as HR systems.

**Fix:** VLAN 50 – Guest. Subnet 192.168.50.0/24. ACL permits internet only (via NAT), explicit deny to all internal VLANs. Isolated completely.

### 2. No DHCP Snooping

DHCP snooping should be enabled on all access switch ports. Without it, a rogue DHCP server brought onto any VLAN can start handing out addresses — wrong gateway, wrong DNS, or an attacker-controlled DNS. This is a straightforward attack to execute and snooping is a straightforward fix.

```
ip dhcp snooping
ip dhcp snooping vlan 10,20,30,40
no ip dhcp snooping information option
interface range Gi0/1 - 24
 ip dhcp snooping limit rate 15
```

Trusted uplink ports (toward distribution switch) get `ip dhcp snooping trust`. Everything else is untrusted by default.

### 3. No 802.1X

Port authentication would require devices to authenticate before being assigned a VLAN. Currently, any device plugged into an access port gets an IP from DHCP and is on the network. 802.1X with RADIUS backend would fix this — unauthenticated devices get quarantined VLAN or no VLAN.

This is a significant implementation effort but the security gain is real. In a healthcare or financial environment it would likely be a compliance requirement (CMMC, PCI-DSS).

### 4. Native VLAN

Access switch trunks should have native VLAN set to an unused VLAN (not VLAN 1, not any VLAN with user traffic) to reduce VLAN hopping surface. I used VLAN 99 as native on trunk links, which is better than VLAN 1 but ideally the native VLAN would be a dedicated "parking" VLAN with no assigned devices.

### 5. Monitoring Gap

This design has no monitoring. In a real deployment you'd want:
- Syslog from all network devices → centralized syslog server
- SNMP polling → NMS (network management system)
- ACL deny logging → SIEM

Without monitoring, a misconfigured ACL letting unauthorized traffic through would go undetected. In production, I'd set up Splunk (or at minimum syslog + basic alerting) to catch anomalies in denied traffic counts.

---

## Unexpected Learnings

### ACL Order Failures Are Quiet

When the HR → Finance deny wasn't working, there was no error message. Traffic just went through. Diagnosing this required doing `show ip access-lists` and checking hit counts on each rule, then working backward to figure out which permit was matching before the deny.

The lesson: ACLs fail silently when they're wrong. Testing requires actually generating traffic and verifying hit counts — not just reviewing the config.

### `ip helper-address` Scope Is More Specific Than Expected

I assumed `ip helper-address` forwarded all DHCP broadcasts. It actually forwards UDP broadcasts for a specific set of ports by default (DHCP/BOOTP, TFTP, DNS, and a few others). This is fine for this use case but worth knowing if you're trying to relay other UDP broadcast services.

### Packet Tracer Firewall Limitations

The Packet Tracer ASA simulation doesn't fully model stateful inspection behavior. Some of the ACL testing had weird behavior that I couldn't fully attribute to configuration vs. simulation artifacts. Real-world testing on GNS3 or Eve-NG would be more reliable for ACL validation. I'd treat the Packet Tracer results as directionally correct but not as a substitute for testing on real hardware.

---

## Overall Assessment

For a campus of this size and departmental structure, this design provides a reasonable security baseline. The VLANs create meaningful isolation between departments, the ACLs enforce that isolation at the routing layer, and the centralized DHCP reduces management complexity.

The main gaps — guest VLAN, DHCP snooping, 802.1X, monitoring — are all real gaps that would need addressing before this design was production-ready. None of them are design-level problems; they're implementation items that were out of scope for this lab exercise.

The bigger takeaway from this project was less about the specific configuration and more about the mindset: every permit in an ACL should have a justification. If you can't explain why a path needs to be open, it probably shouldn't be.
