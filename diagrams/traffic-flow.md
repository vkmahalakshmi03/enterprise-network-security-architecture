# Traffic Flow Diagrams

## HR Workstation → Internet

```mermaid
sequenceDiagram
    participant HR as HR PC 192.168.10.x
    participant ACC as Access Switch ACC-A-F1
    participant DIST as Distribution DIST-A
    participant CORE as Core Switch SW-CORE L3
    participant RT as Core Router RT-CORE
    participant ISP as Internet

    HR->>ACC: Frame tagged VLAN 10
    ACC->>DIST: 802.1Q trunk VLAN 10
    DIST->>CORE: 802.1Q trunk VLAN 10
    Note over CORE: ACL-VLAN10-IN evaluated\npermit ip ... any matches\n(destination is not internal VLAN)
    CORE->>RT: Routed via 10.0.0.0/30 link
    Note over RT: NAT overload\n192.168.10.x → 203.0.113.2:portX
    RT->>ISP: Source: 203.0.113.2
    ISP-->>RT: Response to 203.0.113.2:portX
    RT-->>CORE: De-NATed to 192.168.10.x
    CORE-->>DIST: Routed back to VLAN 10
    DIST-->>ACC: 802.1Q VLAN 10
    ACC-->>HR: Frame delivered
```

---

## HR PC → Finance PC (Should Be Blocked)

```mermaid
sequenceDiagram
    participant HR as HR PC 192.168.10.50
    participant CORE as Core Switch SW-CORE L3
    participant FIN as Finance PC\n192.168.20.50

    HR->>CORE: Packet: src 192.168.10.50\ndst 192.168.20.50
    Note over CORE: ACL-VLAN10-IN evaluated\nRule 50: deny ip 192.168.10.0/24 192.168.20.0/24\nMATCH — packet dropped
    Note over CORE: Hit counter on rule 50 increments
    CORE--xHR: No response (packet silently dropped)
    Note over FIN: Packet never arrives
```

---

## Finance PC → File Server (Should Work)

```mermaid
sequenceDiagram
    participant FIN as Finance PC 192.168.20.60
    participant CORE as Core Switch SW-CORE L3
    participant SRV as File Server 192.168.40.30

    FIN->>CORE: TCP SYN: dst 192.168.40.30 port 445
    Note over CORE: ACL-VLAN20-IN evaluated\nRule 50: permit tcp .20/24 host .40.30 eq 445\nMATCH — permitted
    CORE->>SRV: Routed to VLAN 40
    SRV-->>CORE: TCP SYN-ACK: src .40.30, dst .20.60
    Note over CORE: ACL-VLAN40-IN evaluated\nRule 60: permit tcp host .40.30 .20/24 established\nMATCH — return traffic permitted
    CORE-->>FIN: SMB session established
```

---

## DHCP Discovery Flow (Client → Centralized Server)

```mermaid
sequenceDiagram
    participant PC as Finance PC no IP yet
    participant ACC as ACC-B-F1
    participant DIST as DIST-B
    participant CORE as SW-CORE SVI Vlan20
    participant DHCP as DHCP Server\n192.168.40.10

    PC->>ACC: DHCP Discover\n(broadcast 255.255.255.255)
    Note over ACC: Broadcast forwarded on VLAN 20
    ACC->>DIST: VLAN 20 broadcast
    DIST->>CORE: VLAN 20 broadcast to SVI
    Note over CORE: ip helper-address 192.168.40.10\nDHCP broadcast converted to unicast\nsrc: 192.168.20.1, dst: 192.168.40.10
    CORE->>DHCP: DHCP Discover (unicast relay)
    DHCP-->>CORE: DHCP Offer\n192.168.20.45 offered
    CORE-->>PC: Offer relayed back via VLAN 20
    PC->>DHCP: DHCP Request (broadcast)
    DHCP-->>PC: DHCP ACK\nIP: 192.168.20.45\nGW: 192.168.20.1\nDNS: 192.168.40.20
```

---

## IT Admin → Management VLAN (SSH to Switch)

```mermaid
sequenceDiagram
    participant IT as IT Admin PC 192.168.30.10
    participant CORE as SW-CORE L3
    participant SW as Switch SVI 192.168.99.21

    IT->>CORE: TCP SYN: dst 192.168.99.21 port 22
    Note over CORE: ACL-VLAN30-IN evaluated\nRule 10: permit tcp .30/24 .99/24 eq 22\nMATCH — permitted
    CORE->>SW: Routed to VLAN 99
    Note over SW: VTY line: transport input ssh\naccess-class MGMT-ACCESS in\nMGMT-ACCESS permits 192.168.30.0/24
    SW-->>IT: SSH session established
```

---

## HR PC Attempting to SSH Management VLAN (Should Fail)

```mermaid
sequenceDiagram
    participant HR as HR PC 192.168.10.50
    participant CORE as SW-CORE L3

    HR->>CORE: TCP SYN: dst 192.168.99.21 port 22
    Note over CORE: ACL-VLAN10-IN evaluated\nRule 80: deny ip .10/24 .99/24\nMATCH — dropped before routing
    Note over CORE: Logged if log keyword present
    CORE--xHR: No response
```
