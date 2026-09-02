# Azure Hub-and-Spoke + Private Endpoint Lab

This lab demonstrates how to build a simple **Hub-and-Spoke network architecture in Microsoft Azure** and use **Azure Private Endpoint** and **Private DNS** to provide private connectivity to an Azure Storage account.

The lab focuses on understanding how Azure networking components work together:

* Hub-and-Spoke VNet architecture
* VNet peering
* Private-only virtual machines
* NAT Gateway for controlled outbound Internet access
* Network Security Groups
* Azure Private Endpoint
* Azure Private DNS
* Private connectivity to Azure Storage
* DNS resolution for Private Endpoints
* NSG rule behavior and effective security rules

The lab intentionally avoids public IP addresses on the virtual machines.

---

## Objectives

* Create a Hub VNet and a Spoke VNet.
* Create dedicated subnets for the Hub and Spoke workloads.
* Disable implicit default outbound access.
* Deploy VMs without public IP addresses.
* Configure NAT Gateway for explicit outbound Internet access.
* Create and configure Network Security Groups.
* Establish VNet peering between Hub and Spoke.
* Verify private communication between the VNets.
* Create an Azure Storage account.
* Disable public network access to the Storage account.
* Create an Azure Private Endpoint for the Storage account.
* Create a Private DNS zone for Azure Blob Storage.
* Link the Private DNS zone to both Hub and Spoke VNets.
* Verify DNS resolution to the Private Endpoint private IP.
* Verify TCP connectivity to the Private Endpoint.
* Demonstrate NSG rule behavior using ICMP and TCP/22.
* Inspect effective NSG rules applied to a VM NIC.

---

# Architecture

```text
                         Azure Hub-and-Spoke Network


                    HUB VNET
                   10.0.0.0/16
                        |
                        |
              management-subnet
                  10.0.1.0/24
                        |
                     Hub VM
                   10.0.1.4
                        |
                    NAT Gateway
                        |
                 Hub NAT Public IP
                   40.80.88.223
                        |
                    Internet
                        
                        ||
                        || VNet Peering
                        ||
                        \/
                        
                    SPOKE VNET
                   10.1.0.0/16
                        |
                    app-subnet
                  10.1.1.0/24
                        |
                    Spoke VM
                   10.1.1.4
                        |
                    NAT Gateway
                        |
                Spoke NAT Public IP
                   20.197.58.221
                        |
                    Internet


                 Private Endpoint
                   10.1.1.5
                        |
                        |
              Azure Storage Account
           azhubspokelab260903
                        |
                        |
            Private DNS Zone
      privatelink.blob.core.windows.net
                        |
             +----------+----------+
             |                     |
          Hub VNet             Spoke VNet
          DNS Link             DNS Link
```

---

## Important Design Decisions

### 1. VMs do not have public IP addresses

Both VMs use private addresses only:

```text
Hub VM
10.0.1.4

Spoke VM
10.1.1.4
```

There is therefore no direct Internet-facing endpoint for either VM.

---

### 2. Default outbound access is disabled

Both workload subnets use:

```text
defaultOutboundAccess=false
```

This removes implicit outbound Internet connectivity.

Explicit NAT Gateways are then used to provide outbound connectivity.

---

### 3. NAT Gateway is attached to the subnet

The NAT Gateway is associated with the subnet rather than directly with a VM or NIC.

```text
VM
 |
 v
Subnet
 |
 v
NAT Gateway
 |
 v
Public IP
 |
 v
Internet
```

---

### 4. VNet peering provides private connectivity

The Hub and Spoke VNets are connected using bidirectional VNet peering.

```text
Hub VNet
10.0.0.0/16
     |
     | VNet Peering
     |
Spoke VNet
10.1.0.0/16
```

This allows private IP communication between resources in the two VNets.

---

### 5. Private Endpoint provides private access to Storage

The Storage account is accessed through a Private Endpoint:

```text
Spoke VM
10.1.1.4
     |
     v
Private Endpoint
10.1.1.5
     |
     v
Azure Storage
```

The Storage account's public network access is disabled.

---

# Resources

| Resource                    | Name                                |
| --------------------------- | ----------------------------------- |
| Resource Group              | `az-hub-spoke-lab-rg`               |
| Location                    | `centralindia`                      |
| Hub VNet                    | `az-hub-vnet`                       |
| Hub VNet Address Space      | `10.0.0.0/16`                       |
| Hub Subnet                  | `management-subnet`                 |
| Hub Subnet Prefix           | `10.0.1.0/24`                       |
| Hub VM                      | `az-hub-vm`                         |
| Hub VM NIC                  | `az-hub-vm-nic`                     |
| Hub VM Private IP           | `10.0.1.4`                          |
| Hub NSG                     | `az-hub-vm-nsg`                     |
| Hub NAT Gateway             | `az-hub-nat`                        |
| Hub NAT Public IP           | `az-hub-nat-pip`                    |
| Hub NAT Public IP Address   | `40.80.88.223`                      |
| Spoke VNet                  | `az-spoke-vnet`                     |
| Spoke VNet Address Space    | `10.1.0.0/16`                       |
| Spoke Subnet                | `app-subnet`                        |
| Spoke Subnet Prefix         | `10.1.1.0/24`                       |
| Spoke VM                    | `az-spoke-vm`                       |
| Spoke VM NIC                | `az-spoke-vm-nic`                   |
| Spoke VM Private IP         | `10.1.1.4`                          |
| Spoke NSG                   | `az-spoke-vm-nsg`                   |
| Spoke NAT Gateway           | `az-spoke-nat`                      |
| Spoke NAT Public IP         | `az-spoke-nat-pip`                  |
| Spoke NAT Public IP Address | `20.197.58.221`                     |
| Hub → Spoke Peering         | `hub-to-spoke`                      |
| Spoke → Hub Peering         | `spoke-to-hub`                      |
| Storage Account             | `azhubspokelab260903`               |
| Private Endpoint            | `az-spoke-storage-pe`               |
| Private Endpoint IP         | `10.1.1.5`                          |
| Private DNS Zone            | `privatelink.blob.core.windows.net` |
| Hub DNS Link                | `az-hub-vnet-dns-link`              |
| Spoke DNS Link              | `az-spoke-vnet-dns-link`            |

---

# 1. Create the Resource Group

```powershell
az group create `
  --name az-hub-spoke-lab-rg `
  --location centralindia `
  --tags owner=tarun AutoDelete=Yes
```

Verify:

```powershell
az group show `
  --name az-hub-spoke-lab-rg `
  --query "{Name:name,Location:location,ProvisioningState:properties.provisioningState}" `
  -o table
```

---

# 2. Create the Hub VNet

```powershell
az network vnet create `
  --resource-group az-hub-spoke-lab-rg `
  --name az-hub-vnet `
  --location centralindia `
  --address-prefixes 10.0.0.0/16 `
  --tags owner=tarun AutoDelete=Yes
```

Verify:

```powershell
az network vnet show `
  --resource-group az-hub-spoke-lab-rg `
  --name az-hub-vnet `
  --query "{Name:name,AddressSpace:addressSpace.addressPrefixes,ProvisioningState:provisioningState}" `
  -o table
```

---

# 3. Create the Hub Subnet

```powershell
az network vnet subnet create `
  --resource-group az-hub-spoke-lab-rg `
  --vnet-name az-hub-vnet `
  --name management-subnet `
  --address-prefixes 10.0.1.0/24 `
  --default-outbound-access false `
  --private-endpoint-network-policies Disabled
```

Verify:

```powershell
az network vnet subnet show `
  --resource-group az-hub-spoke-lab-rg `
  --vnet-name az-hub-vnet `
  --name management-subnet `
  --query "{Name:name,Prefix:addressPrefix,DefaultOutboundAccess:defaultOutboundAccess,PrivateEndpointPolicies:privateEndpointNetworkPolicies}" `
  -o table
```

---

# 4. Create the Spoke VNet

```powershell
az network vnet create `
  --resource-group az-hub-spoke-lab-rg `
  --name az-spoke-vnet `
  --location centralindia `
  --address-prefixes 10.1.0.0/16 `
  --tags owner=tarun AutoDelete=Yes
```

Verify:

```powershell
az network vnet show `
  --resource-group az-hub-spoke-lab-rg `
  --name az-spoke-vnet `
  --query "{Name:name,AddressSpace:addressSpace.addressPrefixes,ProvisioningState:provisioningState}" `
  -o table
```

---

# 5. Create the Spoke Subnet

```powershell
az network vnet subnet create `
  --resource-group az-hub-spoke-lab-rg `
  --vnet-name az-spoke-vnet `
  --name app-subnet `
  --address-prefixes 10.1.1.0/24 `
  --default-outbound-access false `
  --private-endpoint-network-policies Disabled
```

Verify:

```powershell
az network vnet subnet show `
  --resource-group az-hub-spoke-lab-rg `
  --vnet-name az-spoke-vnet `
  --name app-subnet `
  --query "{Name:name,Prefix:addressPrefix,DefaultOutboundAccess:defaultOutboundAccess,PrivateEndpointPolicies:privateEndpointNetworkPolicies}" `
  -o table
```

---

# 6. Create Network Security Groups

## Hub NSG

```powershell
az network nsg create `
  --resource-group az-hub-spoke-lab-rg `
  --name az-hub-vm-nsg `
  --location centralindia `
  --tags owner=tarun AutoDelete=Yes
```

## Spoke NSG

```powershell
az network nsg create `
  --resource-group az-hub-spoke-lab-rg `
  --name az-spoke-vm-nsg `
  --location centralindia `
  --tags owner=tarun AutoDelete=Yes
```

---

# 7. Associate NSGs with the Subnets

## Hub

```powershell
az network vnet subnet update `
  --resource-group az-hub-spoke-lab-rg `
  --vnet-name az-hub-vnet `
  --name management-subnet `
  --network-security-group az-hub-vm-nsg
```

## Spoke

```powershell
az network vnet subnet update `
  --resource-group az-hub-spoke-lab-rg `
  --vnet-name az-spoke-vnet `
  --name app-subnet `
  --network-security-group az-spoke-vm-nsg
```

The NSGs were intentionally associated at the **subnet level**.

---

# 8. Create Hub NAT Gateway

Create the Standard Static public IP:

```powershell
az network public-ip create `
  --resource-group az-hub-spoke-lab-rg `
  --name az-hub-nat-pip `
  --location centralindia `
  --allocation-method Static `
  --sku Standard `
  --tags owner=tarun AutoDelete=Yes
```

Create the NAT Gateway:

```powershell
az network nat gateway create `
  --resource-group az-hub-spoke-lab-rg `
  --name az-hub-nat `
  --location centralindia `
  --public-ip-addresses az-hub-nat-pip `
  --idle-timeout 10 `
  --tags owner=tarun AutoDelete=Yes
```

Associate it with the Hub management subnet:

```powershell
az network vnet subnet update `
  --resource-group az-hub-spoke-lab-rg `
  --vnet-name az-hub-vnet `
  --name management-subnet `
  --nat-gateway az-hub-nat
```

The Hub NAT Gateway public IP was:

```text
40.80.88.223
```

---

# 9. Create Spoke NAT Gateway

Create the public IP:

```powershell
az network public-ip create `
  --resource-group az-hub-spoke-lab-rg `
  --name az-spoke-nat-pip `
  --location centralindia `
  --allocation-method Static `
  --sku Standard `
  --tags owner=tarun AutoDelete=Yes
```

Create the NAT Gateway:

```powershell
az network nat gateway create `
  --resource-group az-hub-spoke-lab-rg `
  --name az-spoke-nat `
  --location centralindia `
  --public-ip-addresses az-spoke-nat-pip `
  --idle-timeout 10 `
  --tags owner=tarun AutoDelete=Yes
```

Associate it with the Spoke application subnet:

```powershell
az network vnet subnet update `
  --resource-group az-hub-spoke-lab-rg `
  --vnet-name az-spoke-vnet `
  --name app-subnet `
  --nat-gateway az-spoke-nat
```

The Spoke NAT Gateway public IP was:

```text
20.197.58.221
```

---

# 10. Create the VM NICs

## Hub NIC

```powershell
az network nic create `
  --resource-group az-hub-spoke-lab-rg `
  --name az-hub-vm-nic `
  --vnet-name az-hub-vnet `
  --subnet management-subnet `
  --tags owner=tarun AutoDelete=Yes
```

## Spoke NIC

```powershell
az network nic create `
  --resource-group az-hub-spoke-lab-rg `
  --name az-spoke-vm-nic `
  --vnet-name az-spoke-vnet `
  --subnet app-subnet `
  --tags owner=tarun AutoDelete=Yes
```

Verify:

```powershell
az network nic list `
  --resource-group az-hub-spoke-lab-rg `
  --query "[].{NIC:name,PrivateIP:ipConfigurations[0].privateIPAddress,Subnet:ipConfigurations[0].subnet.id,PublicIP:ipConfigurations[0].publicIPAddress}" `
  -o table
```

Expected private addresses:

```text
Hub VM NIC     → 10.0.1.4
Spoke VM NIC   → 10.1.1.4
```

Neither NIC has a public IP.

---

# 11. Create the Virtual Machines

Both VMs use Ubuntu 22.04 and the `Standard_D2s_v5` SKU.

## Hub VM

```powershell
az vm create `
  --resource-group az-hub-spoke-lab-rg `
  --name az-hub-vm `
  --location centralindia `
  --image Ubuntu2204 `
  --size Standard_D2s_v5 `
  --nics az-hub-vm-nic `
  --admin-username azureuser `
  --generate-ssh-keys `
  --tags owner=tarun AutoDelete=Yes
```

## Spoke VM

```powershell
az vm create `
  --resource-group az-hub-spoke-lab-rg `
  --name az-spoke-vm `
  --location centralindia `
  --image Ubuntu2204 `
  --size Standard_D2s_v5 `
  --nics az-spoke-vm-nic `
  --admin-username azureuser `
  --generate-ssh-keys `
  --tags owner=tarun AutoDelete=Yes
```

Neither VM has a public IP.

---

# 12. Verify NAT Gateway Outbound Connectivity

## Hub VM

```powershell
az vm run-command invoke `
  --resource-group az-hub-spoke-lab-rg `
  --name az-hub-vm `
  --command-id RunShellScript `
  --scripts "curl -4 -s --max-time 10 https://api.ipify.org; echo"
```

Result:

```text
40.80.88.223
```

The Hub VM's Internet-facing source IP is the Hub NAT Gateway public IP.

---

## Spoke VM

```powershell
az vm run-command invoke `
  --resource-group az-hub-spoke-lab-rg `
  --name az-spoke-vm `
  --command-id RunShellScript `
  --scripts "curl -4 -s --max-time 10 https://api.ipify.org; echo"
```

Result:

```text
20.197.58.221
```

The Spoke VM's Internet-facing source IP is the Spoke NAT Gateway public IP.

---

# 13. Create VNet Peering

Create Hub → Spoke peering:

```powershell
az network vnet peering create `
  --resource-group az-hub-spoke-lab-rg `
  --vnet-name az-hub-vnet `
  --name hub-to-spoke `
  --remote-vnet az-spoke-vnet `
  --allow-vnet-access
```

Create Spoke → Hub peering:

```powershell
az network vnet peering create `
  --resource-group az-hub-spoke-lab-rg `
  --vnet-name az-spoke-vnet `
  --name spoke-to-hub `
  --remote-vnet az-hub-vnet `
  --allow-vnet-access
```

Verify:

```powershell
az network vnet peering list `
  --resource-group az-hub-spoke-lab-rg `
  --vnet-name az-hub-vnet `
  --query "[].{Name:name,State:peeringState,Sync:peeringSyncLevel,AllowVNetAccess:allowVirtualNetworkAccess}" `
  -o table
```

Expected:

```text
Name            State       Sync
--------------  ----------  ------------
hub-to-spoke    Connected   FullyInSync
```

The reverse peering was also verified as `Connected` and `FullyInSync`.

---

# 14. Test Hub-to-Spoke Connectivity

Before peering, the VMs could not communicate across the VNets.

After peering, the Hub VM successfully reached the Spoke VM:

```powershell
az vm run-command invoke `
  --resource-group az-hub-spoke-lab-rg `
  --name az-hub-vm `
  --command-id RunShellScript `
  --scripts "ping -c 3 10.1.1.4"
```

Result:

```text
3 packets transmitted
3 packets received
0% packet loss
```

---

# 15. Test Spoke-to-Hub Connectivity

```powershell
az vm run-command invoke `
  --resource-group az-hub-spoke-lab-rg `
  --name az-spoke-vm `
  --command-id RunShellScript `
  --scripts "ping -c 3 10.0.1.4"
```

Result:

```text
3 packets transmitted
3 packets received
0% packet loss
```

This demonstrates bidirectional private connectivity through VNet peering.

---

# 16. Create the Storage Account

Storage account:

```text
azhubspokelab260903
```

The Storage account was configured as:

```text
Kind:                    StorageV2
SKU:                     Standard_LRS
TLS:                     1.2
Allow Blob Public Access: Disabled
Public Network Access:   Disabled
```

Public network access was explicitly disabled:

```powershell
az storage account update `
  --resource-group az-hub-spoke-lab-rg `
  --name azhubspokelab260903 `
  --public-network-access Disabled
```

Verify:

```powershell
az storage account show `
  --resource-group az-hub-spoke-lab-rg `
  --name azhubspokelab260903 `
  --query "{Name:name,PublicNetworkAccess:publicNetworkAccess,AllowBlobPublicAccess:allowBlobPublicAccess}" `
  -o table
```

Expected:

```text
PublicNetworkAccess    Disabled
AllowBlobPublicAccess  False
```

---

# 17. Create the Private Endpoint

The Private Endpoint was created in the Spoke application subnet.

```text
Private Endpoint:
az-spoke-storage-pe

Private IP:
10.1.1.5
```

The Private Endpoint connects to the Blob service of:

```text
azhubspokelab260903
```

The connection was successfully approved and provisioning completed successfully.

The resulting network path is:

```text
Spoke VM
10.1.1.4
   |
   v
Private Endpoint
10.1.1.5
   |
   v
Azure Storage
```

---

# 18. Create the Private DNS Zone

The DNS zone used for Azure Blob Private Endpoints is:

```text
privatelink.blob.core.windows.net
```

Create it:

```powershell
az network private-dns zone create `
  --resource-group az-hub-spoke-lab-rg `
  --name privatelink.blob.core.windows.net `
  --tags owner=tarun AutoDelete=Yes
```

---

# 19. Create the Private DNS Record

The Storage account hostname was mapped to the Private Endpoint IP:

```text
azhubspokelab260903
        ↓
10.1.1.5
```

The resulting DNS record is:

```text
azhubspokelab260903.privatelink.blob.core.windows.net
A → 10.1.1.5
```

---

# 20. Link Private DNS to the Spoke VNet

```powershell
az network private-dns link vnet create `
  --resource-group az-hub-spoke-lab-rg `
  --zone-name privatelink.blob.core.windows.net `
  --name az-spoke-vnet-dns-link `
  --virtual-network az-spoke-vnet `
  --registration-enabled false `
  --tags owner=tarun AutoDelete=Yes
```

---

# 21. Link Private DNS to the Hub VNet

The Hub also needs to resolve the Storage hostname to the Private Endpoint IP.

```powershell
az network private-dns link vnet create `
  --resource-group az-hub-spoke-lab-rg `
  --zone-name privatelink.blob.core.windows.net `
  --name az-hub-vnet-dns-link `
  --virtual-network az-hub-vnet `
  --registration-enabled false `
  --tags owner=tarun AutoDelete=Yes
```

Both DNS links reached:

```text
virtualNetworkLinkState = Completed
```

---

# 22. Verify Private DNS Resolution

## From the Spoke VM

```powershell
az vm run-command invoke `
  --resource-group az-hub-spoke-lab-rg `
  --name az-spoke-vm `
  --command-id RunShellScript `
  --scripts "getent hosts azhubspokelab260903.blob.core.windows.net"
```

Expected:

```text
10.1.1.5
```

---

## From the Hub VM

```powershell
az vm run-command invoke `
  --resource-group az-hub-spoke-lab-rg `
  --name az-hub-vm `
  --command-id RunShellScript `
  --scripts "getent hosts azhubspokelab260903.blob.core.windows.net"
```

Expected:

```text
10.1.1.5
```

The Hub VM therefore resolves the Storage hostname to the **Private Endpoint private IP**, rather than a public Storage endpoint.

---

# 23. Verify Private Endpoint TCP Connectivity

## From Spoke

```powershell
az vm run-command invoke `
  --resource-group az-hub-spoke-lab-rg `
  --name az-spoke-vm `
  --command-id RunShellScript `
  --scripts "nc -zv -w 5 10.1.1.5 443"
```

Result:

```text
Connection to 10.1.1.5 443 port [tcp/https] succeeded!
```

---

## From Hub

```powershell
az vm run-command invoke `
  --resource-group az-hub-spoke-lab-rg `
  --name az-hub-vm `
  --command-id RunShellScript `
  --scripts "nc -zv -w 5 10.1.1.5 443"
```

Result:

```text
Connection to 10.1.1.5 443 port [tcp/https] succeeded!
```

This demonstrates that the Hub can reach the Private Endpoint through the Hub-to-Spoke VNet peering connection.

---

# 24. NSG Default Rules

The custom NSG rules were intentionally kept empty initially.

Azure automatically provides default rules such as:

```text
Inbound

65000  AllowVnetInBound
65001  AllowAzureLoadBalancerInBound
65500  DenyAllInBound
```

and:

```text
Outbound

65000  AllowVnetOutBound
65001  AllowInternetOutBound
65500  DenyAllOutBound
```

The important observation is:

```text
AllowVnetInBound
```

allows traffic between resources considered part of the `VirtualNetwork` service tag.

Therefore, after VNet peering, Hub → Spoke TCP/22 was initially allowed even though no custom SSH rule existed.

---

# 25. NSG ICMP Deny Experiment

A temporary inbound rule was created on the Spoke NSG:

```text
Name:        Deny-Hub-To-Spoke
Priority:    100
Direction:   Inbound
Access:      Deny
Protocol:    ICMP
Source:      10.0.1.0/24
Destination: 10.1.1.0/24
```

Before the rule:

```text
Hub → Spoke ping
3/3 successful
0% packet loss
```

After the rule:

```text
Hub → Spoke ping
100% packet loss
```

The reverse direction remained functional:

```text
Spoke → Hub
3/3 successful
0% packet loss
```

This demonstrated that NSG rules are directional.

The temporary rule was then deleted.

---

# 26. NSG TCP/22 Deny Experiment

A second temporary rule was created:

```text
Name:        Deny-Hub-SSH-To-Spoke
Priority:    100
Direction:   Inbound
Access:      Deny
Protocol:    TCP
Source:      10.0.1.0/24
Destination: 10.1.1.0/24
Destination Port: 22
```

Before the rule:

```text
Hub → Spoke TCP/22
Connection succeeded
```

After the rule:

```text
Hub → Spoke TCP/22
Connection timed out
```

However, ICMP remained functional:

```text
Hub → Spoke ICMP
3/3 successful
0% packet loss
```

This demonstrated that NSG rules can target specific protocols and ports.

The temporary rule was then deleted.

---

# 27. Effective NSG Inspection

The effective NSG configuration was inspected on the Spoke VM NIC.

```powershell
az network nic list-effective-nsg `
  --resource-group az-hub-spoke-lab-rg `
  --name az-spoke-vm-nic `
  --output json
```

The effective configuration confirmed that the subnet-level NSG was being applied to the NIC.

The effective rules included:

```text
AllowVnetInBound
AllowAzureLoadBalancerInBound
DenyAllInBound

AllowVnetOutBound
AllowInternetOutBound
DenyAllOutBound
```

The `VirtualNetwork` service tag was expanded by Azure to include the relevant private address ranges.

For this lab, the effective `VirtualNetwork` range included:

```text
10.0.0.0/15
```

which encompasses:

```text
10.0.0.0/16  → Hub
10.1.0.0/16  → Spoke
```

---

# Key Observations

## Hub-and-Spoke

VNet peering provides private connectivity between the Hub and Spoke:

```text
10.0.1.4
   |
   | VNet Peering
   |
10.1.1.4
```

Both directions were successfully tested.

---

## NAT Gateway

NAT Gateway provides explicit outbound Internet connectivity without assigning public IPs to the VMs.

```text
Hub VM
10.0.1.4
   ↓
Hub NAT
   ↓
40.80.88.223
   ↓
Internet
```

and:

```text
Spoke VM
10.1.1.4
   ↓
Spoke NAT
   ↓
20.197.58.221
   ↓
Internet
```

---

## Private Endpoint

The Storage account is accessed through a private IP:

```text
10.1.1.5
```

The VM does not need to access the Storage service through its public endpoint.

---

## Private DNS

Private DNS makes the normal Storage hostname resolve to the Private Endpoint:

```text
azhubspokelab260903.blob.core.windows.net
                    ↓
                 10.1.1.5
```

Both Hub and Spoke VNet DNS links were required because both VNets needed to resolve the private Storage endpoint.

---

## NSGs

NSGs are stateful and directional.

The experiments demonstrated that:

* ICMP can be denied independently.
* TCP/22 can be denied independently.
* A custom rule with a higher priority can override a lower-priority rule.
* Default `AllowVnetInBound` permits VNet traffic unless a higher-priority custom deny blocks it.
* An NSG does not itself provide network connectivity; it controls whether traffic is allowed once a network path exists.

---

# Private Endpoint vs Public Endpoint

The intended architecture is:

```text
                    Internet
                       X
                       |
                       |
                Public Storage
                       X
                       |
                       |
                Storage Account
                       |
                       |
                Private Endpoint
                   10.1.1.5
                       |
                       |
                  Spoke VM
                   10.1.1.4
```

The Storage account has:

```text
Public Network Access = Disabled
```

Therefore, application traffic is intended to use the Private Endpoint.

---

# What This Lab Demonstrated

* Azure Hub-and-Spoke network architecture.
* VNet address-space planning.
* Subnet design.
* VNet peering.
* Private communication between peered VNets.
* Private-only VMs.
* Explicit outbound connectivity using NAT Gateway.
* Difference between outbound NAT and inbound public connectivity.
* Network Security Groups at subnet level.
* Azure default NSG rules.
* NSG priority and directional behavior.
* Protocol-specific NSG rules.
* Effective NSG inspection.
* Azure Private Endpoint.
* Private Endpoint private IP addressing.
* Azure Storage private connectivity.
* Private DNS zones.
* VNet links to Private DNS.
* DNS resolution through Private Endpoint.
* Hub access to a Private Endpoint located in the Spoke through VNet peering.

---

# Cleanup

Because all resources for this lab are contained inside a dedicated resource group, the complete lab can be removed by deleting the resource group.

```powershell
az group delete `
  --name az-hub-spoke-lab-rg `
  --yes `
  --no-wait
```

Verify deletion:

```powershell
az group show `
  --name az-hub-spoke-lab-rg
```

After deletion has completed, Azure CLI will report that the resource group does not exist.

---

# Git Branch

```text
feat/hub-spoke-private-endpoint
```

This lab is part of the Azure Networking learning series and builds on the previous networking labs covering:

* Network Foundation
* Network Security Groups
* User Defined Routes
* Azure Load Balancer
* NAT Gateway

The next lab will focus specifically on **Azure Service Tags** and how they can be used with Network Security Groups.