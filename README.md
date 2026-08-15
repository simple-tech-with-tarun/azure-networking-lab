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