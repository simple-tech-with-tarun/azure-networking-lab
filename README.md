# Azure Networking Lab

A hands-on collection of Azure networking concepts, configurations, architecture patterns, and troubleshooting scenarios.

This repository focuses on understanding how Azure networking components work individually and how they interact to build secure, scalable, and well-connected cloud environments.

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

## Lab Progress

### 01 — Azure Networking Foundation

**Status:** Completed

#### Architecture

```text
Azure Subscription
└── Resource Group
    └── Virtual Network
        └── Subnet
```

#### Resources Created

| Resource | Name | Configuration |
|---|---|---|
| Resource Group | `az-net-lab-rg` | `centralindia` |
| Virtual Network | `az-net-lab-vnet` | `10.0.0.0/16` |
| Subnet | `az-net-lab-subnet` | `10.0.1.0/24` |

#### Concepts Learned

- **Resource Group** — logical management and lifecycle container for Azure resources.
- **Tags** — metadata used to organize and manage resources.
- **Virtual Network (VNet)** — Azure's private networking boundary.
- **Address Space** — defines the IP range available to a VNet.
- **Subnet** — divides a VNet address space into smaller network segments.
- **CIDR** — notation used to define network ranges and their size.
- A subnet must fall within the address space of its VNet.

#### Azure CLI

Resources were created manually using Azure CLI and inspected using `az ... show` commands.

JMESPath queries were used with `--query` to extract specific properties from Azure resource objects.

Example:

```powershell
az network vnet show `
  --resource-group az-net-lab-rg `
  --name az-net-lab-vnet `
  --query "{Name:name, AddressSpace:addressSpace.addressPrefixes, Subnets:subnets[].{Name:name, Prefix:addressPrefix}}" `
  -o json
```

Result:

```json
{
  "AddressSpace": [
    "10.0.0.0/16"
  ],
  "Name": "az-net-lab-vnet",
  "Subnets": [
    {
      "Name": "az-net-lab-subnet",
      "Prefix": "10.0.1.0/24"
    }
  ]
}
```

#### Troubleshooting Exercise

An initial subnet creation attempt used:

```text
10.1.0.0/24
```

Azure rejected it with:

```text
NetcfgSubnetRangeOutsideVnet
```

The VNet uses:

```text
10.0.0.0/16
```

which covers:

```text
10.0.0.0 - 10.0.255.255
```

Therefore `10.1.0.0/24` is outside the VNet address space.

The subnet was subsequently created using:

```text
10.0.1.0/24
```

#### Current State

```text
az-net-lab-rg
└── az-net-lab-vnet
    │
    ├── Address Space: 10.0.0.0/16
    │
    └── az-net-lab-subnet
        └── Prefix: 10.0.1.0/24
```

### Next

- Network Security Groups (NSGs)
- Network security rules
- Associating an NSG with a subnet
```

**I would make one change before committing:** we should probably call the section **`01 — Azure Networking Foundation`** rather than simply documenting the current state. That gives us a natural structure for `02 — NSGs`, `03 — Routing`, etc. as the lab grows.

If you're happy with that structure, we'll put it into `README.md`, review the diff, and make our **first lab commit**.