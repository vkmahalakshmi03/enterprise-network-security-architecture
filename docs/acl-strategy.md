# ACL Strategy

## Design Philosophy

The ACL rules here follow a default-deny approach at the inter-VLAN boundary, with explicit permits for traffic that has a legitimate business reason. This is the opposite of the "permit everything, deny known bad" approach that's unfortunately common — and that shows up in incident postmortems regularly.

The logic: if you can't articulate *why* VLAN A needs to reach VLAN B, the default answer is no.

Named ACLs are used throughout instead of numbered ACLs. Named ACLs are significantly easier to manage — you can add/remove individual rules by sequence number without rewriting the entire list.

---

## ACL Placement Decision

ACLs are applied **inbound on the distribution switch SVIs** — meaning they evaluate traffic as it arrives on the L3 interface from a given VLAN before the switch makes a routing decision.

**Why inbound vs outbound:**  
Inbound ACLs evaluate traffic before it enters the routing process. This is more efficient — traffic that's going to be denied gets dropped at the ingress point rather than being routed first and then dropped. For a distribution switch handling inter-VLAN traffic, this matters.

The alternative (outbound ACL on the destination SVI) would work but means traffic travels further before being dropped. Also harder to reason about — "what reaches VLAN X" vs "what leaves VLAN Y."

---

## Core ACL Rules by VLAN

### From VLAN 10 (HR) — Applied on SVI Gi0/0.10

```
permit udp 192.168.10.0 0.0.0.255 host 192.168.40.10 eq 67    ! DHCP relay
permit udp 192.168.10.0 0.0.0.255 host 192.168.40.10 eq 68    ! DHCP response
permit tcp 192.168.10.0 0.0.0.255 host 192.168.40.20 eq 53    ! DNS
permit udp 192.168.10.0 0.0.0.255 host 192.168.40.20 eq 53    ! DNS UDP
permit ip 192.168.10.0 0.0.0.255 any                          ! Internet (NAT'd at router)
deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255         ! Block HR → Finance
deny ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255         ! Block HR → IT
deny ip 192.168.10.0 0.0.0.255 192.168.99.0 0.0.0.255         ! Block HR → Mgmt
deny ip any any                                                ! Implicit deny (explicit for logging)
```

**Note on the "permit ip any any" before the denies:** That ordering is wrong above — fixed in the actual config file. In ACLs, order matters. The internet permit has to come after the specific denies if you want inter-VLAN blocks to take effect. See `configs/acl-rules.txt` for the correct ordered ruleset.

This was a mistake I caught during testing when HR → Finance traffic was still going through. Traced it back to ACL ordering. The `permit ip ... any` was above the deny rules, so it was matching first. Classic ACL ordering trap.

---

### From VLAN 20 (Finance/Marketing) — Applied on SVI Gi0/0.20

```
permit udp 192.168.20.0 0.0.0.255 host 192.168.40.10 eq 67    ! DHCP
permit udp 192.168.20.0 0.0.0.255 host 192.168.40.10 eq 68
permit tcp 192.168.20.0 0.0.0.255 host 192.168.40.20 eq 53    ! DNS
permit udp 192.168.20.0 0.0.0.255 host 192.168.40.20 eq 53
permit tcp 192.168.20.0 0.0.0.255 192.168.40.0 0.0.0.255 eq 445  ! SMB to file server
permit tcp 192.168.20.0 0.0.0.255 192.168.40.0 0.0.0.255 eq 80   ! HTTP to internal apps
deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255         ! Block Finance → HR
deny ip 192.168.20.0 0.0.0.255 192.168.30.0 0.0.0.255         ! Block Finance → IT
deny ip 192.168.20.0 0.0.0.255 192.168.99.0 0.0.0.255         ! Block Finance → Mgmt
permit ip 192.168.20.0 0.0.0.255 any                          ! Internet
deny ip any any
```

**Why SMB (445) to VLAN 40:** Finance needs to reach shared file servers. Allowing the entire VLAN 40 subnet via SMB is broader than ideal — in a production environment you'd restrict this to specific server IPs. Noted as a tuning item.

---

### From VLAN 30 (IT/Admin) — Applied on SVI Gi0/0.30

```
permit ip 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255       ! IT → Mgmt VLAN
permit ip 192.168.30.0 0.0.0.255 192.168.40.0 0.0.0.255       ! IT → Servers (admin)
permit tcp 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255 established  ! IT troubleshooting (return traffic)
permit tcp 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255 established
permit ip 192.168.30.0 0.0.0.255 any                          ! Internet
deny ip any any
```

**The IT ACL debate:** IT needs broad access by job function. The `established` keyword allows return traffic from connections IT initiates — this means IT can SSH/ping into HR/Finance VLANs for troubleshooting but unsolicited traffic from those VLANs can't initiate connections back to IT. Not perfect isolation but better than fully open bidirectional.

This is one of those decisions where the "right" answer depends on the organization's risk tolerance and how much you trust insider threat controls. Logging on these rules would be valuable — you'd want to know if IT workstations are making unusual connections into Finance at 2am.

---

### From VLAN 40 (Servers) — Applied on SVI Gi0/0.40

```
permit udp host 192.168.40.10 192.168.10.0 0.0.0.255 eq 68    ! DHCP offers to HR
permit udp host 192.168.40.10 192.168.20.0 0.0.0.255 eq 68    ! DHCP offers to Finance
permit udp host 192.168.40.10 192.168.30.0 0.0.0.255 eq 68    ! DHCP offers to IT
permit tcp host 192.168.40.20 any eq 53                        ! DNS responses
permit udp host 192.168.40.20 any eq 53
deny ip 192.168.40.0 0.0.0.255 192.168.99.0 0.0.0.255         ! Servers can't reach Mgmt
permit ip 192.168.40.0 0.0.0.255 any                          ! Servers → internet if needed
deny ip any any
```

**Servers can't reach management:** Even the server VLAN is blocked from the management VLAN. If a server gets compromised, you don't want it to have a path to switch management interfaces.

---

### VLAN 99 (Management) — Applied on SVI Gi0/0.99

```
permit tcp 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255 eq 22   ! SSH from IT only
deny ip any 192.168.99.0 0.0.0.255                               ! Block everything else to Mgmt
deny ip any any
```

Most restrictive VLAN. Only IT admins can SSH into network devices. No other VLANs have any path here.

---

## ACL Logging

For production use, I'd add `log` keyword to the explicit deny statements:
```
deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255 log
```

This generates syslog messages for denied traffic, which would feed into a SIEM for alerting. In Packet Tracer this doesn't generate actual logs to an external system, but the rule is worth documenting. In my previous role doing enterprise support on HPE-Aruba, ACL hit counts without logging were the main visibility we had — adding `log` would've caught a lot of things earlier.

---

## What the ACLs Don't Cover

1. **Intra-VLAN traffic** — Two HR workstations can still communicate freely. Private VLANs or host-based firewall would be needed.
2. **East-west encrypted traffic** — If two VLANs that are supposed to be isolated are tunneling traffic over HTTPS, ACLs at port 443 won't help without deep packet inspection.
3. **IPv6** — These ACLs are IPv4 only. If IPv6 is enabled on any VLAN, separate IPv6 ACLs are required or traffic bypasses these controls entirely.
