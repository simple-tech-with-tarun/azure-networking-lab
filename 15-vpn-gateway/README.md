# Azure VPN Connectivity Labs

Hands-on Azure networking labs demonstrating **Site-to-Site (S2S) VPN**, **VNet-to-VNet VPN**, and **Point-to-Site (P2S) VPN** connectivity using Azure VPN Gateway.

These labs focus on understanding how Azure VPN Gateway establishes IPsec/IKE and OpenVPN connectivity, how address spaces and traffic selectors affect routing, how Azure and external networks differ, and how private client access to Azure resources works through VPN connectivity.

---

## Labs Covered

- [Site-to-Site VPN (S2S)](#1-site-to-site-vpn-s2s)
- [VNet-to-VNet VPN](#2-vnet-to-vnet-vpn)
- [Point-to-Site VPN (P2S)](#3-point-to-site-vpn-p2s)
- [Architecture Comparison](#4-architecture-comparison)
- [Key Lessons](#5-key-lessons)
- [Cleanup](#6-cleanup)
- [Final Validation Summary](#final-validation-summary)

---

# 1. Site-to-Site VPN (S2S)

## Objective

Build a Site-to-Site IPsec VPN between:

- An Azure VNet
- A simulated external/on-premises network

The external network is simulated inside Azure using a Linux VM running **strongSwan** as the VPN appliance.

> The simulated on-premises VNet physically exists in Azure, but logically represents an external network. This allows the S2S architecture to be demonstrated without requiring a physical on-premises VPN device.

---

## Architecture

```text
                 Azure
        ┌──────────────────────┐
        │                      │
        │  Azure VNet          │
        │  10.20.0.0/16       │
        │                      │
        │  Workload            │
        │  10.20.1.0/24       │
        │       │              │
        │       │              │
        │  Azure VPN Gateway   │
        │  20.204.251.207     │
        └──────────┬───────────┘
                   │
             IPsec / IKEv2
                   │
                   │
        ┌──────────┴───────────┐
        │  Simulated On-Prem   │
        │                      │
        │  VNet                │
        │  10.40.0.0/16       │
        │                      │
        │  Workload            │
        │  10.40.1.0/24       │
        │                      │
        │  strongSwan VM       │
        │  10.40.254.4         │
        │  Public IP           │
        │  20.198.90.95        │
        └──────────────────────┘
```

---

## Azure Configuration

### Azure VNet

```text
VNet:             az-vpn-s2s-vnet-azure
Address space:    10.20.0.0/16

Workload subnet:  10.20.1.0/24
GatewaySubnet:    10.20.254.0/27
```

### Simulated On-Premises VNet

```text
VNet:             az-vpn-s2s-vnet-onprem
Address space:    10.40.0.0/16

Workload subnet:  10.40.1.0/24
VPN subnet:       10.40.254.0/24
```

### Azure VPN Gateway

```text
Name:             az-vpn-s2s-azure-vng
SKU:              VpnGw1AZ
Type:             VPN
VPN type:         RouteBased
Generation:       Gen1
BGP:              Disabled
Active-active:    Disabled
Public IP:        20.204.251.207
```

### Local Network Gateway

```text
Name:             az-vpn-s2s-onprem-lng
Gateway IP:       20.198.90.95
Address prefix:   10.40.0.0/16
```

### VPN Connection

```text
Name:             az-vpn-s2s-azure-to-onprem
Connection type:  IPsec
Protocol:         IKEv2
```

A second reverse connection was deliberately attempted to test the connection model.

Azure rejected it with:

```text
VirtualNetworkGatewayConnectionAlreadyExistsForEndpoints
```

This demonstrated that an S2S connection between a specific Azure VPN Gateway and Local Network Gateway is represented by **one connection resource**. A separate reverse connection is not required.

---

## strongSwan Configuration

The simulated on-premises VPN appliance used Ubuntu with strongSwan 5.9.5.

### IP forwarding

IP forwarding was enabled:

```text
net.ipv4.ip_forward = 1
```

and persisted through:

```text
/etc/sysctl.d/99-vpn-forwarding.conf
```

### IKE/IPsec Configuration

```text
config setup
    uniqueids=no

conn azure-s2s
    type=tunnel
    keyexchange=ikev2
    authby=psk

    left=10.40.254.4
    leftid=20.198.90.95

    right=20.204.251.207
    rightid=20.204.251.207

    leftsubnet=10.40.0.0/16
    rightsubnet=10.20.0.0/16

    ike=aes256-sha1-modp1024,aes256-sha256-modp1024,aes256-sha1-modp2048,aes256-sha256-modp2048
    esp=aes256-sha256

    dpdaction=restart
    dpddelay=45s
    dpdtimeout=120s

    keyingtries=%forever
    auto=add
```

The pre-shared key was stored in:

```text
/etc/ipsec.secrets
```

with permissions:

```text
-rw------- 1 root root
```

---

## S2S Validation

The tunnel successfully established with:

```text
IKEv2
CHILD_SA installed
Traffic selectors:
10.40.0.0/16 === 10.20.0.0/16
```

Azure reported:

```text
Connection status: Connected
```

The strongSwan side reported an established IKE/IPsec SA.

### Traffic Test

Azure workload:

```text
10.20.1.4
```

successfully reached the simulated on-premises VPN appliance:

```text
10.40.254.4
```

Result:

```text
4 packets transmitted
4 packets received
0% packet loss
```

Azure VPN Gateway counters increased after the traffic test, confirming encrypted VPN traffic was being processed.

> The original S2S lab explicitly validated Azure → simulated on-premises traffic. Reverse-direction traffic was not experimentally validated during that run.

---

# 2. VNet-to-VNet VPN

## Objective

Build an IPsec VPN connection between **two Azure VNets** using Azure VPN Gateway.

Unlike S2S, both sides are Azure VNets.

---

## Architecture

```text
                    Azure
        ┌─────────────────────────┐
        │                         │
        │ VNet 1                  │
        │ 10.20.0.0/16           │
        │                         │
        │ Workload                │
        │ 10.20.1.4              │
        │                         │
        │ VPN Gateway             │
        │                         │
        └───────────┬─────────────┘
                    │
                IPsec / IKEv2
                    │
        ┌───────────┴─────────────┐
        │                         │
        │ VNet 2                  │
        │ 10.30.0.0/16           │
        │                         │
        │ Workload                │
        │ 10.30.1.4              │
        │                         │
        │ VPN Gateway             │
        │                         │
        └─────────────────────────┘
                    Azure
```

---

## VNet Configuration

### VNet 1

```text
VNet:             az-vpn-vnet1
Address space:    10.20.0.0/16

Workload subnet:  10.20.1.0/24
GatewaySubnet:    10.20.255.0/27
```

### VNet 2

```text
VNet:             az-vpn-vnet2
Address space:    10.30.0.0/16

Workload subnet:  10.30.1.0/24
GatewaySubnet:    10.30.255.0/27
```

The address spaces intentionally do not overlap.

---

## VPN Gateways

Both VNets used:

```text
SKU:              VpnGw1AZ
VPN type:         RouteBased
Generation:       Gen1
BGP:              Disabled
Active-active:    Disabled
```

---

## Connection Resources

The lab deliberately tested the VNet-to-VNet connection model.

Initial connection:

```text
az-vpn-vnet1-to-vnet2
```

A reverse-perspective connection was then created:

```text
az-vpn-vnet2-to-vnet1
```

After the second resource was created, both connection resources reported:

```text
Connected
```

Deleting the reverse connection caused the remaining connection to become:

```text
NotConnected
```

Recreating the reverse connection restored:

```text
Connected
```

### Important Observation

The two connection resources should **not** be interpreted as two independent VPN tunnels.

They represent the two gateway perspectives required by Azure's VNet-to-VNet connection model while establishing the underlying bidirectional IKE/IPsec security association.

---

# VNet-to-VNet Validation

Two workload VMs were deployed:

```text
VNet 1 workload:
10.20.1.4

VNet 2 workload:
10.30.1.4
```

Both VMs were private-only and had no public IP addresses.

---

## VNet 1 → VNet 2

```text
Source:       10.20.1.4
Destination:  10.30.1.4
```

Result:

```text
4 packets transmitted
4 packets received
0% packet loss
Average latency: 3.41 ms
```

---

## VNet 2 → VNet 1

```text
Source:       10.30.1.4
Destination: 10.20.1.4
```

Result:

```text
4 packets transmitted
4 packets received
0% packet loss
Average latency: 9.65 ms
```

Therefore, **bidirectional workload connectivity was experimentally proven**.

---

# 3. Point-to-Site VPN (P2S)

## Objective

Build a Point-to-Site VPN that allows an external client to securely access private resources inside an Azure VNet.

The lab uses:

- Azure VPN Gateway
- Microsoft Entra ID authentication
- OpenVPN
- Azure VPN Client
- A private Linux VM with no public IP

This demonstrates the difference between **client-to-Azure private connectivity** and traditional site-to-site connectivity.

---

## Architecture

```text
                     Internet
                        │
                        │
                 Azure VPN Gateway
                 4.224.238.41
                        │
                  OpenVPN / P2S
                        │
                        │
              Client address pool
                172.16.100.0/24
                        │
                        │
                 VPN Client
                 172.16.100.2
                        │
                        │
              Azure VNet
              10.50.0.0/16
                        │
                        │
              workload-subnet
               10.50.1.0/24
                        │
                        ▼
              ┌─────────────────┐
              │ Private VM      │
              │                 │
              │ 10.50.1.4      │
              │ Ubuntu          │
              │ Standard_D2s_v5 │
              │                 │
              │ No Public IP    │
              └─────────────────┘
```

---

## Azure VNet Configuration

```text
VNet:             az-vpn-p2s-vnet
Address space:    10.50.0.0/16

Workload subnet:  10.50.1.0/24
GatewaySubnet:    10.50.254.0/27
```

The P2S client address pool was intentionally kept separate from the Azure VNet:

```text
P2S client pool:
172.16.100.0/24
```

This prevents address-space overlap between VPN clients and Azure resources.

---

## VPN Gateway

```text
Name:             az-vpn-p2s-vng
SKU:              VpnGw1AZ
Gateway type:     VPN
VPN type:         RouteBased
Generation:       Generation1
Active-active:    Disabled
BGP:              Disabled
Public IP:        4.224.238.41
```

---

## P2S Authentication

The VPN used Microsoft Entra ID authentication.

```text
Authentication:   Microsoft Entra ID
Protocol:         OpenVPN
```

The configured Entra tenant was:

```text
Tenant ID:
b6cf81b9-5503-4f76-ad37-3ea335f4d099
```

Tenant endpoint:

```text
https://login.microsoftonline.com/b6cf81b9-5503-4f76-ad37-3ea335f4d099/
```

Issuer:

```text
https://sts.windows.net/b6cf81b9-5503-4f76-ad37-3ea335f4d099/
```

Microsoft-registered Azure VPN Client audience:

```text
c632b3df-fb67-4d84-bdcf-b95ad541b5c8
```

The P2S gateway configuration successfully reached:

```text
Provisioning state: Succeeded
```

> No client private keys, passwords, or other secrets are included in this repository.

---

## VPN Client Configuration

The Azure VPN Client configuration was generated from the VPN Gateway:

```text
az network vnet-gateway vpn-client generate
```

The generated configuration contained:

```text
AzureVPN/
└── azurevpnconfig.xml
```

The `azurevpnconfig.xml` profile was imported into **Azure VPN Client** on Windows.

The client successfully authenticated through Microsoft Entra ID and established the P2S connection.

---

## P2S Client Address

After establishing the VPN connection, Windows reported:

```text
PPP adapter az-vpn-p2s-vnet:

IPv4 Address:
172.16.100.2

Subnet Mask:
255.255.255.255
```

The `/32` address is expected for the point-to-point VPN interface.

---

## Routing Validation

Windows installed a route for the Azure VNet:

```text
10.50.0.0
255.255.0.0
On-link
172.16.100.2
```

This demonstrates that traffic destined for:

```text
10.50.0.0/16
```

is routed through the P2S VPN interface rather than through the normal Wi-Fi interface.

Internet traffic continued to use the normal Wi-Fi gateway.

This demonstrated **split tunneling**:

```text
Azure VNet traffic
        │
        ▼
P2S VPN interface
172.16.100.2
        │
        ▼
Azure VNet

Internet traffic
        │
        ▼
Normal Wi-Fi gateway
192.168.8.1
```

---

# Private VM

A private Linux VM was deployed inside the workload subnet.

```text
VM:
az-vpn-p2s-test-vm

OS:
Ubuntu 22.04

Size:
Standard_D2s_v5

Private IP:
10.50.1.4

Public IP:
None
```

The VM's NIC was created separately:

```text
NIC:
az-vpn-p2s-test-vm-nic
```

The NIC received:

```text
Private IP:
10.50.1.4
```

The NIC was attached to:

```text
VNet:
az-vpn-p2s-vnet

Subnet:
workload-subnet
```

---

## Network Security Group

A dedicated NSG was created:

```text
az-vpn-p2s-test-vm-nsg
```

SSH access was explicitly restricted to the P2S client address pool:

```text
Source:
172.16.100.0/24

Destination:
10.50.1.4

Protocol:
TCP

Destination port:
22

Priority:
100
```

The rule was:

```text
Allow-SSH-From-P2S
```

This means the VM does not expose SSH directly to the Internet.

---

# P2S Connectivity Validation

## TCP Connectivity

With the VPN connected, Windows successfully reached the private VM:

```text
Test-NetConnection 10.50.1.4 -Port 22
```

Result:

```text
ComputerName     : 10.50.1.4
RemoteAddress    : 10.50.1.4
RemotePort       : 22
InterfaceAlias   : az-vpn-p2s-vnet
SourceAddress    : 172.16.100.2
TcpTestSucceeded : True
```

This confirmed that traffic was actually traversing the P2S VPN interface.

---

## SSH Validation

The VM was accessed directly through its private IP:

```text
ssh azureuser@10.50.1.4
```

An interactive SSH session was successfully established.

The VM reported:

```text
hostname:
az-vpn-p2s-test-vm
```

Its network interface showed:

```text
eth0:
10.50.1.4/24
```

The VM's default route was:

```text
default via 10.50.1.1
```

This confirmed that the VM itself remained a normal private Azure workload and did not require a public IP for inbound VPN access.

---

# P2S Outbound Connectivity

From the private VM:

```text
curl -4 https://api.ipify.org
```

returned:

```text
40.80.81.215
```

A second external service produced the same outbound public address:

```text
curl -4 https://ifconfig.me
```

Result:

```text
40.80.81.215
```

This demonstrates that:

- The VM has private inbound connectivity through the P2S VPN.
- The VM does not require a public IP.
- Its outbound Internet traffic uses Azure's separate egress path.

Therefore, **private inbound access and public outbound access are independent concepts**.

---

# P2S VPN ON/OFF Test

The strongest validation was performed by testing the same private destination with the VPN connected and disconnected.

## VPN Disconnected

```text
Test-NetConnection 10.50.1.4 -Port 22
```

Result:

```text
InterfaceAlias   : Wi-Fi
SourceAddress    : 192.168.8.4
RemoteAddress    : 10.50.1.4
PingSucceeded    : False
TcpTestSucceeded : False
```

The client could not reach the private Azure VM through the normal Wi-Fi interface.

---

## VPN Connected

The VPN was then reconnected and the exact same test was repeated:

```text
Test-NetConnection 10.50.1.4 -Port 22
```

Result:

```text
InterfaceAlias   : az-vpn-p2s-vnet
SourceAddress    : 172.16.100.2
RemoteAddress    : 10.50.1.4
TcpTestSucceeded : True
```

This experimentally demonstrated:

```text
VPN OFF

192.168.8.4
     │
     │ Wi-Fi
     X
     │
10.50.1.4


VPN ON

172.16.100.2
     │
     │ P2S OpenVPN
     ▼
Azure VPN Gateway
     │
     │ VNet routing
     ▼
10.50.1.4
```

The private VM was therefore reachable specifically through the P2S VPN path.

---

# 4. Architecture Comparison

| Feature | S2S VPN | VNet-to-VNet VPN | P2S VPN |
|---|---|---|---|
| Azure side | Azure VNet | Azure VNet | Azure VNet |
| Remote side | External network | Azure VNet | Individual client |
| Primary use | Azure ↔ on-prem / external network | Azure VNet ↔ Azure VNet | Remote client ↔ Azure |
| VPN Gateway | Required | Required on both sides | Required |
| IPsec/IKE | Yes | Yes | No |
| OpenVPN | No | No | Yes |
| Microsoft Entra ID | No | No | Yes |
| Local Network Gateway | Required | Not required | Not required |
| Remote VPN device | Required | Not applicable | Not required |
| Client VPN software | No | No | Yes |
| strongSwan in lab | Yes | No | No |
| Client address pool | No | No | `172.16.100.0/24` |
| Private workload access | Yes | Yes | Yes |
| Typical authentication | PSK / certificate | PSK / certificate | Entra ID / certificate |
| Connection direction | Site ↔ Site | VNet ↔ VNet | Client ↔ VNet |

---

# 5. Key Lessons

## 1. Non-overlapping address spaces are fundamental

The networks must have distinct address ranges.

Example:

```text
Azure VNet:       10.20.0.0/16
Remote network:   10.40.0.0/16
```

Overlapping address spaces create routing ambiguity and prevent normal VPN connectivity.

The same principle applies to P2S client pools:

```text
Azure VNet:
10.50.0.0/16

P2S client pool:
172.16.100.0/24
```

The client pool must not overlap with the Azure VNet or relevant connected networks.

---

## 2. S2S, VNet-to-VNet, and P2S solve different problems

S2S represents:

```text
Azure ↔ External network
```

VNet-to-VNet represents:

```text
Azure VNet ↔ Azure VNet
```

P2S represents:

```text
Individual client ↔ Azure VNet
```

All three provide private connectivity, but their architectural models and configuration requirements are different.

---

## 3. A Local Network Gateway represents the remote VPN endpoint

In S2S, Azure does not manage the remote VPN appliance.

The Local Network Gateway tells Azure:

```text
Remote VPN endpoint:
20.198.90.95

Remote networks:
10.40.0.0/16
```

The actual remote VPN device—in this lab, strongSwan—must independently configure the other side of the tunnel.

---

## 4. S2S requires configuration on both sides

Creating the Azure VPN Gateway and connection does not establish a functional tunnel by itself.

The remote VPN device must also have matching:

- IKE parameters
- IPsec parameters
- Authentication
- Pre-shared key
- Local/remote networks
- Traffic selectors

The tunnel becomes operational when both sides successfully negotiate.

---

## 5. VNet-to-VNet does not mean two independent tunnels

The experiment demonstrated that Azure's VNet-to-VNet connection model uses connection resources representing both gateway perspectives.

This should not be confused with two independent IPsec tunnels.

The actual security association remains a bidirectional VPN relationship.

---

## 6. P2S authentication can use Microsoft Entra ID

The P2S lab demonstrated authentication using:

```text
Microsoft Entra ID
        +
OpenVPN
        +
Azure VPN Client
```

This provides a fundamentally different client-authentication model from the pre-shared-key approach used in the S2S lab.

---

## 7. A VPN connection being "Connected" is not enough

A VPN gateway reporting:

```text
Connected
```

does not by itself prove that an application can reach its destination.

The P2S lab went further:

```text
VPN connected
      ↓
Route installed
      ↓
Private IP reachable
      ↓
TCP/22 reachable
      ↓
SSH session established
```

End-to-end workload testing provides much stronger evidence of functional connectivity.

---

## 8. Routing determines which interface carries the traffic

During the P2S test:

```text
VPN ON:
Source = 172.16.100.2
Interface = az-vpn-p2s-vnet
```

While with the VPN disconnected:

```text
VPN OFF:
Source = 192.168.8.4
Interface = Wi-Fi
```

The same destination behaved differently because the available routing path changed.

This demonstrates why **VPN connectivity and routing are inseparable concepts**.

---

## 9. Private inbound access does not require a public IP

The P2S VM had:

```text
Private IP:
10.50.1.4

Public IP:
None
```

Yet it was successfully accessed over SSH from the external Windows client.

The VPN terminated at the Azure VPN Gateway, and Azure then routed the traffic privately to the VM.

This is a key distinction:

```text
Public IP on VM
≠
Required for remote access
```

Private connectivity mechanisms can provide remote access without exposing the workload directly to the Internet.

---

## 10. NSGs remain part of the connectivity path

The P2S VM was protected by:

```text
az-vpn-p2s-test-vm-nsg
```

with SSH explicitly permitted only from:

```text
172.16.100.0/24
```

This demonstrates that establishing a VPN does not automatically bypass Azure network security controls.

VPN connectivity provides the path.

The NSG still determines whether the traffic is allowed.

---

## 11. VNet peering is generally the simpler Azure-Azure choice

For ordinary Azure-to-Azure VNet connectivity, **VNet peering** is generally the native/direct option.

VNet-to-VNet VPN is useful when the architecture specifically requires VPN-based connectivity or when demonstrating VPN Gateway/IPsec behavior.

---

# 6. Cleanup

All lab resources should be removed after validation to avoid unnecessary Azure costs.

The labs use dedicated resource groups and the following tags:

```text
Owner=Tarun
AutoDelete=Yes
```

Before deleting resources, verify the resource group and resources:

```powershell
az resource list `
  --resource-group "az-vpn-p2s-lab-rg" `
  --output table
```

For the P2S lab, the resource group contains resources including:

```text
az-vpn-p2s-vnet
az-vpn-p2s-vng
az-vpn-p2s-gateway-pip
az-vpn-p2s-test-vm
az-vpn-p2s-test-vm-nic
az-vpn-p2s-test-vm-nsg
```

After completing validation, the dedicated resource group can be removed with:

```powershell
az group delete `
  --name "az-vpn-p2s-lab-rg" `
  --yes `
  --no-wait
```

> Do not execute the cleanup command until all required screenshots, outputs, notes, and validation evidence have been captured.

---

# Final Validation Summary

## S2S VPN

```text
Azure VNet                    10.20.0.0/16
        │
        │ IKEv2 / IPsec
        │
Simulated external network    10.40.0.0/16

Tunnel: Connected
Traffic: Successful
Packet loss: 0%
```

---

## VNet-to-VNet VPN

```text
Azure VNet 1                  10.20.0.0/16
        │
        │ IKEv2 / IPsec
        │
Azure VNet 2                  10.30.0.0/16

Tunnel: Connected
VNet 1 → VNet 2: 4/4
VNet 2 → VNet 1: 4/4
Packet loss: 0% both directions
```

---

## P2S VPN

```text
Windows Client
192.168.8.4
      │
      │ Internet
      ▼
Azure VPN Gateway
4.224.238.41
      │
      │ OpenVPN / Entra ID
      ▼
P2S Client
172.16.100.2
      │
      │ VNet route
      ▼
Azure VNet
10.50.0.0/16
      │
      ▼
Private VM
10.50.1.4
```

Validation:

```text
VPN OFF:
192.168.8.4 → 10.50.1.4:22
TCP: Failed

VPN ON:
172.16.100.2 → 10.50.1.4:22
TCP: Succeeded

SSH:
Successful
```

---

These labs provide a practical foundation for understanding:

- **Azure VPN Gateway**
- **Site-to-Site VPN**
- **VNet-to-VNet VPN**
- **Point-to-Site VPN**
- **IKE/IPsec**
- **OpenVPN**
- **Microsoft Entra ID authentication**
- **Local Network Gateway**
- **strongSwan**
- **Traffic selectors**
- **VPN client address pools**
- **Azure routing**
- **NSGs**
- **Private-only workloads**
- **Split tunneling**
- **Azure-to-Azure versus Azure-to-external VPN architectures**

The experiments emphasize an important networking principle:

> **A tunnel being established is only the beginning. Real connectivity is proven when the intended traffic reaches the intended workload through the intended path.**