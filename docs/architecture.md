# Network Architecture

## The Design Problem

The brief was a three-building campus:
- **Building A** – 1-story, reception and conference rooms
- **Building B** – 5 floors, primary office space  
- **Building C** – 5 floors, primary office space  

Standard requirement: connect everything, provide internet access, keep departments separated.

The part that made this interesting was the security angle. A flat network here would've been straightforward to implement, but it would've meant HR computers could reach Finance file servers, guest devices in the conference room could reach internal systems, and an IT admin's workstation would have the same network-layer access as a reception desk PC. That's the kind of thing that looks fine until something goes wrong.

---

## Physical Topology

The physical design is a hybrid of star and bus.

**Within each floor/building:** Star topology. Every end device connects to an access switch. Single point of cable management per floor, easy to troubleshoot — if a device goes offline, you're looking at that one port, not tracing a chain of devices.

**Across floors within a building:** Bus-like backbone. Each floor's access switch uplinks to the building distribution switch. The distribution switch is the aggregation point for that building's traffic before it hits the core.

**Across buildings:** Everything converges at the core switch/router, which is the single point connecting all three buildings and providing the path to the internet.

```
ISP
 |
[Firewall]
 |
[Core Router]
 |
[Core Switch]
 /    |    \
[Dist-A] [Dist-B] [Dist-C]
 |         |         |
[Access] [Floor 1-5] [Floor 1-5]
switches  switches    switches
```

**Why not a full mesh?** Cost and complexity. A mesh between three buildings with 5 floors each would require a significant amount of inter-building cabling with limited operational benefit for this scale. The hierarchical model handles failures reasonably well — if Building B's distribution switch goes down, Buildings A and C still have connectivity.

**Single point of failure concern:** The core switch is a real concern in this design. In a production environment you'd want a redundant core or at minimum a hot standby using HSRP/VRRP. This lab doesn't implement that, but it's called out as a gap.

---

## Logical Topology

The logical design is hierarchical (three-tier: core, distribution, access) with VLAN-based segmentation at the distribution layer.

### Three-Tier Hierarchy

**Core layer:** Core switch handles inter-building routing. Trunk links carrying all VLANs between buildings.

**Distribution layer:** Per-building distribution switches handle VLAN aggregation and apply inter-VLAN routing (via SVIs). ACLs are applied at this layer to control what traffic can move between VLANs.

**Access layer:** Per-floor access switches connect end devices. Ports configured as access ports, assigned to the appropriate VLAN for that department.

### Why Apply ACLs at Distribution, Not Core?

The initial instinct might be to centralize ACL enforcement at the core. The problem: at the core layer, you're making routing decisions for all traffic across all buildings. Applying ACLs there means every inter-VLAN packet gets evaluated against a potentially large ruleset at the most latency-sensitive point in the network.

At the distribution layer, ACLs are applied closer to where the traffic originates. The rule evaluation happens before traffic is forwarded to the core. This keeps the core doing what it does best (fast switching between buildings) and pushes policy enforcement down.

In practice at scale this is also a management question — per-building distribution switches let you apply building-specific policies without touching the shared core config.

### IP Addressing Scheme

| VLAN | Name | Subnet | Gateway | Usable Range |
|------|------|--------|---------|--------------|
| 10 | HR | 192.168.10.0/24 | 192.168.10.1 | .2 – .254 |
| 20 | Finance-Marketing | 192.168.20.0/24 | 192.168.20.1 | .2 – .254 |
| 30 | IT-Admin | 192.168.30.0/24 | 192.168.30.1 | .2 – .254 |
| 40 | Servers | 192.168.40.0/24 | 192.168.40.1 | .2 – .254 |
| 99 | Management | 192.168.99.0/24 | 192.168.99.1 | .2 – .254 |

/24 subnets give 254 usable addresses per VLAN — more than enough for a campus this size and leaves room to grow.

The management VLAN (99) is used for switch management interfaces. It's intentionally separate from user VLANs and access to it is restricted via ACL to IT Admin VLAN only.

---

## Device Inventory (Lab Topology)

| Device | Type | Count | Notes |
|--------|------|-------|-------|
| Core Router | Cisco 2911 | 1 | Inter-building routing, NAT |
| Core Switch | Cisco 3560 (L3) | 1 | VLAN routing, trunk aggregation |
| Distribution Switch | Cisco 2960 | 3 | One per building |
| Access Switch | Cisco 2960 | 8 | Per floor (B1 = 1, B2/C = 5 floors × 1 each + 1 per building) |
| DHCP Server | Generic server | 1 | Centralized DHCP for all VLANs |
| Firewall | ASA 5505 (simulated) | 1 | Perimeter |

Total active network devices: 10–12 depending on count

---

## Design Decisions I'd Change

A few things I'd approach differently in a real deployment:

1. **Redundant core** – HSRP or dual core switches for failover. Single core is a clear availability gap.
2. **DHCP snooping** – Should be enabled on all access switch ports to prevent rogue DHCP servers. Packet Tracer supports this but I didn't configure it in the initial lab pass.
3. **802.1X port authentication** – Access switch ports should ideally require authentication before being assigned to a VLAN. Not implemented here.
4. **Separate guest VLAN** – Conference room devices (visitor laptops, etc.) should be on an isolated guest VLAN with internet-only access and no path to internal VLANs. VLAN 10 as configured gives HR devices access — conference room devices should be split out.

These are documented in `security-analysis.md` with more detail.
