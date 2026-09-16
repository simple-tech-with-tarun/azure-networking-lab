# Azure Networking Lab

A hands-on collection of Azure networking labs covering core networking concepts, architecture patterns, connectivity, security, routing, and hybrid networking.

The goal of this repository is to understand **how Azure networking works in practice** by building and testing real Azure resources rather than only studying individual services.

The labs are built incrementally using Azure CLI, PowerShell, Linux utilities, and infrastructure-as-code where appropriate.

---

## 📚 What You'll Find Here

- Azure Virtual Networks
- Subnets and address spaces
- Network Security Groups
- Routing and User Defined Routes
- Network interfaces
- Public IP addresses
- Azure Load Balancer
- Application Gateway
- Azure Firewall
- Private Endpoints
- Private DNS
- Service Endpoints
- VNet Peering
- VPN Gateway
- Azure Virtual WAN
- Hub-and-spoke networking
- Hybrid connectivity
- Network troubleshooting
- Azure networking with Terraform

---

## 🧭 Lab Progress

| # | Lab | Topics | Status |
|---|---|---|---|
| 01 | [Azure Networking Foundation](./01-network-foundation/) | VNets, subnets, NSGs, traffic flow, connectivity testing | 🟢 Completed |
| 02 | Azure Routing & User Defined Routes | Route tables, system routes, UDRs, next hops | 🟢 Completed |
| 03 | VNet Peering | VNet-to-VNet connectivity and routing | 🟢 Completed |
| 04 | Private Endpoints & Private DNS | Private connectivity and name resolution | 🟢 Completed |
| 05 | Load Balancing | Azure Load Balancer and Application Gateway | 🟢 Completed |
| 06 | Azure Firewall | Network security, filtering, and traffic inspection | 🟢 Completed |
| 07 | VPN / Hybrid Connectivity | VPN Gateway and Site-to-Site connectivity | 🟢 Completed |
| 08 | Azure Virtual WAN | Virtual WAN, Virtual Hub, VPN Site, IPsec, strongSwan | 🟢 Completed |
| 09 | Hub-and-Spoke Networking | Centralized connectivity and network architecture | ⚪ Planned |
| 10 | Terraform Networking | Azure networking with Terraform | ⚪ Planned |

---

## 🌐 Azure Virtual Networks

A Virtual Network provides the fundamental network boundary for many Azure workloads.

A basic architecture:

```text
Virtual Network
│
├── Subnet
│   ├── Application
│   └── Workloads
│
├── Subnet
│   └── Private Endpoints
│
└── Subnet
    └── Network Services
```

A VNet defines the overall address space, while subnets divide that address space into smaller network segments.

---

# 🧪 Lab 01 — Azure Networking Foundation

**Status:** 🟢 Completed

The first lab establishes the basic Azure networking foundation and uses Network Security Groups to demonstrate traffic control between two subnets.

### Architecture

```text
Azure Subscription
└── Resource Group
    │
    └── VNet: 10.0.0.0/16
        │
        ├── Subnet 1: 10.0.1.0/24
        │   ├── NSG 1
        │   └── VM 1
        │
        └── Subnet 2: 10.0.2.0/24
            ├── NSG 2
            └── VM 2
```

### Resources

| Resource | Name | Configuration |
|---|---|---|
| Resource Group | `az-net-lab-rg` | `centralindia` |
| Virtual Network | `az-net-lab-vnet` | `10.0.0.0/16` |
| Subnet 1 | `az-net-lab-subnet` | `10.0.1.0/24` |
| Subnet 2 | `az-net-lab-subnet2` | `10.0.2.0/24` |
| NSG 1 | `az-net-lab-nsg1` | Subnet 1 |
| NSG 2 | `az-net-lab-nsg2` | Subnet 2 |
| VM 1 | `az-net-lab-vm1` | Subnet 1 |
| VM 2 | `az-net-lab-vm2` | Subnet 2 |

### Concepts Covered

- Resource Groups
- Azure Virtual Networks
- Address spaces
- Subnets
- CIDR notation
- Network Security Groups
- Default NSG rules
- Custom NSG rules
- NSG association
- Inbound vs outbound filtering
- NSG rule priority
- Subnet-to-subnet traffic
- Network connectivity testing
- Azure VM Run Command
- Azure CLI inspection and JMESPath queries

### Traffic Experiment

The lab demonstrates TCP/443 connectivity between the two subnets.

```text
VM 1
10.0.1.4
   │
   │ TCP/443
   ▼
VM 2
10.0.2.4
```

NSG rules were deliberately modified to demonstrate how Azure evaluates network security rules.

The experiment included:

- Denying traffic with a higher-priority rule
- Allowing traffic with a lower numerical priority
- Testing inbound and outbound filtering
- Verifying connectivity from both VMs
- Changing rule priorities
- Removing and re-associating NSGs

### Key Learning

NSG rules are evaluated according to priority.

```text
Priority 100  → Allow
Priority 300  → Deny
```

Because lower numerical values are evaluated first, the `Allow` rule matches before the `Deny` rule.

### Troubleshooting

The lab intentionally includes failed operations and corrections, including:

- Subnet outside the VNet address space
- Invalid Azure CLI arguments
- Invalid NSG rule priorities
- NSG priority conflicts
- Rule naming limitations
- Verifying effective network configuration

---

## 📖 Detailed Lab Documentation

The complete documentation for Lab 01 is available here:

**[01 — Azure Networking Foundation](./01-network-foundation/)**

The chronological command history and observations are preserved in:

**[Session Log](./01-network-foundation/session.log)**

---

# 🧪 Lab 02 — Azure Routing & User Defined Routes

**Status:** 🟢 Completed

This lab explores how Azure determines network paths using system routes and how **User Defined Routes (UDRs)** can be used to influence traffic flow.

### Concepts Covered

- Azure system routes
- Route tables
- User Defined Routes
- Route prefixes
- Next-hop types
- Route priority
- Traffic forwarding
- Network troubleshooting
- Route inspection

---

# 🧪 Lab 03 — VNet Peering

**Status:** 🟢 Completed

This lab demonstrates connectivity between Azure Virtual Networks using **VNet Peering**.

### Concepts Covered

- VNet-to-VNet connectivity
- Peering configuration
- Azure routing between peered VNets
- Private IP communication
- Peering connectivity testing
- VNet address-space considerations

---

# 🧪 Lab 04 — Private Endpoints & Private DNS

**Status:** 🟢 Completed

This lab demonstrates private access to Azure services using **Private Endpoints** and **Private DNS**.

### Concepts Covered

- Private Endpoints
- Private IP addressing
- Private DNS zones
- DNS resolution
- Private service access
- Public vs private connectivity
- Network isolation

---

# 🧪 Lab 05 — Load Balancing

**Status:** 🟢 Completed

This lab explores Azure load-balancing services and how traffic can be distributed across backend workloads.

### Services Covered

- Azure Load Balancer
- Application Gateway

### Concepts Covered

- Frontend IP configuration
- Backend pools
- Health probes
- Load-balancing rules
- Traffic distribution
- Application-layer routing
- Backend availability

---

# 🧪 Lab 06 — Azure Firewall

**Status:** 🟢 Completed

This lab explores Azure Firewall as a centralized network security and traffic filtering service.

### Concepts Covered

- Azure Firewall
- Firewall policies
- Network rules
- Application rules
- Traffic filtering
- Network routing
- Centralized security
- Traffic inspection

---

# 🧪 Lab 07 — VPN / Hybrid Connectivity

**Status:** 🟢 Completed

This lab introduces **Site-to-Site VPN connectivity** and the concepts involved in connecting networks using Azure VPN Gateway.

### Concepts Covered

- VPN Gateway
- Site-to-Site VPN
- IPsec
- IKE
- Pre-shared keys
- VPN connections
- Hybrid network connectivity
- Tunnel validation
- Connectivity troubleshooting

---

# 🧪 Lab 08 — Azure Virtual WAN

**Status:** 🟢 Completed**

This lab introduces **Azure Virtual WAN** and demonstrates how Azure VNets and an external branch network can be connected through a Microsoft-managed Virtual Hub.

The lab also establishes a **Site-to-Site IPsec VPN** between the branch environment and the Virtual WAN VPN Gateway.

### Architecture

```text
                    Azure Virtual WAN
                           │
                    ┌──────┴──────┐
                    │ Virtual Hub │
                    │ 10.0.0.0/16 │
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              │                         │
       VNet Connection            VNet Connection
              │                         │
        ┌─────┴─────┐             ┌─────┴─────┐
        │   VNet A  │             │   VNet B  │
        │10.1.0.0/16│             │10.2.0.0/16│
        └─────┬─────┘             └───────────┘
              │
          Azure VM
          10.1.1.4


             Site-to-Site IPsec VPN
                       │
                       │
              ┌────────┴────────┐
              │   VPN Gateway   │
              │  Virtual Hub    │
              └────────┬────────┘
                       │
                 Encrypted Tunnel
                       │
                ┌──────┴──────┐
                │  Branch VM  │
                │  strongSwan │
                │ 10.50.1.4   │
                │ 10.50.2.4   │
                └─────────────┘
```

### Components Covered

- Azure Virtual WAN
- Virtual Hub
- VNet Connections
- Virtual WAN VPN Gateway
- VPN Site
- VPN Site Link
- VPN Connection
- Site-to-Site IPsec
- IKEv2
- Pre-shared key authentication
- strongSwan
- IPsec traffic validation

### Lab Environment

```text
VNet A
10.1.0.0/16
└── WorkloadSubnet
    └── Azure VM
        10.1.1.4

VNet B
10.2.0.0/16

Branch VNet
10.50.0.0/16
├── WANSubnet
│   └── 10.50.1.4
│
└── LANSubnet
    └── 10.50.2.4
```

The branch VM used two network interfaces:

- WAN NIC — `10.50.1.4`
- LAN NIC — `10.50.2.4`

IP forwarding was enabled on the branch interfaces.

### Site-to-Site IPsec

The branch VM was used as a simulated on-premises VPN device.

**strongSwan** was installed on the branch VM and configured to establish an IKEv2/IPsec tunnel with the Virtual WAN VPN Gateway.

```text
Branch VM
     │
     │ strongSwan
     │
     ▼
 IKEv2 / IPsec
     │
     ▼
Virtual WAN VPN Gateway
     │
     ▼
Virtual Hub
     │
     ▼
VNet Connection
     │
     ▼
VNet A
     │
     ▼
Azure VM
```

### Tunnel Validation

The final strongSwan state showed:

```text
IKE_SA:   ESTABLISHED
CHILD_SA: INSTALLED
```

Traffic was then tested from the branch VM to the Azure workload VM.

A TCP connection to:

```text
10.1.1.4:22
```

successfully connected.

IPsec traffic counters also increased, confirming that actual traffic was traversing the encrypted tunnel.

### Key Learning

The lab demonstrated the difference between establishing the VPN security association and actually passing application traffic.

```text
IKE Security Association
          │
          ▼
Authentication
          │
          ▼
IPsec Child Security Association
          │
          ▼
Encrypted Data Traffic
          │
          ▼
Application Connectivity
```

A successful IKE session alone does not prove that application traffic is working.

### Next Phase

The next phase of the Virtual WAN lab will introduce **BGP**.

```text
Branch
   │
   │ BGP
   ▼
IPsec VPN
   │
   ▼
Virtual WAN
   │
   ▼
Virtual Hub
   │
   ▼
Azure VNets
```

BGP will be configured when this lab is recreated and extended.

---

## 🛠️ Tools

- Microsoft Azure
- Azure CLI
- PowerShell
- Linux
- strongSwan
- Git
- GitHub
- Terraform

---

## ⚠️ Lab Environment

These labs create real Azure resources and may incur Azure charges.

Resources should be removed after completing a lab unless they are intentionally being retained.

Resource names and configuration values are used primarily for repeatability and learning.

---

## 📈 Repository Philosophy

These labs are intentionally hands-on.

Rather than only documenting the expected result, the repository captures:

- What was built
- Why it was built
- Commands used
- Failed attempts
- Troubleshooting
- Observed behavior
- Configuration changes
- Lessons learned

The goal is to build practical Azure networking knowledge through experimentation rather than simply following a tutorial.

---

## 🗺️ Roadmap

The repository progressively moves from fundamental Azure networking toward more advanced connectivity and infrastructure architectures.

```text
Azure Networking Foundation
          │
          ▼
Routing & UDRs
          │
          ▼
VNet Peering
          │
          ▼
Private Endpoints & DNS
          │
          ▼
Load Balancing
          │
          ▼
Azure Firewall
          │
          ▼
VPN / Hybrid Connectivity
          │
          ▼
Azure Virtual WAN
          │
          ▼
Hub-and-Spoke
          │
          ▼
Terraform Networking
```

Each lab builds on the networking concepts introduced earlier while gradually increasing the architectural complexity.