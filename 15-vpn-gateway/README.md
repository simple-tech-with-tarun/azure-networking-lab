# Azure VPN Connectivity Labs

Hands-on Azure networking labs demonstrating **Site-to-Site (S2S) VPN** and **VNet-to-VNet VPN** connectivity using Azure VPN Gateway.

These labs focus on understanding how Azure VPN Gateway establishes IPsec/IKE connectivity, how address spaces and traffic selectors work, and how Azure-Azure and Azure-external VPN architectures differ.

---

## Labs Covered

- [Site-to-Site VPN (S2S)](#1-site-to-site-vpn-s2s)
- [VNet-to-VNet VPN](#2-vnet-to-vnet-vpn)
- [Architecture Comparison](#3-architecture-comparison)
- [Key Lessons](#4-key-lessons)
- [Cleanup](#5-cleanup)

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

A second reverse connection was deliberately attempted to test the model.

Azure rejected it with:

```text
VirtualNetworkGatewayConnectionAlreadyExistsForEndpoints
```

This demonstrated that an S2S connection between a specific Azure VPN Gateway and Local Network Gateway is represented by **one connection resource**. A separate reverse connection resource is not required.

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
   │ 10.20.0.0/16            │
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
   │ 10.30.0.0/16            │
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
Source:      10.20.1.4
Destination: 10.30.1.4
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
Source:      10.30.1.4
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

# 3. Architecture Comparison

| Feature | S2S VPN | VNet-to-VNet VPN |
|---|---|---|
| Azure side | Azure VNet | Azure VNet |
| Remote side | External network | Azure VNet |
| Azure resource representing remote side | Local Network Gateway | Second VPN Gateway |
| VPN Gateway | Required | Required on both sides |
| IPsec/IKE | Yes | Yes |
| Typical use | Azure ↔ on-prem / AWS / GCP | Azure VNet ↔ Azure VNet |
| Separate reverse connection | Not required | Required by this Azure connection model |
| Remote VPN device | Required | Not applicable |
| strongSwan in lab | Yes | No |
| Traffic direction | Bidirectional | Bidirectional |

---

# 4. Key Lessons

## 1. Non-overlapping address spaces are fundamental

The networks must have distinct address ranges.

Example:

```text
Azure VNet:       10.20.0.0/16
Remote VNet:      10.40.0.0/16
```

Overlapping address spaces create routing ambiguity and prevent normal VPN connectivity.

---

## 2. S2S and VNet-to-VNet are architecturally different

S2S represents:

```text
Azure ↔ External network
```

VNet-to-VNet represents:

```text
Azure VNet ↔ Azure VNet
```

Even though both use IPsec/IKE and VPN Gateway, they solve different connectivity problems.

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

Creating the Azure VPN Gateway and connection does not establish a tunnel by itself.

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

## 6. Workload testing is more important than relying on one telemetry field

VPN connection counters can appear asymmetric or ambiguous.

In the VNet-to-VNet lab, one connection resource showed:

```text
0 / 0
```

while the other showed:

```text
336 / 336
```

However, actual workload traffic succeeded in **both directions**.

Therefore, end-to-end private IP connectivity is the stronger validation of functional routing and VPN connectivity.

---

## 7. VNet peering is generally the simpler Azure-Azure choice

For ordinary Azure-to-Azure VNet connectivity, **VNet peering** is generally the native/direct option.

VNet-to-VNet VPN is useful when the architecture specifically requires VPN-based connectivity or when demonstrating VPN Gateway/IPsec behavior.

---

# 5. Cleanup

Both labs used dedicated resource groups with automatic-cleanup tags:

```text
Owner=Tarun
AutoDelete=Yes
```

The completed lab environments were deleted after validation to avoid unnecessary Azure costs.

---

# Final Validation Summary

### S2S VPN

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

### VNet-to-VNet VPN

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

These labs provide a practical foundation for understanding **Azure VPN Gateway, IKE/IPsec, Local Network Gateway, strongSwan, traffic selectors, routing, and Azure-to-Azure versus Azure-to-external VPN architectures**.