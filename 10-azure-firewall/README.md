# Azure Firewall + UDR Traffic Inspection

This lab demonstrates how to use **Azure Firewall with User Defined Routes (UDRs)** to inspect and control private traffic between a hub VNet and a spoke VNet.

The lab uses a small hub-and-spoke architecture where traffic from a private workload in the spoke is forced through Azure Firewall before reaching a private workload in the hub.

---

## Architecture

```text
                           Hub VNet
                        10.70.0.0/16
                              │
                     AzureFirewallSubnet
                        10.70.1.0/26
                              │
                     ┌────────▼────────┐
                     │  Azure Firewall  │
                     │   10.70.1.4     │
                     │    Standard     │
                     └────────┬────────┘
                              │
                     hub-workload-subnet
                        10.70.10.0/24
                              │
                     ┌────────▼────────┐
                     │    Hub Server   │
                     │   10.70.10.4    │
                     │    TCP :8080    │
                     └─────────────────┘
                              ▲
                              │
                         VNet Peering
                              │
                              ▼
                     ┌─────────────────┐
                     │   Spoke VNet    │
                     │   10.80.0.0/16  │
                     │                 │
                     │ workload-subnet  │
                     │  10.80.1.0/24   │
                     │       │         │
                     │       ▼         │
                     │ Test VM         │
                     │ 10.80.1.4       │
                     └─────────────────┘
```

---

## What This Lab Demonstrates

- Azure hub-and-spoke networking
- Azure VNet peering
- Azure Firewall Standard
- Azure Firewall network rules
- User Defined Routes (UDRs)
- Virtual Appliance next-hop routing
- Forced traffic inspection
- Private VM-to-VM communication
- Firewall allow/deny behavior
- NAT Gateway for private workload internet egress

---

## Resource Overview

| Resource | Name | Address / Configuration |
|---|---|---|
| Resource Group | `az-firewall-lab-rg` | `centralindia` |
| Hub VNet | `az-firewall-hub-vnet` | `10.70.0.0/16` |
| Firewall Subnet | `AzureFirewallSubnet` | `10.70.1.0/26` |
| Hub Workload Subnet | `hub-workload-subnet` | `10.70.10.0/24` |
| Azure Firewall | `az-firewall` | `10.70.1.4` |
| Spoke VNet | `az-firewall-spoke-vnet` | `10.80.0.0/16` |
| Spoke Workload Subnet | `workload-subnet` | `10.80.1.0/24` |
| Spoke Test VM | `az-firewall-test-vm` | `10.80.1.4` |
| Hub Server VM | `az-firewall-hub-server-vm` | `10.70.10.4` |
| Spoke Route Table | `az-firewall-spoke-rt` | BGP propagation disabled |
| Hub Route Table | `az-firewall-hub-rt` | BGP propagation disabled |
| NAT Gateway | `az-firewall-nat` | Spoke outbound access |

Both VMs were deployed **without public IP addresses**.

---

# 1. Create the Resource Group

```powershell
az group create `
  --name az-firewall-lab-rg `
  --location centralindia `
  --tags owner=tarun AutoDelete=Yes
```

---

# 2. Create the Hub VNet

```powershell
az network vnet create `
  --resource-group az-firewall-lab-rg `
  --name az-firewall-hub-vnet `
  --address-prefixes 10.70.0.0/16 `
  --location centralindia
```

Create the mandatory Azure Firewall subnet:

```powershell
az network vnet subnet create `
  --resource-group az-firewall-lab-rg `
  --vnet-name az-firewall-hub-vnet `
  --name AzureFirewallSubnet `
  --address-prefixes 10.70.1.0/26
```

Create the hub workload subnet:

```powershell
az network vnet subnet create `
  --resource-group az-firewall-lab-rg `
  --vnet-name az-firewall-hub-vnet `
  --name hub-workload-subnet `
  --address-prefixes 10.70.10.0/24
```

---

# 3. Create the Spoke VNet

```powershell
az network vnet create `
  --resource-group az-firewall-lab-rg `
  --name az-firewall-spoke-vnet `
  --address-prefixes 10.80.0.0/16 `
  --location centralindia
```

Create the workload subnet:

```powershell
az network vnet subnet create `
  --resource-group az-firewall-lab-rg `
  --vnet-name az-firewall-spoke-vnet `
  --name workload-subnet `
  --address-prefixes 10.80.1.0/24
```

---

# 4. Create Azure Firewall

Create the Firewall public IP:

```powershell
az network public-ip create `
  --resource-group az-firewall-lab-rg `
  --name az-firewall-pip `
  --location centralindia `
  --sku Standard `
  --allocation-method Static
```

Create the firewall:

```powershell
az network firewall create `
  --resource-group az-firewall-lab-rg `
  --name az-firewall `
  --location centralindia `
  --sku AZFW_VNet `
  --tier Standard
```

Create the firewall IP configuration:

```powershell
az network firewall ip-config create `
  --resource-group az-firewall-lab-rg `
  --firewall-name az-firewall `
  --name az-firewall-ipconfig `
  --public-ip-address az-firewall-pip `
  --vnet-name az-firewall-hub-vnet
```

The firewall received the private IP:

```text
10.70.1.4
```

---

# 5. Configure VNet Peering

Hub → Spoke:

```powershell
az network vnet peering create `
  --resource-group az-firewall-lab-rg `
  --vnet-name az-firewall-hub-vnet `
  --name hub-to-spoke `
  --remote-vnet az-firewall-spoke-vnet `
  --allow-vnet-access
```

Spoke → Hub:

```powershell
az network vnet peering create `
  --resource-group az-firewall-lab-rg `
  --vnet-name az-firewall-spoke-vnet `
  --name spoke-to-hub `
  --remote-vnet az-firewall-hub-vnet `
  --allow-vnet-access
```

Allow forwarded traffic:

```powershell
az network vnet peering update `
  --resource-group az-firewall-lab-rg `
  --vnet-name az-firewall-hub-vnet `
  --name hub-to-spoke `
  --allow-forwarded-traffic true

az network vnet peering update `
  --resource-group az-firewall-lab-rg `
  --vnet-name az-firewall-spoke-vnet `
  --name spoke-to-hub `
  --allow-forwarded-traffic true
```

The peerings were verified as:

```text
Connected
FullyInSync
ForwardedTraffic = True
```

---

# 6. Deploy the Private Test VMs

## Spoke VM

Create the NIC:

```powershell
az network nic create `
  --resource-group az-firewall-lab-rg `
  --name az-firewall-test-nic `
  --vnet-name az-firewall-spoke-vnet `
  --subnet workload-subnet
```

Create the VM:

```powershell
az vm create `
  --resource-group az-firewall-lab-rg `
  --name az-firewall-test-vm `
  --location centralindia `
  --image Ubuntu2204 `
  --size Standard_D2s_v5 `
  --nics az-firewall-test-nic `
  --admin-username azureuser `
  --generate-ssh-keys
```

The VM received:

```text
10.80.1.4
```

No public IP was assigned.

## Hub Server

Create the NIC:

```powershell
az network nic create `
  --resource-group az-firewall-lab-rg `
  --name az-firewall-hub-server-nic `
  --vnet-name az-firewall-hub-vnet `
  --subnet hub-workload-subnet
```

Create the VM:

```powershell
az vm create `
  --resource-group az-firewall-lab-rg `
  --name az-firewall-hub-server-vm `
  --location centralindia `
  --image Ubuntu2204 `
  --size Standard_D2s_v5 `
  --nics az-firewall-hub-server-nic `
  --admin-username azureuser `
  --generate-ssh-keys
```

The VM received:

```text
10.70.10.4
```

No public IP was assigned.

---

# 7. Start a Test HTTP Server

A simple Python HTTP server was started on the hub VM:

```powershell
az vm run-command invoke `
  --resource-group az-firewall-lab-rg `
  --name az-firewall-hub-server-vm `
  --command-id RunShellScript `
  --scripts "nohup python3 -m http.server 8080 --bind 0.0.0.0 >/tmp/http.log 2>&1 &"
```

Verify that port `8080` is listening:

```powershell
az vm run-command invoke `
  --resource-group az-firewall-lab-rg `
  --name az-firewall-hub-server-vm `
  --command-id RunShellScript `
  --scripts "ss -lntp | grep :8080"
```

Expected:

```text
0.0.0.0:8080
```

---

# 8. Configure the Spoke UDR

Create the route table:

```powershell
az network route-table create `
  --resource-group az-firewall-lab-rg `
  --name az-firewall-spoke-rt `
  --location centralindia `
  --disable-bgp-route-propagation true
```

Create the route to the hub workload:

```powershell
az network route-table route create `
  --resource-group az-firewall-lab-rg `
  --route-table-name az-firewall-spoke-rt `
  --name to-hub-workload `
  --address-prefix 10.70.10.0/24 `
  --next-hop-type VirtualAppliance `
  --next-hop-ip-address 10.70.1.4
```

Attach the route table:

```powershell
az network vnet subnet update `
  --resource-group az-firewall-lab-rg `
  --vnet-name az-firewall-spoke-vnet `
  --name workload-subnet `
  --route-table az-firewall-spoke-rt
```

The resulting route was:

```text
10.70.10.0/24
        ↓
VirtualAppliance
        ↓
10.70.1.4
```

---

# 9. Configure the Hub UDR

Create the route table:

```powershell
az network route-table create `
  --resource-group az-firewall-lab-rg `
  --name az-firewall-hub-rt `
  --location centralindia `
  --disable-bgp-route-propagation true
```

Create the return route:

```powershell
az network route-table route create `
  --resource-group az-firewall-lab-rg `
  --route-table-name az-firewall-hub-rt `
  --name to-spoke-workload `
  --address-prefix 10.80.1.0/24 `
  --next-hop-type VirtualAppliance `
  --next-hop-ip-address 10.70.1.4
```

Attach the route table:

```powershell
az network vnet subnet update `
  --resource-group az-firewall-lab-rg `
  --vnet-name az-firewall-hub-vnet `
  --name hub-workload-subnet `
  --route-table az-firewall-hub-rt
```

The return path was therefore:

```text
10.80.1.0/24
        ↓
VirtualAppliance
        ↓
10.70.1.4
```

This creates symmetric inspection through Azure Firewall.

---

# 10. Create the Azure Firewall Network Rule

The firewall network rule allows TCP traffic from the spoke workload subnet to the hub server on port `8080`.

```powershell
az network firewall network-rule create `
  --resource-group az-firewall-lab-rg `
  --firewall-name az-firewall `
  --collection-name allow-spoke-to-hub `
  --name allow-http-8080 `
  --protocols TCP `
  --source-addresses 10.80.1.0/24 `
  --destination-addresses 10.70.10.0/24 `
  --destination-ports 8080 `
  --action Allow `
  --priority 100
```

The resulting policy:

```text
Collection:    allow-spoke-to-hub
Priority:      100
Action:        Allow

Rule:          allow-http-8080
Protocol:      TCP
Source:        10.80.1.0/24
Destination:   10.70.10.0/24
Port:          8080
```

The collection was verified with:

```powershell
az network firewall show `
  --resource-group az-firewall-lab-rg `
  --name az-firewall `
  --query "networkRuleCollections" `
  --output json
```

The collection reported:

```text
provisioningState: Succeeded
```

---

# 11. Connectivity Test — Allowed Traffic

From the spoke VM, connect to the hub server:

```powershell
az vm run-command invoke `
  --resource-group az-firewall-lab-rg `
  --name az-firewall-test-vm `
  --command-id RunShellScript `
  --scripts "curl -s --max-time 10 http://10.70.10.4:8080"
```

The command successfully returned the Python HTTP server's directory listing.

This proves:

```text
Spoke VM
   ↓
Spoke UDR
   ↓
Azure Firewall
   ↓
Hub workload
   ↓
HTTP server
```

The traffic successfully passed through the firewall.

---

# 12. Negative Test — Non-Allowed Traffic

Port `8081` was intentionally not included in the firewall rule.

```powershell
az vm run-command invoke `
  --resource-group az-firewall-lab-rg `
  --name az-firewall-test-vm `
  --command-id RunShellScript `
  --scripts "curl -s --connect-timeout 3 --max-time 5 http://10.70.10.4:8081"
```

The command executed successfully on the VM, but produced no HTTP response.

This demonstrates that the explicitly allowed TCP/8080 traffic is permitted while the unapproved port is not permitted.

---

# Traffic Flow

The final traffic flow is:

```text
10.80.1.4
Spoke VM
    │
    │ TCP/8080
    ▼
Spoke UDR
10.70.10.0/24
→ 10.70.1.4
    │
    ▼
Azure Firewall
10.70.1.4
    │
    │ Firewall Network Rule
    │ ALLOW TCP/8080
    ▼
Hub VM
10.70.10.4:8080
```

The return traffic follows the corresponding hub UDR back through the firewall.

---

# Key Takeaways

### 1. UDR controls the path

The UDR determines that traffic destined for the opposite workload subnet should use the Azure Firewall as the next hop.

### 2. Azure Firewall controls the policy

The firewall determines whether the traffic is allowed.

In this lab:

```text
TCP/8080 → Allow
TCP/8081 → Not allowed
```

### 3. Both directions matter

A return route through the firewall is required for symmetric traffic inspection.

### 4. VNet peering alone does not force inspection

Without the UDRs, Azure's normal VNet routing could allow the workloads to communicate directly.

### 5. Private workloads can remain private

Neither workload VM requires a public IP. NAT Gateway provides outbound internet connectivity for the spoke workload subnet without exposing the VM directly.

---

# Cleanup

When finished with the lab, delete the entire resource group:

```powershell
az group delete `
  --name az-firewall-lab-rg `
  --yes `
  --no-wait
```

This removes:

- Azure Firewall
- Public IPs
- NAT Gateway
- Route tables
- VNets
- VNet peerings
- NICs
- Virtual machines
- Other resources in the lab resource group

---

## Lab Status

**Completed ✅**

Concepts covered:

- [x] Azure Firewall
- [x] Firewall network rules
- [x] Hub-and-spoke architecture
- [x] VNet peering
- [x] UDR / route tables
- [x] Virtual Appliance next hop
- [x] Symmetric routing
- [x] Private VM-to-VM communication
- [x] Traffic inspection
- [x] Allow / deny validation
- [x] NAT Gateway for private workload egress