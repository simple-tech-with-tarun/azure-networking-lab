# Azure Bastion Private VM Lab

A hands-on Azure networking lab demonstrating how to securely administer a
private Azure VM using **Azure Bastion**, without assigning a public IP
address directly to the VM.

The lab also demonstrates the distinction between:

- Private VM connectivity
- Bastion-based administrative access
- VM outbound Internet connectivity
- Azure Bastion SKU capabilities
- Bastion Native Client / SSH tunneling

---

## Architecture

```text
                         Internet
                            │
                            │ HTTPS / SSH
                            ▼
                    ┌─────────────────┐
                    │  Azure Bastion   │
                    │     Standard     │
                    │  Native Client   │
                    │    Tunneling     │
                    └────────┬────────┘
                             │
                             │ Private connection
                             ▼
                    ┌─────────────────┐
                    │      VNet       │
                    │ 10.100.0.0/16   │
                    │                 │
                    │ ┌─────────────┐ │
                    │ │ Bastion     │ │
                    │ │ Subnet      │ │
                    │ │10.100.1.0/26│ │
                    │ └─────────────┘ │
                    │                 │
                    │ ┌─────────────┐ │
                    │ │ VM Subnet   │ │
                    │ │10.100.2.0/24│ │
                    │ │             │ │
                    │ │ VM          │ │
                    │ │10.100.2.4   │ │
                    │ │             │ │
                    │ │ No Public IP│ │
                    │ └─────────────┘ │
                    └─────────────────┘
```

---

## Objectives

By completing this lab, we demonstrate:

- Creating an Azure VNet and dedicated subnets
- Creating the required `AzureBastionSubnet`
- Deploying a Linux VM without a public IP
- Verifying that the VM is privately addressed
- Deploying Azure Bastion
- Understanding Bastion SKU capabilities
- Upgrading Bastion from Basic to Standard
- Enabling Bastion Native Client tunneling
- Connecting to a private VM using SSH through Bastion
- Verifying private VM routing
- Demonstrating that Bastion does not automatically provide outbound Internet access

---

## Azure Resources

| Resource | Name |
|---|---|
| Resource Group | `az-bastion-lab-rg` |
| Location | `centralindia` |
| VNet | `az-bastion-lab-vnet` |
| VNet Address Space | `10.100.0.0/16` |
| VM Subnet | `vm-subnet` |
| VM Subnet CIDR | `10.100.2.0/24` |
| Bastion Subnet | `AzureBastionSubnet` |
| Bastion Subnet CIDR | `10.100.1.0/26` |
| VM | `az-bastion-vm` |
| VM Private IP | `10.100.2.4` |
| VM NIC | `az-bastion-vm-nic` |
| Bastion | `az-bastion` |
| Bastion SKU | `Standard` |
| Bastion Public IP | `az-bastion-pip` |
| Public IP SKU | `Standard` |

---

# 1. Create the Resource Group

```powershell
az group create `
  --name az-bastion-lab-rg `
  --location centralindia `
  --tags owner=tarun AutoDelete=Yes
```

Verify:

```powershell
az group show `
  --name az-bastion-lab-rg `
  --query "{name:name,location:location,provisioningState:properties.provisioningState,tags:tags}" `
  --output json
```

---

# 2. Create the VNet

Create the VNet with a `/16` address space and the VM subnet.

```powershell
az network vnet create `
  --resource-group az-bastion-lab-rg `
  --name az-bastion-lab-vnet `
  --location centralindia `
  --address-prefix 10.100.0.0/16 `
  --subnet-name vm-subnet `
  --subnet-prefix 10.100.2.0/24
```

Verify:

```powershell
az network vnet show `
  --resource-group az-bastion-lab-rg `
  --name az-bastion-lab-vnet `
  --query "{name:name,addressSpace:addressSpace.addressPrefixes,subnets:subnets[].{name:name,prefix:addressPrefix}}" `
  --output json
```

Expected:

```text
VNet:
10.100.0.0/16

VM subnet:
10.100.2.0/24
```

---

# 3. Create AzureBastionSubnet

Azure Bastion requires a dedicated subnet named exactly:

```text
AzureBastionSubnet
```

Create it:

```powershell
az network vnet subnet create `
  --resource-group az-bastion-lab-rg `
  --vnet-name az-bastion-lab-vnet `
  --name AzureBastionSubnet `
  --address-prefixes 10.100.1.0/26
```

Verify:

```powershell
az network vnet show `
  --resource-group az-bastion-lab-rg `
  --name az-bastion-lab-vnet `
  --query "subnets[].{name:name,prefix:addressPrefix}" `
  --output table
```

Expected:

```text
Name                   Prefix
---------------------  -----------
vm-subnet              10.100.2.0/24
AzureBastionSubnet     10.100.1.0/26
```

---

# 4. Verify VM Subnet Outbound Configuration

The VM subnet was created with outbound access disabled.

Verify:

```powershell
az network vnet subnet show `
  --resource-group az-bastion-lab-rg `
  --vnet-name az-bastion-lab-vnet `
  --name vm-subnet `
  --query "{name:name,prefix:addressPrefix,defaultOutboundAccess:defaultOutboundAccess,natGateway:natGateway.id,routeTable:routeTable.id,networkSecurityGroup:networkSecurityGroup.id}" `
  --output json
```

The important configuration is:

```text
defaultOutboundAccess: false
natGateway: null
```

This ensures that the VM is not given outbound Internet connectivity through a NAT Gateway.

---

# 5. Create the VM NIC

Create a NIC attached only to the private VM subnet:

```powershell
az network nic create `
  --resource-group az-bastion-lab-rg `
  --name az-bastion-vm-nic `
  --vnet-name az-bastion-lab-vnet `
  --subnet vm-subnet
```

The NIC received:

```text
Private IP: 10.100.2.4
```

---

# 6. Deploy the VM Without a Public IP

Create the Ubuntu VM using the existing NIC:

```powershell
az vm create `
  --resource-group az-bastion-lab-rg `
  --name az-bastion-vm `
  --image Ubuntu2204 `
  --size Standard_D2s_v5 `
  --admin-username azureuser `
  --ssh-key-values "$env:USERPROFILE\.ssh\id_rsa.pub" `
  --nics az-bastion-vm-nic
```

Verify:

```powershell
az vm show `
  --resource-group az-bastion-lab-rg `
  --name az-bastion-vm `
  --show-details `
  --query "{name:name,powerState:powerState,privateIp:privateIps,publicIp:publicIps}" `
  --output json
```

Expected result:

```text
Private IP: 10.100.2.4
Public IP: none
```

The VM therefore cannot be accessed directly from the Internet.

---

# 7. Verify the NIC

```powershell
az network nic show `
  --resource-group az-bastion-lab-rg `
  --name az-bastion-vm-nic `
  --query "{name:name,privateIp:ipConfigurations[0].privateIPAddress,publicIp:ipConfigurations[0].publicIPAddress.id,subnet:ipConfigurations[0].subnet.id}" `
  --output json
```

Expected:

```json
{
  "name": "az-bastion-vm-nic",
  "privateIp": "10.100.2.4",
  "publicIp": null
}
```

This confirms that the VM has no public IP attached to its NIC.

---

# 8. Create the Bastion Public IP

Azure Bastion requires a public IP.

Create a Standard static public IP:

```powershell
az network public-ip create `
  --resource-group az-bastion-lab-rg `
  --name az-bastion-pip `
  --location centralindia `
  --sku Standard `
  --allocation-method Static
```

Important:

The public IP belongs to **Azure Bastion**, not the VM.

---

# 9. Create Azure Bastion

Initially, Bastion was deployed using the Basic SKU:

```powershell
az network bastion create `
  --resource-group az-bastion-lab-rg `
  --name az-bastion `
  --vnet-name az-bastion-lab-vnet `
  --public-ip-address az-bastion-pip `
  --location centralindia `
  --sku Basic
```

Verify:

```powershell
az network bastion show `
  --resource-group az-bastion-lab-rg `
  --name az-bastion `
  --query "{name:name,sku:sku.name,state:provisioningState,dnsName:dnsName,scaleUnits:scaleUnits}" `
  --output json
```

---

# 10. Basic SKU Limitation

The initial Basic Bastion deployment successfully provisioned:

```text
SKU: Basic
State: Succeeded
```

However, attempting to use:

```powershell
az network bastion ssh
```

returned:

```text
Bastion Host SKU must be Standard or Premium and Native Client must be enabled.
```

This demonstrated an important distinction:

> A successfully deployed Bastion host does not necessarily support every Bastion connection method.

The Basic SKU was therefore upgraded instead of deleting and recreating the resource.

---

# 11. Upgrade Bastion to Standard

The installed Azure CLI extension expected the SKU as an object.

This was confirmed using:

```powershell
az network bastion update --sku "??"
```

The CLI reported:

```text
Object Properties

name : The name of the sku of this Bastion Host.
Allowed values: Basic, Developer, Premium, Standard.
```

Therefore the correct syntax was:

```powershell
az network bastion update `
  --resource-group az-bastion-lab-rg `
  --name az-bastion `
  --sku name=Standard
```

Verify:

```powershell
az network bastion show `
  --resource-group az-bastion-lab-rg `
  --name az-bastion `
  --query "{name:name,sku:sku.name,state:provisioningState,scaleUnits:scaleUnits}" `
  --output json
```

Expected:

```json
{
  "name": "az-bastion",
  "sku": "Standard",
  "state": "Succeeded",
  "scaleUnits": 2
}
```

---

# 12. Enable Bastion Native Client Tunneling

Enable tunneling:

```powershell
az network bastion update `
  --resource-group az-bastion-lab-rg `
  --name az-bastion `
  --enable-tunneling true
```

Verify:

```powershell
az network bastion show `
  --resource-group az-bastion-lab-rg `
  --name az-bastion `
  --query "{name:name,sku:sku.name,state:provisioningState,enableTunneling:enableTunneling}" `
  --output json
```

Expected:

```json
{
  "name": "az-bastion",
  "sku": "Standard",
  "state": "Succeeded",
  "enableTunneling": true
}
```

---

# 13. Connect to the Private VM Through Bastion

The VM has:

```text
Private IP: 10.100.2.4
Public IP: none
```

SSH access was performed through Azure Bastion:

```powershell
az network bastion ssh `
  --name az-bastion `
  --resource-group az-bastion-lab-rg `
  --target-resource-id "/subscriptions/<subscription-id>/resourceGroups/az-bastion-lab-rg/providers/Microsoft.Compute/virtualMachines/az-bastion-vm" `
  --auth-type ssh-key `
  --username azureuser `
  --ssh-key "$env:USERPROFILE\.ssh\id_rsa"
```

The connection successfully opened an Ubuntu shell:

```text
Welcome to Ubuntu 22.04.5 LTS

azureuser@az-bastion-vm:~$
```

This proves that the VM can be administered without having a public IP.

---

# 14. Verify the VM's Network Interface

From the Bastion SSH session:

```bash
hostname
```

Expected:

```text
az-bastion-vm
```

Check the interfaces:

```bash
ip addr show
```

The primary VM interface showed:

```text
eth0
inet 10.100.2.4/24
```

This confirms that the VM is operating on the private VNet address space.

---

# 15. Verify the VM Routing Table

Run:

```bash
ip route
```

Observed:

```text
default via 10.100.2.1 dev eth0 proto dhcp src 10.100.2.4 metric 100
10.100.2.0/24 dev eth0 proto kernel scope link src 10.100.2.4 metric 100
10.100.2.1 dev eth0 proto dhcp scope link src 10.100.2.4 metric 100
168.63.129.16 via 10.100.2.1 dev eth0 proto dhcp src 10.100.2.4 metric 100
169.254.169.254 via 10.100.2.1 dev eth0 proto dhcp src 10.100.2.4 metric 100
```

The VM has a default route through the Azure virtual network gateway:

```text
default via 10.100.2.1
```

However, a default route alone does not guarantee Internet connectivity.

---

# 16. Verify Outbound Internet Connectivity

Attempt:

```bash
curl -4 -s https://api.ipify.org
```

The request did not complete.

A controlled test was performed:

```bash
timeout 5 curl -4 -v https://api.ipify.org
echo $?
```

The connection attempted to reach the external address but timed out:

```text
Trying 104.26.12.205:443...
124
```

Exit code:

```text
124
```

indicates that the `timeout` command terminated the request after five seconds.

This confirms that the VM does not currently have usable outbound Internet connectivity.

---

# Key Networking Findings

## Bastion provides management access

Azure Bastion allowed SSH access to:

```text
10.100.2.4
```

without assigning a public IP to the VM.

The management path is:

```text
Administrator
      │
      ▼
Azure Bastion
      │
      │ Private connectivity
      ▼
Private VM
10.100.2.4
```

---

## Bastion does not provide VM Internet access

A common misconception is that deploying Bastion gives private VMs Internet connectivity.

This lab demonstrated that this is not the case.

Bastion provides a secure management path to the VM.

It does not act as a general-purpose outbound NAT service for the VM.

---

## Bastion Public IP vs VM Public IP

The architecture contains a public IP:

```text
az-bastion-pip
```

but it is associated with:

```text
Azure Bastion
```

and not:

```text
az-bastion-vm
```

The VM itself remains private.

---

# Final Architecture

```text
                         Internet
                            │
                            │
                            ▼
                 ┌────────────────────┐
                 │   Bastion Public IP │
                 │    40.x.x.x         │
                 └─────────┬──────────┘
                           │
                           ▼
                 ┌────────────────────┐
                 │   Azure Bastion    │
                 │                    │
                 │       Standard     │
                 │       Tunneling    │
                 └─────────┬──────────┘
                           │
                           │ Private
                           ▼
              ┌──────────────────────────┐
              │       VNet                │
              │     10.100.0.0/16        │
              │                          │
              │  ┌────────────────────┐  │
              │  │ AzureBastionSubnet │  │
              │  │  10.100.1.0/26     │  │
              │  └────────────────────┘  │
              │                          │
              │  ┌────────────────────┐  │
              │  │     vm-subnet      │  │
              │  │   10.100.2.0/24    │  │
              │  │                    │  │
              │  │  Ubuntu VM         │  │
              │  │  10.100.2.4       │  │
              │  │                    │  │
              │  │  NO PUBLIC IP      │  │
              │  └────────────────────┘  │
              │                          │
              └──────────────────────────┘

                    VM Internet
                         X
                   No outbound NAT
```

---

# Important Lessons

### 1. Private VM ≠ inaccessible VM

A VM does not need a public IP to be administratively accessible.

Azure Bastion provides a managed entry point into the VNet.

### 2. Bastion and NAT solve different problems

| Service | Purpose |
|---|---|
| Azure Bastion | Secure administrative access to private VMs |
| NAT Gateway | Outbound Internet connectivity |
| Public IP on VM | Direct public connectivity |
| Private Endpoint | Private access to Azure/PaaS services |

### 3. Public IP placement matters

The public IP in this architecture belongs to Bastion.

The workload VM remains private.

### 4. SKU affects capabilities

The initial Basic Bastion deployment could not be used with the desired Native Client SSH workflow.

Upgrading to Standard and enabling tunneling allowed:

```text
az network bastion ssh
```

to successfully connect to the private VM.

### 5. A default route does not guarantee Internet access

The VM had:

```text
default via 10.100.2.1
```

but external HTTPS connectivity still timed out because no outbound NAT path was configured.

---

# Validation Checklist

- [x] Resource group created
- [x] VNet created
- [x] VM subnet created
- [x] AzureBastionSubnet created
- [x] VM subnet configured without default outbound access
- [x] VM NIC created
- [x] VM deployed without public IP
- [x] Bastion public IP created
- [x] Bastion deployed
- [x] Basic SKU tested
- [x] Basic → Standard upgrade completed
- [x] Native Client tunneling enabled
- [x] Bastion SSH connection successful
- [x] VM private IP verified
- [x] VM routing table inspected
- [x] Outbound Internet connectivity tested
- [x] Lack of outbound Internet connectivity confirmed

---

# Cleanup

This lab uses the resource group:

```text
az-bastion-lab-rg
```

Delete the entire lab when finished:

```powershell
az group delete `
  --name az-bastion-lab-rg `
  --yes `
  --no-wait
```

Because the resource group was created with:

```text
AutoDelete=Yes
```

it is also clearly identifiable as a temporary lab environment.

---

# What This Lab Demonstrates

This lab demonstrates a production-relevant Azure networking pattern:

> **Keep workload VMs private and use Azure Bastion for administrative access instead of exposing SSH/RDP directly to the Internet.**

The VM remained on:

```text
10.100.2.4
```

with:

```text
Public IP = None
```

while administrative access was provided through:

```text
Azure Bastion
    ↓
Standard SKU
    ↓
Native Client / SSH tunneling
    ↓
Private VM
```

This provides a practical foundation for more advanced Azure networking patterns such as:

- VNet Peering
- Hub-and-Spoke networking
- Azure Firewall
- User Defined Routes
- Private DNS architecture
- Network Security Groups
- Private workloads with controlled egress
```