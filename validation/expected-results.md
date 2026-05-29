# Expected Results

For each test case in `test-cases.md`, this documents the expected behavior in detail — what a passing test looks like and what commands to run to confirm it.

---

## TC-01 Expected: Intra-VLAN Connectivity

**Pass criteria:**
- Ping succeeds (0% packet loss in Packet Tracer)
- Response time < 1ms (same broadcast domain, no routing)
- `show mac address-table` on the access switch shows both source and destination MACs in the same VLAN

**What failure looks like:**
- 100% packet loss on ping within same VLAN → check access port VLAN assignment
- Run `show interfaces fastEthernet 0/X` on access switch — confirm `Access Mode VLAN: 20` (or whichever VLAN expected)

---

## TC-02 Expected: Internet Access

**Pass criteria:**
- Ping to simulated external IP succeeds
- `show ip nat translations` shows entry like:
  ```
  Pro Inside global    Inside local       Outside local      Outside global
  icmp 203.0.113.2:3   192.168.10.50:3   8.8.8.8:3          8.8.8.8:3
  ```
- `show ip nat statistics` shows increasing "Hits" count

**What failure looks like:**
- No NAT translation entry → check `ip nat inside/outside` on interfaces
- Translation exists but no reply → check default route `ip route 0.0.0.0 0.0.0.0 203.0.113.1`
- VLAN 30 traffic not NAT'd → check NAT ACL includes 192.168.30.0/24

---

## TC-03 Expected: DHCP Assignment

**Pass criteria:**
- Client receives IP from correct subnet (verify with `ipconfig` or equivalent)
- Gateway = 192.168.x.1 for that VLAN
- DNS = 192.168.40.20
- `show ip dhcp binding` on server shows MAC-to-IP mapping

```
IP address       Hardware address    Lease expiration        Type
192.168.20.21    00D0.BC3A.1234      Sep 29 2025 08:00 AM    Automatic
```

**What failure looks like:**
- Client shows 169.254.x.x (APIPA) → DHCP failed entirely. Check:
  1. `show ip interface Vlan20` → confirm `Helper address is 192.168.40.10`
  2. ACL on VLAN 20 SVI allows UDP 67/68 to 192.168.40.10 (rules 10-20)
  3. DHCP pool defined for 192.168.20.0/24
  4. Excluded address range doesn't cover entire subnet

---

## TC-04 Expected: Inter-VLAN Blocking

**Pass criteria:**
- Ping fails (100% packet loss, no reply)
- `show ip access-lists ACL-VLAN10-IN` shows the deny rule with hit count > 0:

```
Extended IP access list ACL-VLAN10-IN
    10 permit udp 192.168.10.0 0.0.0.255 host 192.168.40.10 eq 67
    20 permit udp 192.168.10.0 0.0.0.255 host 192.168.40.10 eq 68
    30 permit tcp 192.168.10.0 0.0.0.255 host 192.168.40.20 eq 53
    40 permit udp 192.168.10.0 0.0.0.255 host 192.168.40.20 eq 53
    50 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255 (4 matches)  ← this increments
    ...
```

**Important:** If ping fails but the deny rule shows 0 hits, the packet is being dropped somewhere else (possibly no route, wrong gateway). That's not the same as ACL enforcement — distinguish between "dropped by ACL" and "dropped because unreachable."

**What the ordering bug looked like (TC-08 regression):**
```
    50 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255 (0 matches)
    90 permit ip 192.168.10.0 0.0.0.255 any (12 matches)
```
The permit rule hit count was incrementing for *all* traffic including cross-VLAN, meaning it was matching before the deny. Moving the internet permit below the VLAN deny rules fixed this.

---

## TC-05 Expected: Permitted Services

**Pass criteria (TC-05a — Finance to file server SMB):**
- TCP connection establishes (SYN → SYN-ACK → ACK)
- `show ip access-lists ACL-VLAN20-IN` rule 50 shows hits
- `show ip access-lists ACL-VLAN40-IN` rule 60 shows hits (return traffic permitted)

The return traffic test is easy to miss. Both directions need to work:
- Finance → Server: ruled by ACL-VLAN20-IN rule 50 (outbound request)
- Server → Finance: ruled by ACL-VLAN40-IN rule 60 (`established` keyword)

If the request goes through but no response comes back, check the server-side ACL.

---

## TC-06 Expected: Management VLAN

**Pass criteria (IT SSH to switch):**
- SSH session establishes
- `show ip access-lists ACL-VLAN30-IN` rule 10 shows hits
- `show users` on the switch shows active SSH session

**Pass criteria (Finance blocked from Mgmt):**
- Connection times out (not refused — the ACL drops before TCP RST can be sent)
- `show ip access-lists ACL-VLAN99-IN` rule 30 shows hits (deny with log)

**Note on timeout vs refused:**
A TCP RST (connection refused) means the packet reached the destination and was rejected at the application layer. A timeout means the packet was dropped before it arrived. ACL drops cause timeouts, not RST. If you see a TCP RST from a management SVI for a Finance connection, the ACL isn't working.

---

## TC-09 Expected: Trunk Verification

Expected output of `show interfaces trunk` on DIST-B:

```
Port        Mode             Encapsulation  Status        Native vlan
Gi0/1       on               802.1q         trunking      99

Port        Vlans allowed on trunk
Gi0/1       20,99

Port        Vlans allowed and active in management domain
Gi0/1       20,99

Port        Vlans in spanning tree forwarding state and not pruned
Gi0/1       20,99
```

VLAN 10, 30, 40 should NOT appear. If they do, check the trunk allowed vlan config.

Expected output of `show vlan brief` on ACC-B-F1:

```
VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active
20   Finance-Marketing                active    Fa0/1, Fa0/2, ... Fa0/20
99   Management                       active
```

VLAN 1 will appear (default, unavoidable) but should have no ports assigned. VLANs 10, 30, 40 should not appear at all on this switch.
