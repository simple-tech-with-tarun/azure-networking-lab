# Azure Route Server + BGP Lab

## Overview

This lab demonstrates how **Azure Route Server** exchanges routes dynamically with a network virtual appliance (NVA) using **BGP (Border Gateway Protocol)**.

The lab focuses on the core BGP lifecycle:

> **BGP Peering → Route Advertisement → Route Learning → Azure Route Installation → Route Withdrawal → Route Removal**

The NVA runs **FRRouting (FRR)** on Ubuntu and acts as the BGP peer.

The lab intentionally uses a synthetic route (`10.100.0.0/24`) to demonstrate BGP route advertisement and withdrawal without requiring a real network behind the NVA.

---

## Objectives

By completing this lab, the following concepts are demonstrated:

- Azure Route Server
- BGP Autonomous System Numbers (ASN)
- BGP peering
- FRRouting (FRR)
- NVA-to-Route Server BGP sessions
- BGP route advertisement
- Route Server learned routes
- Azure effective route tables
- BGP route withdrawal
- Dynamic route removal
- BGP control plane vs. data plane
- Persisting FRR configuration

---

## Architecture

```text
                         Azure VNet
                       10.0.0.0/16
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
 RouteServerSubnet       WorkloadSubnet       NvaSubnet
   10.0.0.0/24            10.0.1.0/24         10.0.2.0/24
          │                   │                   │
          │                   ▼                   ▼
          │              Workload VM          NVA / FRR
          │                10.0.1.4            10.0.2.4
          │                                      │
          │                                      │
          │              BGP Peering              │
          │                                      │
          ▼                                      │
   Azure Route Server ◄─────────────────────────┘
   AS 65515
   10.0.0.4
   10.0.0.5
```

### Route flow

The NVA advertises a route through BGP:

```text
NVA / FRR
10.0.2.4
AS 65001
      │
      │ BGP
      ▼
Azure Route Server
10.0.0.4 / 10.0.0.5
AS 65515
      │
      │ Azure routing system
      ▼
Workload NIC
10.0.1.4
```

---

## Resource Details

### Resource Group

```text
Name: az-route-server-bgp-lab-rg
Region: centralindia
```

### Virtual Network

```text
Name: az-route-server-bgp-vnet
Address space: 10.0.0.0/16
```

### Subnets

| Subnet | Address Prefix | Purpose |
|---|---|---|
| RouteServerSubnet | 10.0.0.0/24 | Dedicated subnet for Azure Route Server |
| WorkloadSubnet | 10.0.1.0/24 | Workload VM |
| NvaSubnet | 10.0.2.0/24 | NVA / FRR VM |

### Route Server

```text
Name: az-route-server-bgp-rs
ASN: 65515
BGP IP: 10.0.0.4
BGP IP: 10.0.0.5
```

Azure Route Server uses two BGP IP addresses for its managed route exchange service.

### NVA

```text
Name: az-route-server-bgp-nva-vm
Private IP: 10.0.2.4
ASN: 65001
OS: Ubuntu 22.04
Routing software: FRRouting (FRR)
VM size: Standard_D2s_v5
```

### Workload VM

```text
Name: az-route-server-bgp-workload-vm
Private IP: 10.0.1.4
OS: Ubuntu 22.04
VM size: Standard_D2s_v5
```

---

# Phase 1 — Build the Network

## Create the Resource Group

```powershell
az group create `
  --name "az-route-server-bgp-lab-rg" `
  --location "centralindia" `
  --tags Owner=Tarun
```

Azure automatically applied the `AutoDelete=Yes` tag according to the lab environment.

---

## Create the VNet

```powershell
az network vnet create `
  --resource-group "az-route-server-bgp-lab-rg" `
  --name "az-route-server-bgp-vnet" `
  --location "centralindia" `
  --address-prefixes "10.0.0.0/16"
```

---

## Create Route Server Subnet

Azure Route Server requires a dedicated subnet.

```powershell
az network vnet subnet create `
  --resource-group "az-route-server-bgp-lab-rg" `
  --vnet-name "az-route-server-bgp-vnet" `
  --name "RouteServerSubnet" `
  --address-prefixes "10.0.0.0/24"
```

---

## Create NVA Subnet

```powershell
az network vnet subnet create `
  --resource-group "az-route-server-bgp-lab-rg" `
  --vnet-name "az-route-server-bgp-vnet" `
  --name "NvaSubnet" `
  --address-prefixes "10.0.2.0/24"
```

---

## Create Workload Subnet

```powershell
az network vnet subnet create `
  --resource-group "az-route-server-bgp-lab-rg" `
  --vnet-name "az-route-server-bgp-vnet" `
  --name "WorkloadSubnet" `
  --address-prefixes "10.0.1.0/24"
```

---

# Phase 2 — Build the NVA

The NVA is an Ubuntu VM running FRRouting.

Its role is to act as an external BGP speaker from Azure Route Server's perspective.

## NVA Network Security Group

```powershell
az network nsg create `
  --resource-group "az-route-server-bgp-lab-rg" `
  --name "az-route-server-bgp-nva-nsg" `
  --location "centralindia"
```

During initial setup, SSH access was temporarily allowed for configuration.

After the NVA configuration was complete, the public IP and SSH access were removed so that the NVA was no longer exposed to the Internet.

---

## NVA NIC

The NVA NIC was configured with:

```text
Private IP: 10.0.2.4
IP forwarding: Enabled
```

IP forwarding is important because the NVA is intended to forward traffic rather than only act as an endpoint.

---

## NVA VM

```powershell
az vm create `
  --resource-group "az-route-server-bgp-lab-rg" `
  --name "az-route-server-bgp-nva-vm" `
  --location "centralindia" `
  --nics "az-route-server-bgp-nva-nic" `
  --image "Ubuntu2204" `
  --size "Standard_D2s_v5" `
  --admin-username "azureuser" `
  --generate-ssh-keys `
  --tags Owner=Tarun AutoDelete=Yes
```

---

# Phase 3 — Install FRRouting

FRRouting was installed on the NVA:

```bash
sudo apt update && sudo apt install -y frr
```

Verify the FRR service:

```bash
sudo systemctl status frr --no-pager
```

---

## Enable BGP

FRR's BGP daemon was initially disabled.

Check the configuration:

```bash
sudo grep -E '^(zebra|bgpd)' /etc/frr/daemons
```

Enable `bgpd`:

```bash
sudo sed -i 's/^bgpd=no/bgpd=yes/' /etc/frr/daemons
```

Restart FRR:

```bash
sudo systemctl restart frr
```

---

# Phase 4 — Create Azure Route Server

## Create Route Server Public IP

```powershell
az network public-ip create `
  --resource-group "az-route-server-bgp-lab-rg" `
  --name "az-route-server-bgp-rs-pip" `
  --location "centralindia" `
  --sku "Standard" `
  --allocation-method "Static" `
  --version "IPv4" `
  --tags Owner=Tarun AutoDelete=Yes
```

---

## Create Route Server

```powershell
az network routeserver create `
  --resource-group "az-route-server-bgp-lab-rg" `
  --name "az-route-server-bgp-rs" `
  --location "centralindia" `
  --hosted-subnet "/subscriptions/<subscription-id>/resourceGroups/az-route-server-bgp-lab-rg/providers/Microsoft.Network/virtualNetworks/az-route-server-bgp-vnet/subnets/RouteServerSubnet" `
  --public-ip-address "az-route-server-bgp-rs-pip" `
  --tags Owner=Tarun AutoDelete=Yes
```

The Route Server was provisioned with:

```text
ASN: 65515
BGP IPs:
  10.0.0.4
  10.0.0.5
```

Verify the ASN:

```powershell
az network routeserver show `
  --resource-group "az-route-server-bgp-lab-rg" `
  --name "az-route-server-bgp-rs" `
  --query "virtualRouterAsn" `
  --output tsv
```

Expected:

```text
65515
```

---

# Phase 5 — Create BGP Peering

The NVA uses ASN `65001`.

Azure Route Server uses ASN `65515`.

Create the Route Server peering:

```powershell
az network routeserver peering create `
  --resource-group "az-route-server-bgp-lab-rg" `
  --routeserver "az-route-server-bgp-rs" `
  --name "nva-bgp-peer" `
  --peer-ip "10.0.2.4" `
  --peer-asn "65001"
```

This tells Route Server:

```text
Peer IP: 10.0.2.4
Peer ASN: 65001
```

---

# Phase 6 — Configure FRR BGP

Enter FRR configuration mode:

```bash
sudo vtysh
```

Configure BGP:

```text
configure terminal

router bgp 65001
 neighbor 10.0.0.4 remote-as 65515
 neighbor 10.0.0.5 remote-as 65515

no bgp ebgp-requires-policy

end
```

The two neighbors correspond to the two Route Server BGP IP addresses.

---

## Why `no bgp ebgp-requires-policy`?

FRR enables the `ebgp-requires-policy` behavior by default in this configuration mode.

Without explicit inbound and outbound routing policies, the BGP sessions can reach an **Established** state while route exchange is still blocked.

The BGP summary initially showed:

```text
(Policy)
```

After disabling the requirement:

```text
no bgp ebgp-requires-policy
```

the sessions became usable for this simplified lab.

For production environments, explicit BGP route policies/filtering are preferable.

---

# Phase 7 — Verify BGP Sessions

Check the BGP summary:

```text
show ip bgp summary
```

Expected peers:

```text
10.0.0.4
10.0.0.5
```

Both sessions should be:

```text
Established
```

At this stage there are no advertised test prefixes yet.

---

# Phase 8 — Create a Test Route

To demonstrate route advertisement without building another network behind the NVA, a synthetic route was created.

The prefix used was:

```text
10.100.0.0/24
```

Create a static Null0 route:

```text
configure terminal

ip route 10.100.0.0/24 Null0

end
```

Verify:

```text
show ip route 10.100.0.0/24
```

The route appears as a static blackhole route.

This route is intentionally synthetic and exists only to demonstrate BGP control-plane behavior.

---

# Phase 9 — Advertise the Route with BGP

Add the route to the BGP configuration:

```text
configure terminal

router bgp 65001
 network 10.100.0.0/24

end
```

Verify the BGP table:

```text
show ip bgp
```

The expected route is:

```text
*> 10.100.0.0/24
```

The route is now locally originated by the NVA's BGP process.

---

# Phase 10 — Verify Route Server Learned Routes

Use the Azure Route Server learned-routes command:

```powershell
az network routeserver peering list-learned-routes `
  --resource-group "az-route-server-bgp-lab-rg" `
  --routeserver "az-route-server-bgp-rs" `
  --name "nva-bgp-peer" `
  --output json
```

The Route Server reported:

```text
Network: 10.100.0.0/24
Next hop: 10.0.2.4
Source peer: 10.0.2.4
AS path: 65001
```

The route was learned through both Route Server instances.

This proves:

```text
NVA → Route Server
```

BGP route advertisement was successful.

---

# Phase 11 — Verify Routes Advertised by Route Server

The Route Server also advertised the VNet route toward the NVA.

Command:

```powershell
az network routeserver peering list-advertised-routes `
  --resource-group "az-route-server-bgp-lab-rg" `
  --routeserver "az-route-server-bgp-rs" `
  --name "nva-bgp-peer" `
  --output json
```

The Route Server advertised:

```text
Network: 10.0.0.0/16
Next hop: 10.0.0.4 / 10.0.0.5
AS path: 65515
```

This demonstrates the reverse direction:

```text
Route Server → NVA
```

---

# Phase 12 — Create the Workload VM

## Workload NIC

```powershell
az network nic create `
  --resource-group "az-route-server-bgp-lab-rg" `
  --name "az-route-server-bgp-workload-nic" `
  --location "centralindia" `
  --subnet "/subscriptions/<subscription-id>/resourceGroups/az-route-server-bgp-lab-rg/providers/Microsoft.Network/virtualNetworks/az-route-server-bgp-vnet/subnets/WorkloadSubnet" `
  --tags Owner=Tarun AutoDelete=Yes
```

The workload NIC received:

```text
10.0.1.4
```

---

## Workload VM

```powershell
az vm create `
  --resource-group "az-route-server-bgp-lab-rg" `
  --name "az-route-server-bgp-workload-vm" `
  --location "centralindia" `
  --nics "az-route-server-bgp-workload-nic" `
  --image "Ubuntu2204" `
  --size "Standard_D2s_v5" `
  --admin-username "azureuser" `
  --generate-ssh-keys `
  --tags Owner=Tarun AutoDelete=Yes
```

The workload VM has no public IP.

---

# Phase 13 — Verify Azure Effective Route Table

The most important Azure-side verification was:

```powershell
az network nic show-effective-route-table `
  --resource-group "az-route-server-bgp-lab-rg" `
  --name "az-route-server-bgp-workload-nic" `
  --output json
```

The important entry was:

```text
10.100.0.0/24
nextHopIpAddress: 10.0.2.4
nextHopType: VirtualNetworkGateway
source: VirtualNetworkGateway
state: Active
```

This proves that the BGP-learned route was installed into Azure's effective routing information for the workload NIC.

Conceptually:

```text
10.0.1.4
  │
  │ destination = 10.100.0.0/24
  ▼
Azure routing
  │
  │ next hop = 10.0.2.4
  ▼
NVA / FRR
```

---

# Important Observation — Guest Route vs Azure Effective Route

The workload VM's Linux routing table was also checked:

```bash
ip route
```

The guest OS did not show:

```text
10.100.0.0/24
```

This does not mean the Azure route was missing.

Azure's effective route table represents routing information applied by the Azure networking platform, while the Linux guest maintains its own kernel routing table.

For this lab, the Azure effective route table is the authoritative verification for the Azure-side route installation.

---

# Phase 14 — Demonstrate BGP Route Withdrawal

The route was intentionally withdrawn from BGP.

On the NVA, the following configuration was removed:

```text
configure terminal

router bgp 65001
 no network 10.100.0.0/24

end
```

The underlying static Null0 route was not removed.

Only the **BGP advertisement** was removed.

---

## Verify FRR

```text
show ip bgp
```

FRR reported:

```text
No BGP prefixes displayed, 0 exist
```

This confirms that the test prefix was no longer being advertised through BGP.

---

# Phase 15 — Verify Route Server Withdrawal

Check Route Server learned routes:

```powershell
az network routeserver peering list-learned-routes `
  --resource-group "az-route-server-bgp-lab-rg" `
  --routeserver "az-route-server-bgp-rs" `
  --name "nva-bgp-peer" `
  --output json
```

The result was:

```json
{
  "RouteServiceRole_IN_0": [],
  "RouteServiceRole_IN_1": []
}
```

The Route Server no longer had the `10.100.0.0/24` route.

This proves that the BGP withdrawal propagated to Azure Route Server.

---

# Phase 16 — Verify Azure Route Removal

Finally, the workload NIC effective route table was checked specifically for the test prefix:

```powershell
az network nic show-effective-route-table `
  --resource-group "az-route-server-bgp-lab-rg" `
  --name "az-route-server-bgp-workload-nic" `
  --query "value[?addressPrefix[0]=='10.100.0.0/24']" `
  --output json
```

Result:

```json
[]
```

This confirms that Azure removed the previously installed BGP route.

---

# BGP Route Lifecycle Demonstrated

## Advertisement

```text
NVA / FRR
AS 65001
10.100.0.0/24
      │
      │ BGP UPDATE
      ▼
Azure Route Server
AS 65515
      │
      ▼
Azure routing system
      │
      ▼
Workload NIC
10.100.0.0/24
next hop → 10.0.2.4
```

## Withdrawal

```text
NVA / FRR
      │
      │ Withdraw 10.100.0.0/24
      ▼
Azure Route Server
      │
      │ Remove learned prefix
      ▼
Azure routing system
      │
      │ Remove effective route
      ▼
Workload NIC
10.100.0.0/24 → absent
```

---

# Control Plane vs Data Plane

One of the most important concepts from this lab is the difference between the **control plane** and **data plane**.

### Control Plane

BGP is the control plane.

It determines:

- Which prefixes exist
- Which peer advertised them
- Which ASN advertised them
- Which routes should be installed
- When a route should be withdrawn

In this lab:

```text
FRR → BGP → Route Server → Azure routing
```

### Data Plane

The data plane actually forwards packets.

The Route Server itself is primarily a **route exchange service**. It does not act as a general-purpose packet-forwarding NVA.

The NVA would perform packet forwarding when configured for that purpose.

---

# Key BGP Concepts Learned

## Autonomous System Number

An ASN identifies a BGP autonomous system.

This lab used:

```text
NVA:          65001
Route Server: 65515
```

---

## BGP Neighbor

A BGP neighbor is another BGP speaker with which a session is established.

The NVA had two neighbors:

```text
10.0.0.4
10.0.0.5
```

Both belong to Azure Route Server.

---

## Route Advertisement

The NVA advertised:

```text
10.100.0.0/24
```

using:

```text
network 10.100.0.0/24
```

---

## Route Withdrawal

Removing the network statement:

```text
no network 10.100.0.0/24
```

caused the prefix to be withdrawn.

Azure subsequently removed the route.

---

## Next Hop

Azure's effective route table showed:

```text
10.100.0.0/24 → 10.0.2.4
```

The NVA was therefore identified as the next hop for the learned prefix.

---

# Persisting FRR Configuration

After completing the BGP configuration, the live FRR configuration was persisted:

```bash
sudo vtysh -c 'write memory'
```

FRR confirmed:

```text
Integrated configuration saved to /etc/frr/frr.conf
```

The saved BGP configuration was verified:

```bash
sudo grep -A8 '^router bgp' /etc/frr/frr.conf
```

The resulting configuration included:

```text
router bgp 65001
 no bgp ebgp-requires-policy
 neighbor 10.0.0.4 remote-as 65515
 neighbor 10.0.0.5 remote-as 65515
exit
```

The test advertisement was intentionally absent because it had already been withdrawn.

---

# Security Considerations

The NVA was initially given temporary Internet-facing SSH access for configuration.

After the setup was complete:

- The NVA public IP was detached.
- The temporary SSH NSG rule was removed.
- The NVA remained accessible through Azure management tooling such as VM Run Command.
- The workload VM did not receive a public IP.

This is preferable to leaving administrative access exposed unnecessarily.

---

# Cleanup

To remove the complete lab:

```powershell
az group delete `
  --name "az-route-server-bgp-lab-rg" `
  --yes `
  --no-wait
```

This removes the resources contained in the resource group.

Before deleting the resource group, verify that it contains only resources belonging to this lab.

---

# Final Architecture Summary

```text
┌─────────────────────────────────────────────────────────┐
│                    Azure VNet                           │
│                    10.0.0.0/16                          │
│                                                         │
│  ┌──────────────────┐                                   │
│  │ Route Server     │                                   │
│  │ AS 65515         │                                   │
│  │ 10.0.0.4         │                                   │
│  │ 10.0.0.5         │                                   │
│  └────────┬─────────┘                                   │
│           │                                             │
│           │ BGP                                         │
│           │                                             │
│  ┌────────▼─────────┐        ┌──────────────────────┐   │
│  │ NVA / FRR        │        │ Workload VM          │   │
│  │ AS 65001         │        │ 10.0.1.4             │   │
│  │ 10.0.2.4         │        │                      │   │
│  └──────────────────┘        └──────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘

Test prefix:
10.100.0.0/24

Advertisement:
NVA → Route Server → Azure routing → Workload NIC

Withdrawal:
NVA → Route Server removes route → Azure removes route
```

---

# Conclusion

This lab demonstrated the fundamental role of Azure Route Server in dynamically exchanging routes with an NVA through BGP.

The most important practical takeaway is:

> **The NVA advertises routes, Route Server exchanges them, and Azure installs the resulting routes into the networking fabric.**

The route lifecycle was verified end-to-end:

```text
Advertise
   ↓
Learn
   ↓
Install
   ↓
Withdraw
   ↓
Remove
```

This provides a foundation for more advanced Azure networking scenarios involving:

- Multiple NVAs
- Dynamic route exchange
- Active/active network appliances
- Hub-and-spoke routing
- SD-WAN integration
- Hybrid connectivity
- BGP route filtering
- Route preference and failover