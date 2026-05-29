# Network Topology Diagrams

> **Note:** These Mermaid diagrams render on GitHub. Lucidchart versions with full visual detail are maintained separately. The Mermaid versions capture logical relationships and are useful for quick reference in code review/documentation contexts.

---

## Physical Topology

```mermaid
graph TD
    ISP["ISP / Internet"]
    FW["Firewall ASA 5505"]
    RT["Core Router RT-CORE\n203.0.113.2"]
    SWCORE["Core Switch SW-CORE\nL3 Cisco 3560"]

    DISTA["DIST-A Building A\nCisco 2960"]
    DISTB["DIST-B Building B\nCisco 2960"]
    DISTC["DIST-C Building C\nCisco 2960"]

    ACCA1["ACC-A-F1 Bldg A Floor 1"]
    ACCB1["ACC-B-F1"]
    ACCB2["ACC-B-F2"]
    ACCB3["ACC-B-F3"]
    ACCB4["ACC-B-F4"]
    ACCB5["ACC-B-F5"]
    ACCC1["ACC-C-F1"]
    ACCC2["ACC-C-F2"]
    ACCC3["ACC-C-F3"]
    ACCC4["ACC-C-F4"]
    ACCC5["ACC-C-F5"]

    SERVERS["Server VLAN 40\nDHCP + DNS + File"]

    ISP --> FW --> RT --> SWCORE

    SWCORE --> DISTA
    SWCORE --> DISTB
    SWCORE --> DISTC
    SWCORE --> SERVERS

    DISTA --> ACCA1

    DISTB --> ACCB1
    DISTB --> ACCB2
    DISTB --> ACCB3
    DISTB --> ACCB4
    DISTB --> ACCB5

    DISTC --> ACCC1
    DISTC --> ACCC2
    DISTC --> ACCC3
    DISTC --> ACCC4
    DISTC --> ACCC5
```

---

## Logical Topology (VLAN View)

```mermaid
graph LR
    INET["Internet"]

    subgraph Edge
        FW["Firewall"]
        RT["Core Router\nNAT/PAT"]
    end

    subgraph Core
        SWCORE["Core Switch L3\nInter-VLAN Routing\nACL Enforcement"]
    end

    subgraph V10["VLAN 10 HR 192.168.10.0/24"]
        HR1["Reception PCs"]
        HR2["HR Printers"]
        CONF["Conference Room note: should be Guest VLAN"]
    end

    subgraph V20["VLAN 20 Finance-Marketing 192.168.20.0/24"]
        FIN["Finance Workstations"]
        MKT["Marketing Workstations"]
        PHONES["IP Phones"]
    end

    subgraph V30["VLAN 30 IT-Admin 192.168.30.0/24"]
        IT["IT Workstations"]
        ADM["Admin PCs"]
    end

    subgraph V40["VLAN 40 Servers 192.168.40.0/24"]
        DHCP["DHCP Server .10"]
        DNS["DNS Server .20"]
        FILES["File Server .30"]
    end

    subgraph V99["VLAN 99 Management 192.168.99.0/24"]
        MGMT["Switch SVIs\nSSH from IT only"]
    end

    INET --> FW --> RT --> SWCORE
    SWCORE <--> V10
    SWCORE <--> V20
    SWCORE <--> V30
    SWCORE <--> V40
    SWCORE <--> V99
```

---

## Building to VLAN Mapping

```mermaid
graph TD
    subgraph BA["Building A - 1 Floor Reception"]
        A1["Floor 1 - VLAN 10 HR\nReception + Conference"]
    end

    subgraph BB["Building B - 5 Floors"]
        B1["Floor 1 - VLAN 20 Finance"]
        B2["Floor 2 - VLAN 20 Finance"]
        B3["Floor 3 - VLAN 20 Finance"]
        B4["Floor 4 - VLAN 20 Marketing"]
        B5["Floor 5 - VLAN 20 Marketing"]
    end

    subgraph BC["Building C - 5 Floors"]
        C1["Floor 1 - VLAN 20 Finance"]
        C2["Floor 2 - VLAN 20 Finance"]
        C3["Floor 3 - VLAN 20 Finance"]
        C4["Floor 4 - VLAN 30 IT Admin"]
        C5["Floor 5 - VLAN 30 IT Admin"]
    end
```

---

## Trunk VLAN Pruning

```mermaid
graph LR
    SWCORE["SW-CORE\nAll VLANs 10,20,30,40,99"]

    SWCORE -->|"10,99"| DISTA["DIST-A"]
    SWCORE -->|"20,99"| DISTB["DIST-B"]
    SWCORE -->|"20,30,40,99"| DISTC["DIST-C"]

    DISTA -->|"10,99"| ACCA["ACC-A-F1"]
    DISTB -->|"20,99"| ACCB["ACC-B F1-F5"]
    DISTC -->|"20,99"| ACCC13["ACC-C F1-F3"]
    DISTC -->|"30,99"| ACCC45["ACC-C F4-F5"]
```

Trunk pruning: each trunk only carries VLANs needed downstream.
Building B distribution switch never carries VLAN 30 or 10 — no devices of those types are connected there.
