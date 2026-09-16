# Azure Virtual WAN Lab

## Overview

This lab explores **Azure Virtual WAN** by building a small hub-based networking environment and establishing a **Site-to-Site IPsec VPN connection** between an Azure-based branch network and Azure Virtual WAN.

The lab focuses on understanding how Virtual WAN components fit together and how traffic can securely travel between a branch network and Azure VNets through a Virtual Hub.

---

## Architecture

```text
                    Azure Virtual WAN
                           |
                    +--------------+
                    | Virtual Hub  |
                    | 10.0.0.0/16  |
                    +------+-------+
                           |
              +------------+------------+
              |                         |
       VNet Connection             VNet Connection
              |                         |
      +-------+-------+         +-------+-------+
      |    VNet A     |         |    VNet B     |
      | 10.1.0.0/16  |         | 10.2.0.0/16  |
      +-------+-------+         +---------------+
              |
        Azure VM
       10.1.1.4


             Site-to-Site IPsec VPN
                       |
                       |
              +--------+--------+
              |   VPN Gateway  |
              |  Virtual Hub   |
              +--------+--------+
                       |
                 Public Internet
                       |
                +------+------+
                | Branch VM   |
                | strongSwan  |
                | 10.50.1.4   |
                | 10.50.2.4   |
                +------+------+
                       |
                 Branch LAN
                 10.50.2.0/24
```

---

## What is Azure Virtual WAN?

**Azure Virtual WAN** is a Microsoft-managed networking service designed to connect Azure VNets, branches, remote users, and other networks through a centralized Microsoft-managed network architecture.

A Virtual WAN environment can contain one or more **Virtual Hubs**.

The Virtual Hub provides the central networking point through which connected VNets and branch connectivity can be integrated.

---

## Core Components

### Virtual WAN

The main Virtual WAN resource for the lab:

```text
az-vwan-lab
```

### Virtual Hub

The Virtual Hub created for the lab:

```text
az-vwan-lab-hub
Address space: 10.0.0.0/16
```

The Virtual Hub provides the managed networking infrastructure for the connected VNets and VPN gateway.

### VNet Connections

Two Azure VNets were connected to the Virtual Hub:

```text
VNet A
10.1.0.0/16

VNet B
10.2.0.0/16
```

The workload VM was deployed in VNet A.

### VPN Gateway

A managed VPN Gateway was deployed inside the Virtual Hub to provide branch VPN connectivity.

### VPN Site

A VPN Site was created to represent the branch network:

```text
Branch network: 10.50.0.0/16
Branch public IP: <branch-public-ip>
```

The VPN Site represents the external/branch location from the Virtual WAN perspective.

### VPN Site Link

The branch VPN Site contains a VPN Site Link representing the branch's VPN endpoint.

### VPN Connection

A VPN Connection was created between the Virtual Hub VPN Gateway and the branch VPN Site.

The connection uses:

```text
IKEv2
Pre-shared key authentication
IPsec
```

---

## Lab Environment

### Azure VNet A

```text
VNet:      az-vwan-lab-vnet-a
CIDR:      10.1.0.0/16
Subnet:    WorkloadSubnet
Subnet:    10.1.1.0/24
VM:        az-vwan-lab-vm-a
Private IP: 10.1.1.4
```

### Azure VNet B

```text
VNet:      az-vwan-lab-vnet-b
CIDR:      10.2.0.0/16
Subnet:    WorkloadSubnet
Subnet:    10.2.1.0/24
```

VNet B was connected to the Virtual Hub but did not contain a workload VM in this phase.

### Branch Network

```text
VNet:       az-vwan-lab-branch-vnet
CIDR:       10.50.0.0/16

WANSubnet:  10.50.1.0/24
LANSubnet:  10.50.2.0/24
```

The branch VM used two NICs:

```text
WAN NIC
10.50.1.4

LAN NIC
10.50.2.4
```

IP forwarding was enabled on both NICs.

---

## Site-to-Site IPsec VPN

The branch VM was used as a simulated on-premises VPN device.

Instead of using a physical VPN appliance, **strongSwan** was installed on the branch VM.

```text
Branch VM
     |
 strongSwan
     |
 IKEv2 / IPsec
     |
 Virtual WAN VPN Gateway
     |
 Virtual Hub
```

The VPN tunnel was configured using:

- IKEv2
- Pre-shared key authentication
- AES/SHA based IKE parameters
- AES-GCM IPsec encryption
- Dead Peer Detection
- Automatic tunnel restart

---

## strongSwan

strongSwan was installed on the branch VM and configured with a local connection profile:

```text
azure-vwan
```

The configuration established an encrypted tunnel between:

```text
Branch
10.50.0.0/16
        |
        | IPsec
        |
Azure
10.1.0.0/16
```

The final tunnel state showed:

```text
IKE_SA: ESTABLISHED
CHILD_SA: INSTALLED
```

This confirmed that both the IKE security association and the IPsec child security association were successfully established.

---

## Traffic Validation

After establishing the tunnel, connectivity from the branch VM to the Azure workload VM was tested.

The test connected to:

```text
10.1.1.4:22
```

The connection succeeded.

IPsec traffic counters also increased, confirming that traffic was actually passing through the encrypted tunnel rather than merely having an established IKE session.

Therefore, the lab successfully demonstrated:

```text
Branch VM
   |
   | Encrypted IPsec tunnel
   |
Virtual WAN VPN Gateway
   |
Virtual Hub
   |
VNet Connection
   |
VNet A
   |
Azure VM
```

---

## What I Learned

- Azure Virtual WAN provides a managed networking architecture for connecting Azure and external networks.
- A **Virtual Hub** is the central networking component inside a Virtual WAN.
- Azure VNets connect to the Virtual Hub through **VNet Connections**.
- A **VPN Site** represents an external/branch network from the Virtual WAN perspective.
- A **VPN Site Link** represents the VPN endpoint/link of that site.
- The Virtual Hub's managed VPN Gateway establishes VPN connectivity with the branch.
- strongSwan can be used to simulate an on-premises VPN appliance.
- An IKE session being established does not necessarily mean application traffic is working.
- IPsec traffic counters are useful for confirming that encrypted traffic is actually flowing.
- A successful TCP connection across the tunnel provides practical end-to-end validation.

---

## Next Phase

The next phase of this networking lab will extend the existing architecture with **BGP**.

The goal will be to understand:

```text
Branch
   |
 BGP
   |
IPsec VPN
   |
Virtual WAN
   |
Virtual Hub
   |
Azure VNets
```

The current lab intentionally stops before BGP so that the basic Virtual WAN and Site-to-Site IPsec architecture is understood first.

---

> This README documents the completed first phase of the Azure Virtual WAN lab. BGP configuration will be documented separately when the lab is recreated and extended.