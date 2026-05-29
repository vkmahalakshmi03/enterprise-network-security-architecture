# Security Zones & Boundaries

## Zone Map

```mermaid
graph TD
    subgraph UNTRUST["UNTRUST ZONE\nInternet / ISP"]
        INET["Public Internet"]
    end

    subgraph DMZ["PERIMETER\nFirewall + Router"]
        FW["Firewall ASA\nStateful Inspection\nPerimeter ACLs"]
        RT["Router RT-CORE\nNAT PAT\nNo Direct Internet to Internal"]
    end

    subgraph RESTRICTED["RESTRICTED ZONE\nManagement VLAN 99"]
        MGMT["Switch SVIs\nSSH Only\nIT Admin Access Only"]
    end

    subgraph PRIVILEGED["PRIVILEGED ZONE\nIT Admin VLAN 30"]
        IT["IT Workstations\nBroadest Internal Access\nCan Reach Mgmt + Servers"]
    end

    subgraph INTERNAL["INTERNAL ZONE\nUser VLANs"]
        subgraph V10["VLAN 10 - HR"]
            HR["Reception\nConference Room"]
        end
        subgraph V20["VLAN 20 - Finance-Marketing"]
            FIN["Finance\nMarketing\nIP Phones"]
        end
    end

    subgraph SERVICES["SERVICES ZONE\nServer VLAN 40"]
        DHCP["DHCP .10"]
        DNS["DNS .20"]
        FILES["File Server .30"]
    end

    INET -->|"Filtered by FW"| FW
    FW --> RT
    RT -->|"NAT Outbound Only"| INET

    RT <-->|"Routed"| PRIVILEGED
    RT <-->|"Routed"| INTERNAL
    RT <-->|"Routed"| SERVICES

    PRIVILEGED -->|"SSH port 22 only"| RESTRICTED
    PRIVILEGED -->|"Full admin"| SERVICES
    PRIVILEGED -->|"Troubleshooting established only"| INTERNAL

    INTERNAL -->|"DHCP UDP 67/68\nDNS TCP/UDP 53"| SERVICES
    V20 -->|"SMB 445\nHTTP 80/443"| FILES

    INTERNAL -->|"BLOCKED by ACL"| RESTRICTED
    INTERNAL -->|"BLOCKED by ACL"| PRIVILEGED
    V10 -->|"BLOCKED by ACL"| V20
    V20 -->|"BLOCKED by ACL"| V10
```

---

## Access Matrix

The table below shows which source zones can reach which destination zones.  
✅ = Permitted | ❌ = Denied | ⚠️ = Partially permitted (specific ports/services only)

| Source \ Destination | Internet | VLAN 10 HR | VLAN 20 Finance | VLAN 30 IT | VLAN 40 Servers | VLAN 99 Mgmt |
|---|---|---|---|---|---|---|
| **Internet** | — | ❌ | ❌ | ❌ | ❌ | ❌ |
| **VLAN 10 HR** | ✅ (NAT) | — | ❌ | ❌ | ⚠️ DHCP+DNS only | ❌ |
| **VLAN 20 Finance** | ✅ (NAT) | ❌ | — | ❌ | ⚠️ DHCP+DNS+SMB+HTTP | ❌ |
| **VLAN 30 IT** | ✅ (NAT) | ⚠️ Established only | ⚠️ Established only | — | ✅ Full | ⚠️ SSH only |
| **VLAN 40 Servers** | ✅ (NAT) | ⚠️ DHCP responses | ⚠️ DHCP+DNS+established | ⚠️ DHCP responses | — | ❌ |
| **VLAN 99 Mgmt** | ✅ (NAT) | ❌ | ❌ | ✅ | ❌ | — |

---

## Trust Levels

```mermaid
graph LR
    subgraph L4["Trust Level 4 - Highest"]
        MGMT["VLAN 99 Management\nNetwork devices only"]
    end

    subgraph L3["Trust Level 3 - Privileged"]
        IT["VLAN 30 IT Admin\nNetwork + Server access"]
    end

    subgraph L2["Trust Level 2 - Internal"]
        FIN["VLAN 20 Finance/Marketing\nBusiness user access"]
        HR["VLAN 10 HR\nUser access + PII handling"]
    end

    subgraph L1["Trust Level 1 - Services"]
        SRV["VLAN 40 Servers\nResponse traffic only"]
    end

    subgraph L0["Trust Level 0 - Untrusted"]
        INET["Internet\nAll traffic filtered"]
    end
```

Higher trust can initiate connections to equal or lower trust (with ACL permits).  
Lower trust cannot initiate connections to higher trust zones.  
This is the core segmentation principle — enforce at the distribution layer ACLs.

---

## Threat Model Mapping

| Threat Scenario | Control in Place | Gap |
|---|---|---|
| External attacker → internal network | Firewall perimeter | Firewall sim is limited in Packet Tracer |
| Phishing on HR PC → Finance data | ACL: VLAN 10 → VLAN 20 denied | Intra-VLAN not controlled |
| Compromised Finance PC → IT Admin | ACL: VLAN 20 → VLAN 30 denied | — |
| Lateral movement via Management VLAN | ACL: non-IT VLANs → VLAN 99 denied | VTY also has access-class |
| Rogue DHCP server on user VLAN | Not mitigated | DHCP snooping not configured |
| Visitor device in conference room | Partial — on VLAN 10 with HR | Should be Guest VLAN |
| Compromised server → management | ACL: VLAN 40 → VLAN 99 denied | — |
| VLAN hopping attack | Native VLAN = 99, trunk pruning | Not fully hardened |
