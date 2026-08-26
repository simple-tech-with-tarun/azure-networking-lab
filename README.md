# Azure Networking Lab

A hands-on collection of Azure networking concepts, configurations,
architecture patterns, and troubleshooting scenarios.

This repository focuses on understanding how Azure networking components
work individually and how they interact to build secure, scalable, and
well-connected cloud environments.

The labs are built incrementally using Azure CLI, PowerShell, Linux
utilities, and infrastructure-as-code where appropriate.

---

## 📚 What You'll Find Here

- Azure Virtual Networks
- Subnets
- Network Security Groups
- Route tables
- User Defined Routes
- Public IP addresses
- Network interfaces
- Azure Load Balancer
- Application Gateway
- Azure Firewall
- Private Endpoints
- Private DNS
- Service Endpoints
- VPN Gateway
- VNet peering
- Hub-and-spoke networking
- Network troubleshooting
- Azure networking with Terraform

---

## 🧭 Lab Progress

| # | Lab | Topics | Status |
|---|---|---|---|
| 01 | [Azure Networking Foundation](./01-network-foundation/) | VNets, subnets, NSGs, traffic flow, connectivity testing | 🟢 Completed |
| 02 | Network Security & Traffic Control | Advanced NSG scenarios | ⚪ Planned |
| 03 | Routing | Route tables and UDRs | ⚪ Planned |
| 04 | VNet Connectivity | VNet peering | ⚪ Planned |
| 05 | Private Networking | Private Endpoints and Private DNS | ⚪ Planned |
| 06 | Azure Load Balancing | Load Balancer and Application Gateway | ⚪ Planned |
| 07 | Network Security | Azure Firewall | ⚪ Planned |
| 08 | Hybrid Connectivity | VPN Gateway | ⚪ Planned |
| 09 | Infrastructure as Code | Azure networking with Terraform | ⚪ Planned |

---

## 🌐 Azure Virtual Networks

A Virtual Network provides the fundamental network boundary for many
Azure workloads.

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

A VNet defines the overall address space, while subnets divide that
address space into smaller network segments.

---

# 🧪 Lab 01 — Azure Networking Foundation

**Status:** 🟢 Completed

The first lab establishes the basic Azure networking foundation and
then uses Network Security Groups to demonstrate traffic control
between two subnets.

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

NSG rules were deliberately modified to demonstrate how Azure evaluates
network security rules.

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

Because lower numerical values are evaluated first, the `Allow` rule
matches before the `Deny` rule.

### Troubleshooting

The lab intentionally includes failed operations and corrections,
including:

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

## 🛠️ Tools

- Microsoft Azure
- Azure CLI
- PowerShell
- Linux
- Git
- GitHub
- Terraform

---

## ⚠️ Lab Environment

These labs create real Azure resources and may incur Azure charges.

Resources should be removed after completing a lab unless they are
intentionally being retained.

Resource names and configuration values are used primarily for
repeatability and learning.

---

## 📈 Repository Philosophy

These labs are intentionally hands-on.

Rather than only documenting the expected result, the repository
captures:

- What was built
- Why it was built
- Commands used
- Failed attempts
- Troubleshooting
- Observed behavior
- Configuration changes
- Lessons learned

The goal is to build practical Azure networking knowledge through
experimentation rather than simply following a tutorial.

---

## 🗺️ Roadmap

The repository will progressively move from basic Azure networking
toward more complex architectures.

```text
Network Foundation
       │
       ▼
NSGs & Traffic Control
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
Hybrid Connectivity
       │
       ▼
Terraform
```
```

### One important structural point

I **wouldn't call the next lab `02 — NSGs`** anymore.

We've already gone well beyond basic NSG creation in Lab 01. We created two NSGs, associated them to subnets, created allow/deny rules, tested priorities, and actually tested traffic between two VMs.

So I'd consider **NSGs part of the foundation lab**.

That gives us a much cleaner progression:

```text
01 — Azure Networking Foundation
02 — Azure Routing & User Defined Routes
03 — VNet Peering
04 — Private Endpoints & Private DNS
05 — Load Balancing
06 — Azure Firewall
07 — VPN / Hybrid Connectivity
08 — Hub-and-Spoke
09 — Terraform Networking
```