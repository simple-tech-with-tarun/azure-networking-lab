Absolutely. I’d keep this README focused on the **Service Tags concept**, without adding unnecessary VM or Load Balancer infrastructure.

# Azure Service Tags Lab

A hands-on Azure networking lab demonstrating how **Service Tags** simplify Network Security Group (NSG) rules by allowing Azure-managed service IP ranges to be referenced by name instead of manually maintaining IP addresses.

---

## 🎯 Objective

Understand how Azure Service Tags work and how they can be used in NSG rules to control traffic to Azure services.

In this lab, we:

* Create an Azure Virtual Network and subnet.
* Create a Network Security Group.
* Inspect Azure-managed Service Tags.
* Create an NSG rule using the `Storage` Service Tag.
* Associate the NSG with the subnet.
* Inspect custom and default NSG rules.
* Understand the difference between Service Tags, NSG rules, and routing.

---

## 🏗️ Architecture

```text
Azure
│
└── Resource Group: az-service-tags-lab-rg
    │
    ├── VNet: az-service-tags-vnet
    │   └── Address Space: 10.20.0.0/16
    │
    └── Subnet: test-subnet
        ├── Address Space: 10.20.1.0/24
        │
        └── NSG: az-service-tags-nsg
            │
            └── Allow-Storage-Outbound
                ├── Direction: Outbound
                ├── Protocol: TCP
                ├── Source: VirtualNetwork
                ├── Destination: Storage
                └── Destination Port: 443
```

---

## 🧠 What Are Azure Service Tags?

An Azure **Service Tag** is a Microsoft-managed label representing a group of IP address prefixes associated with an Azure service or networking category.

Instead of manually maintaining a large list of IP addresses, an NSG rule can reference a Service Tag.

For example:

```text
Destination: Storage
```

instead of manually specifying many Azure Storage IP ranges.

Microsoft maintains the underlying IP prefixes represented by the Service Tag.

### Common examples

| Service Tag         | Purpose                                             |
| ------------------- | --------------------------------------------------- |
| `Storage`           | Azure Storage service IP ranges                     |
| `AzureCloud`        | Azure public cloud service IP ranges                |
| `AzureLoadBalancer` | Azure Load Balancer infrastructure                  |
| `VirtualNetwork`    | Virtual network and related virtual-network traffic |
| `Internet`          | Internet traffic category used by NSG rules         |

---

## ⚠️ Service Tags Are Not Routes

A Service Tag does **not** create a route.

This distinction is important:

```text
Routing
   │
   └── Determines where traffic should go

NSG
   │
   └── Determines whether traffic should be allowed or denied

Service Tag
   │
   └── Provides a logical destination/source match
       representing Microsoft-managed IP ranges
```

For example:

```text
Destination = Storage
```

does not tell Azure how to reach Storage.

The routing system still determines the next hop.

The Service Tag simply allows the NSG rule to recognize traffic destined for the IP ranges represented by the `Storage` tag.

---

## 🔬 Lab Configuration

### Resource Group

```text
Name:     az-service-tags-lab-rg
Location: centralindia
```

### Virtual Network

```text
Name:          az-service-tags-vnet
Address Space: 10.20.0.0/16
```

### Subnet

```text
Name:          test-subnet
Address Space: 10.20.1.0/24
```

### Network Security Group

```text
Name: az-service-tags-nsg
```

The NSG is associated with `test-subnet`.

---

## 🔐 Custom NSG Rule

The main rule created in this lab is:

```text
Name:        Allow-Storage-Outbound
Priority:    100
Direction:   Outbound
Access:      Allow
Protocol:    TCP
Source:      VirtualNetwork
Destination: Storage
Port:        443
```

The important part is:

```text
Destination = Storage
```

The rule does not contain a manually maintained list of Storage IP addresses.

---

## 🛠️ Azure CLI

### Create the NSG rule

```powershell
az network nsg rule create `
  --resource-group az-service-tags-lab-rg `
  --nsg-name az-service-tags-nsg `
  --name Allow-Storage-Outbound `
  --priority 100 `
  --direction Outbound `
  --access Allow `
  --protocol Tcp `
  --source-address-prefixes VirtualNetwork `
  --source-port-ranges "*" `
  --destination-address-prefixes Storage `
  --destination-port-ranges 443 `
  --description "Allow HTTPS traffic to Azure Storage using the Storage service tag"
```

### Inspect the custom rule

```powershell
az network nsg rule show `
  --resource-group az-service-tags-lab-rg `
  --nsg-name az-service-tags-nsg `
  --name Allow-Storage-Outbound `
  --query "{Name:name,Priority:priority,Direction:direction,Access:access,Protocol:protocol,Source:sourceAddressPrefix,Destination:destinationAddressPrefix,Port:destinationPortRange}" `
  --output table
```

Expected result:

```text
Name                    Priority  Direction  Access  Protocol  Source          Destination  Port
----------------------  --------  ---------  ------  --------  --------------  -----------  ----
Allow-Storage-Outbound  100       Outbound   Allow   Tcp       VirtualNetwork  Storage       443
```

---

## 🔎 Inspect Azure Service Tags

Azure CLI can be used to inspect the IP prefixes associated with Service Tags.

For example:

```powershell
az network list-service-tags `
  --location centralindia `
  --query "values[?name=='Storage' || name=='AzureCloud'].{Tag:name,Prefixes:length(properties.addressPrefixes)}" `
  --output table
```

This demonstrates that a Service Tag such as `Storage` represents a large collection of IP prefixes.

The NSG rule does not need to contain those individual prefixes.

---

## 📋 Inspect Custom NSG Rules

```powershell
az network nsg rule list `
  --resource-group az-service-tags-lab-rg `
  --nsg-name az-service-tags-nsg `
  --query "[].{Name:name,Priority:priority,Direction:direction,Access:access,Source:sourceAddressPrefix,Destination:destinationAddressPrefix,Port:destinationPortRange}" `
  --output table
```

---

## 📋 Inspect Default NSG Rules

Azure automatically creates default NSG rules.

```powershell
az network nsg show `
  --resource-group az-service-tags-lab-rg `
  --name az-service-tags-nsg `
  --query "defaultSecurityRules[].{Name:name,Priority:priority,Direction:direction,Access:access,Source:sourceAddressPrefix,Destination:destinationAddressPrefix,Port:destinationPortRange}" `
  --output table
```

Important default outbound rules include:

```text
AllowVnetOutBound       65000   Allow   VirtualNetwork → VirtualNetwork
AllowInternetOutBound   65001   Allow   *              → Internet
DenyAllOutBound         65500   Deny    *              → *
```

Our custom rule has priority `100`, so it is evaluated before these default rules.

---

## 🧩 Service Tag vs Resource Name

A Service Tag does **not** reference a specific resource created in your subscription.

For example:

```text
Storage Account A
Storage Account B
Storage Account C
```

are not individually represented by the `Storage` Service Tag.

Instead:

```text
Storage
   ↓
Microsoft-managed Azure Storage IP ranges
```

The Service Tag represents the applicable Azure service address ranges.

This distinction is important when designing security policies.

---

## 🔀 Service Tags vs Routing

A useful mental model:

```text
                Packet
                  │
                  ▼
          ┌───────────────┐
          │    Routing    │
          │               │
          │ Where should  │
          │ traffic go?   │
          └───────┬───────┘
                  │
                  ▼
          ┌───────────────┐
          │      NSG      │
          │               │
          │ Is traffic    │
          │ allowed?      │
          └───────┬───────┘
                  │
                  ▼
        Service Tag matching
                  │
                  ▼
          Allow / Deny
```

**Service Tags simplify traffic matching. They do not provide connectivity.**

---

## 🎓 Key Takeaways

1. **Service Tags are Microsoft-managed.**
   Users do not create the built-in tags such as `Storage` or `AzureCloud`.

2. **Service Tags represent IP ranges.**
   Azure maintains the underlying prefixes.

3. **NSGs can reference Service Tags.**
   This avoids manually maintaining large lists of Azure service IP addresses.

4. **Service Tags do not create routes.**
   Routing and security filtering are separate networking functions.

5. **A Service Tag does not identify your individual resource.**
   `Storage` represents Azure Storage service ranges, not a particular Storage Account.

6. **NSG priority still applies.**
   Our custom rule at priority `100` is evaluated before the default rules at priorities `65000+`.

---

## 🧹 Cleanup

Delete the entire lab resource group when finished:

```powershell
az group delete `
  --name az-service-tags-lab-rg `
  --yes `
  --no-wait
```

> **Note:** The `AutoDelete=Yes` tag used on the resource group is only a tag. It does not automatically delete the resource group unless separate automation has been configured.

---

## 📚 Concepts Practiced

* Azure Service Tags
* Network Security Groups
* NSG rule priorities
* Azure default NSG rules
* Azure Storage networking
* Microsoft-managed IP prefixes
* Service Tags vs IP addresses
* Service Tags vs routing
* Subnet-level NSG association
* Azure CLI networking commands