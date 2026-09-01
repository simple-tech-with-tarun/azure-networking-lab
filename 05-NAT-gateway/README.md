# Azure NAT Gateway Lab

This lab demonstrates how **Azure NAT Gateway** provides controlled outbound Internet connectivity for resources deployed in an Azure Virtual Network.

The lab intentionally uses two subnets:

- **Subnet 1** — no NAT Gateway
- **Subnet 2** — NAT Gateway attached

Both subnets have `defaultOutboundAccess=false`, allowing us to clearly demonstrate the difference between a subnet with and without an explicit outbound connectivity mechanism.

---

## Objectives

- Create an Azure Virtual Network with two subnets.
- Create network interfaces without public IP addresses.
- Associate Network Security Groups with the subnets.
- Create a Standard Static Public IP for the NAT Gateway.
- Create and configure an Azure NAT Gateway.
- Associate the NAT Gateway with only one subnet.
- Deploy VMs without public IP addresses.
- Verify outbound Internet connectivity.
- Verify that the NAT Gateway public IP is used for outbound connections.
- Compare the behavior of resources with and without NAT Gateway association.

---

## Architecture

```text
                         Azure VNet
                       10.30.0.0/16
                              |
              +---------------+---------------+
              |                               |
              |                               |
       Subnet 1                         Subnet 2
     10.30.1.0/24                     10.30.2.0/24
              |                               |
             NSG                             NSG
              |                               |
          VM1 NIC                         VM2 NIC
          10.30.1.4                      10.30.2.4
              |                               |
              |                         NAT Gateway
              |                       az-nat-lab-nat
              |                               |
              |                        Public IP
              |                       20.198.107.136
              |                               |
              X                               |
       No outbound path                  Internet
```

### Important design decision

The NAT Gateway is associated with **Subnet 2**, not directly with the VM or NIC.

Therefore:

```text
VM2 → Subnet 2 → NAT Gateway → Internet
```

while:

```text
VM1 → Subnet 1 → No NAT Gateway → No outbound Internet
```

---

## Resources

| Resource | Name |
|---|---|
| Resource Group | `az-nat-lab-rg` |
| Virtual Network | `az-nat-lab-vnet` |
| VNet Address Space | `10.30.0.0/16` |
| Subnet 1 | `az-nat-lab-subnet1` |
| Subnet 1 Prefix | `10.30.1.0/24` |
| Subnet 2 | `az-nat-lab-subnet2` |
| Subnet 2 Prefix | `10.30.2.0/24` |
| VM1 NIC | `az-nat-lab-vm1-nic` |
| VM2 NIC | `az-nat-lab-vm2-nic` |
| Subnet 1 NSG | `az-nat-lab-subnet1-nsg` |
| Subnet 2 NSG | `az-nat-lab-subnet2-nsg` |
| NAT Gateway | `az-nat-lab-nat` |
| NAT Gateway Public IP | `az-nat-lab-nat-pip` |
| NAT Public IP | `20.198.107.136` |

---

## 1. Create the Resource Group

```powershell
az group create `
  --name az-nat-lab-rg `
  --location centralindia `
  --tags owner=tarun AutoDelete=Yes
```

Verify:

```powershell
az group show `
  --name az-nat-lab-rg `
  --query "{Name:name,Location:location,ProvisioningState:properties.provisioningState}" `
  -o table
```

---

## 2. Create the Virtual Network

```powershell
az network vnet create `
  --resource-group az-nat-lab-rg `
  --name az-nat-lab-vnet `
  --location centralindia `
  --address-prefixes 10.30.0.0/16 `
  --tags owner=tarun AutoDelete=Yes
```

Verify:

```powershell
az network vnet show `
  --resource-group az-nat-lab-rg `
  --name az-nat-lab-vnet `
  --query "{Name:name,AddressSpace:addressSpace.addressPrefixes,ProvisioningState:provisioningState}" `
  -o table
```

---

## 3. Create the Subnets

Both subnets explicitly disable default outbound access.

### Subnet 1

```powershell
az network vnet subnet create `
  --resource-group az-nat-lab-rg `
  --vnet-name az-nat-lab-vnet `
  --name az-nat-lab-subnet1 `
  --address-prefixes 10.30.1.0/24 `
  --default-outbound-access false
```

### Subnet 2

```powershell
az network vnet subnet create `
  --resource-group az-nat-lab-rg `
  --vnet-name az-nat-lab-vnet `
  --name az-nat-lab-subnet2 `
  --address-prefixes 10.30.2.0/24 `
  --default-outbound-access false
```

Verify:

```powershell
az network vnet subnet list `
  --resource-group az-nat-lab-rg `
  --vnet-name az-nat-lab-vnet `
  --query "[].{Subnet:name,Prefix:addressPrefix,DefaultOutboundAccess:defaultOutboundAccess}" `
  -o table
```

Expected:

```text
Subnet              Prefix          DefaultOutboundAccess
------------------  -------------   ---------------------
az-nat-lab-subnet1  10.30.1.0/24    False
az-nat-lab-subnet2  10.30.2.0/24    False
```

---

## 4. Create the Network Interfaces

The NICs are created without public IP addresses.

### VM1 NIC

```powershell
az network nic create `
  --resource-group az-nat-lab-rg `
  --name az-nat-lab-vm1-nic `
  --vnet-name az-nat-lab-vnet `
  --subnet az-nat-lab-subnet1 `
  --tags owner=tarun AutoDelete=Yes
```

### VM2 NIC

```powershell
az network nic create `
  --resource-group az-nat-lab-rg `
  --name az-nat-lab-vm2-nic `
  --vnet-name az-nat-lab-vnet `
  --subnet az-nat-lab-subnet2 `
  --tags owner=tarun AutoDelete=Yes
```

Verify:

```powershell
az network nic list `
  --resource-group az-nat-lab-rg `
  --query "[].{NIC:name,PrivateIP:ipConfigurations[0].privateIPAddress,Subnet:ipConfigurations[0].subnet.id,PublicIP:ipConfigurations[0].publicIPAddress}" `
  -o table
```

The NICs received private addresses:

```text
VM1 NIC → 10.30.1.4
VM2 NIC → 10.30.2.4
```

No public IP is associated with either NIC.

---

## 5. Create Network Security Groups

Create one NSG for each subnet.

```powershell
az network nsg create `
  --resource-group az-nat-lab-rg `
  --name az-nat-lab-subnet1-nsg `
  --location centralindia `
  --tags owner=tarun AutoDelete=Yes
```

```powershell
az network nsg create `
  --resource-group az-nat-lab-rg `
  --name az-nat-lab-subnet2-nsg `
  --location centralindia `
  --tags owner=tarun AutoDelete=Yes
```

---

## 6. Associate the NSGs with the Subnets

```powershell
az network vnet subnet update `
  --resource-group az-nat-lab-rg `
  --vnet-name az-nat-lab-vnet `
  --name az-nat-lab-subnet1 `
  --network-security-group az-nat-lab-subnet1-nsg
```

```powershell
az network vnet subnet update `
  --resource-group az-nat-lab-rg `
  --vnet-name az-nat-lab-vnet `
  --name az-nat-lab-subnet2 `
  --network-security-group az-nat-lab-subnet2-nsg
```

---

## 7. Allow SSH

An inbound SSH rule was added to both NSGs.

```powershell
az network nsg rule create `
  --resource-group az-nat-lab-rg `
  --nsg-name az-nat-lab-subnet1-nsg `
  --name allow-ssh `
  --priority 100 `
  --direction Inbound `
  --access Allow `
  --protocol Tcp `
  --source-address-prefixes "*" `
  --destination-address-prefixes "*" `
  --destination-port-ranges 22
```

```powershell
az network nsg rule create `
  --resource-group az-nat-lab-rg `
  --nsg-name az-nat-lab-subnet2-nsg `
  --name allow-ssh `
  --priority 100 `
  --direction Inbound `
  --access Allow `
  --protocol Tcp `
  --source-address-prefixes "*" `
  --destination-address-prefixes "*" `
  --destination-port-ranges 22
```

---

## 8. Create the NAT Gateway Public IP

Create a **Standard Static** public IP.

```powershell
az network public-ip create `
  --resource-group az-nat-lab-rg `
  --name az-nat-lab-nat-pip `
  --location centralindia `
  --allocation-method Static `
  --sku Standard `
  --tags owner=tarun AutoDelete=Yes
```

The assigned public IP was:

```text
20.198.107.136
```

Verify:

```powershell
az network public-ip show `
  --resource-group az-nat-lab-rg `
  --name az-nat-lab-nat-pip `
  --query "{Name:name,IP:ipAddress,SKU:sku.name,Allocation:publicIPAllocationMethod,State:provisioningState}" `
  -o table
```

---

## 9. Create the NAT Gateway

```powershell
az network nat gateway create `
  --resource-group az-nat-lab-rg `
  --name az-nat-lab-nat `
  --location centralindia `
  --public-ip-addresses az-nat-lab-nat-pip `
  --idle-timeout 10 `
  --tags owner=tarun AutoDelete=Yes
```

Verify:

```powershell
az network nat gateway show `
  --resource-group az-nat-lab-rg `
  --name az-nat-lab-nat `
  --query "{Name:name,Location:location,State:provisioningState,IdleTimeout:idleTimeoutInMinutes,PublicIPs:publicIpAddresses[].id}" `
  -o table
```

---

## 10. Associate the NAT Gateway with Subnet 2

The NAT Gateway is intentionally associated with **only Subnet 2**.

```powershell
az network vnet subnet update `
  --resource-group az-nat-lab-rg `
  --vnet-name az-nat-lab-vnet `
  --name az-nat-lab-subnet2 `
  --nat-gateway az-nat-lab-nat
```

Verify:

```powershell
az network vnet subnet show `
  --resource-group az-nat-lab-rg `
  --vnet-name az-nat-lab-vnet `
  --name az-nat-lab-subnet2 `
  --query "{Subnet:name,Prefix:addressPrefix,NATGateway:natGateway.id,DefaultOutboundAccess:defaultOutboundAccess}" `
  -o table
```

Expected:

```text
Subnet              Prefix          NATGateway           DefaultOutboundAccess
------------------  -------------   -------------------  ---------------------
az-nat-lab-subnet2  10.30.2.0/24    az-nat-lab-nat        False
```

Subnet 1 remains intentionally unassociated with a NAT Gateway.

---

## 11. Create the Virtual Machines

The VMs use the existing NICs.

### VM1

```powershell
az vm create `
  --resource-group az-nat-lab-rg `
  --name az-nat-lab-vm1 `
  --location centralindia `
  --image Ubuntu2204 `
  --size Standard_D2s_v5 `
  --nics az-nat-lab-vm1-nic `
  --admin-username azureuser `
  --generate-ssh-keys `
  --tags owner=tarun AutoDelete=Yes
```

### VM2

```powershell
az vm create `
  --resource-group az-nat-lab-rg `
  --name az-nat-lab-vm2 `
  --location centralindia `
  --image Ubuntu2204 `
  --size Standard_D2s_v5 `
  --nics az-nat-lab-vm2-nic `
  --admin-username azureuser `
  --generate-ssh-keys `
  --tags owner=tarun AutoDelete=Yes
```

Both VMs were created without public IP addresses.

```text
VM1 → 10.30.1.4
VM2 → 10.30.2.4
```

---

## 12. Test Outbound Internet Connectivity

### VM1

```powershell
az vm run-command invoke `
  --resource-group az-nat-lab-rg `
  --name az-nat-lab-vm1 `
  --command-id RunShellScript `
  --scripts "curl -s https://api.ipify.org"
```

Result:

```text
stdout: empty
```

VM1 does not have an explicit outbound Internet path because:

- Its subnet has `defaultOutboundAccess=false`.
- Its subnet has no NAT Gateway.
- Its NIC has no public IP.

---

### VM2

```powershell
az vm run-command invoke `
  --resource-group az-nat-lab-rg `
  --name az-nat-lab-vm2 `
  --command-id RunShellScript `
  --scripts "curl -s https://api.ipify.org"
```

Result:

```text
20.198.107.136
```

The returned public IP is the NAT Gateway's public IP.

This confirms that outbound traffic from VM2 is being translated by the NAT Gateway.

---

## Key Observation

The VMs themselves do **not** require public IP addresses for outbound Internet access when their subnet is associated with a NAT Gateway.

The flow is:

```text
VM private IP
      |
      ▼
Subnet
      |
      ▼
NAT Gateway
      |
      | Source NAT
      ▼
NAT Public IP
      |
      ▼
Internet
```

For this lab:

```text
10.30.2.4
    ↓
az-nat-lab-subnet2
    ↓
az-nat-lab-nat
    ↓
20.198.107.136
    ↓
Internet
```

---

## NAT Gateway vs Public IP on a VM

A NAT Gateway provides **outbound-only** Internet connectivity for resources in an associated subnet.

This is different from assigning a public IP directly to a VM.

### Public IP on VM

```text
Internet
   ↕
VM Public IP
   ↕
VM
```

The VM has a directly reachable public endpoint, subject to NSG and other controls.

### NAT Gateway

```text
VM
 │
 ▼
Subnet
 │
 ▼
NAT Gateway
 │
 ▼
Internet
```

The VM remains private while the NAT Gateway provides outbound Internet connectivity.

---

## What This Lab Demonstrated

- NAT Gateway is associated at the **subnet level**.
- A subnet can use NAT Gateway for outbound Internet connectivity without giving individual VMs public IP addresses.
- Multiple resources in the subnet can share the NAT Gateway's public IP.
- `defaultOutboundAccess=false` removes implicit outbound connectivity and makes the explicit NAT configuration visible.
- A subnet without NAT Gateway does not automatically inherit NAT connectivity from another subnet.
- NAT Gateway performs source network address translation for outbound connections.
- The public IP observed by external services is the NAT Gateway public IP, not the VM's private IP.

---

## Cleanup

Delete the entire resource group when the lab is complete:

```powershell
az group delete `
  --name az-nat-lab-rg `
  --yes `
  --no-wait
```

Verify deletion:

```powershell
az group show `
  --name az-nat-lab-rg
```

If the resource group has finished deleting, Azure CLI will report that the resource group does not exist.

---

## Git Branch

```text
feat/NAT-gateway
```

This lab is part of the Azure Networking learning series and builds on the previous networking labs covering:

- Network Foundation
- Network Security Groups
- User Defined Routes
- Azure Load Balancer
- NAT Gateway