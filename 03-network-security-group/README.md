# Azure Networking Lab 3 — Network Security Groups (NSG)

## Overview

This lab demonstrates how **Azure Network Security Groups (NSGs)** control inbound and outbound network traffic.

The lab uses two VMs in separate subnets within the same VNet and progressively applies NSG rules to control SSH and network connectivity.

The focus is on understanding:

- NSG association at subnet and NIC levels
- Default NSG rules
- Inbound and outbound rules
- Rule priority
- Source-based access control
- Public IP allowlisting
- `/32` CIDR notation
- Two-sided traffic evaluation

---

## Architecture

```text
                    Azure VNet
                  10.0.0.0/16
                       │
          ┌────────────┴────────────┐
          │                         │
   Subnet 1                     Subnet 2
   10.0.1.0/24                  10.0.2.0/24
          │                         │
          │                         │
       VM1                       VM2
     10.0.1.4                  10.0.2.4
                                    │
                              Public IP
                              40.80.87.71
                                    │
                                    │
                               Internet
```

VM2 uses a NIC-level NSG:

```text
VM2
 │
 └── NIC
      │
      └── az-nsg-lab-vm2NSG
```

---

## Prerequisites

- Azure subscription
- Azure CLI
- PowerShell
- Basic knowledge of:
  - CIDR notation
  - TCP/IP
  - Azure VNets and subnets
  - SSH

---

# 1. Create the Resource Group

```powershell
az group create `
  --name az-nsg-lab-rg `
  --location centralindia `
  --tags owner=tarun
```

Verify:

```powershell
az group show `
  --name az-nsg-lab-rg `
  --query "{Name:name,Location:location}" `
  -o table
```

---

# 2. Create the VNet and Subnets

Create the VNet with subnet 1:

```powershell
az network vnet create `
  -g az-nsg-lab-rg `
  --location centralindia `
  --name az-nsg-lab-net `
  --address-prefixes 10.0.0.0/16 `
  --subnet-name az-nsg-lab-subnet1 `
  --subnet-prefixes 10.0.1.0/24 `
  --tags owner=tarun
```

Create subnet 2:

```powershell
az network vnet subnet create `
  --resource-group az-nsg-lab-rg `
  --vnet-name az-nsg-lab-net `
  --name az-nsg-lab-subnet2 `
  --address-prefixes 10.0.2.0/24
```

Verify:

```powershell
az network vnet subnet list `
  --resource-group az-nsg-lab-rg `
  --vnet-name az-nsg-lab-net `
  --query "[].{Subnet:name,Prefix:addressPrefix}" `
  -o table
```

Expected:

```text
Subnet                  Prefix
----------------------  ------------
az-nsg-lab-subnet1      10.0.1.0/24
az-nsg-lab-subnet2      10.0.2.0/24
```

---

# 3. Create the NSG

Create the NSG that will eventually be associated with VM2's NIC:

```powershell
az network nsg create `
  --resource-group az-nsg-lab-rg `
  --name az-nsg-lab-vm2NSG `
  --location centralindia `
  --tags owner=tarun
```

Inspect the default rules:

```powershell
az network nsg rule list `
  -g az-nsg-lab-rg `
  --nsg-name az-nsg-lab-vm2NSG `
  --query "[].{Name:name,Priority:priority,Direction:direction,Access:access,Protocol:protocol,Source:sourceAddressPrefix,Port:destinationPortRange}" `
  -o table
```

Azure provides default rules including:

- Allow traffic from `VirtualNetwork`
- Allow traffic from `AzureLoadBalancer`
- Deny all other inbound traffic
- Allow traffic to `VirtualNetwork`
- Allow traffic to `Internet`
- Deny all other outbound traffic

Custom rules can override these defaults by using a higher priority.

---

# 4. Create the Virtual Machines

VM size used in this lab:

```text
Standard_D2s_v5
```

Create VM1:

```powershell
az vm create `
  --resource-group az-nsg-lab-rg `
  --name az-nsg-lab-vm1 `
  --location centralindia `
  --image Ubuntu2204 `
  --size Standard_D2s_v5 `
  --vnet-name az-nsg-lab-net `
  --subnet az-nsg-lab-subnet1 `
  --admin-username azureuser `
  --generate-ssh-keys `
  --public-ip-sku Standard `
  --tags owner=tarun
```

Create VM2:

```powershell
az vm create `
  --resource-group az-nsg-lab-rg `
  --name az-nsg-lab-vm2 `
  --location centralindia `
  --image Ubuntu2204 `
  --size Standard_D2s_v5 `
  --vnet-name az-nsg-lab-net `
  --subnet az-nsg-lab-subnet2 `
  --admin-username azureuser `
  --generate-ssh-keys `
  --public-ip-sku Standard `
  --tags owner=tarun
```

> `Standard_B2s` was unavailable in Central India during the lab, so `Standard_D2s_v5` was used instead.

---

# 5. Understand the NIC-Level NSG

VM creation automatically created a NIC-level NSG allowing SSH.

Check VM2's NIC:

```powershell
az network nic show `
  -g az-nsg-lab-rg `
  -n az-nsg-lab-vm2VMNic `
  --query "{NIC:name,NSG:networkSecurityGroup.id}" `
  -o table
```

The NIC is associated with:

```text
az-nsg-lab-vm2NSG
```

The default SSH rule was:

```text
default-allow-ssh
Priority: 1000
Direction: Inbound
Access: Allow
Protocol: TCP
Source: *
Destination port: 22
```

This explains why VM2 initially accepted SSH connections.

---

# 6. Allow SSH from Subnet 1

Create a custom inbound rule:

```powershell
az network nsg rule create `
  -g az-nsg-lab-rg `
  --nsg-name az-nsg-lab-vm2NSG `
  --name allow-ssh-from-subnet1 `
  --priority 100 `
  --direction Inbound `
  --access Allow `
  --protocol Tcp `
  --source-address-prefixes 10.0.1.0/24 `
  --destination-address-prefixes 10.0.2.0/24 `
  --destination-port-ranges 22
```

Verify:

```powershell
az network nsg rule list `
  -g az-nsg-lab-rg `
  --nsg-name az-nsg-lab-vm2NSG `
  --query "[].{Name:name,Priority:priority,Direction:direction,Access:access,Protocol:protocol,Source:sourceAddressPrefix,Destination:destinationAddressPrefix,Port:destinationPortRange}" `
  -o table
```

The rule means:

> Allow TCP/22 traffic originating from subnet 1 and destined for VM2's subnet.

Test from VM1:

```powershell
az vm run-command invoke `
  -g az-nsg-lab-rg `
  -n az-nsg-lab-vm1 `
  --command-id RunShellScript `
  --scripts "timeout 5 bash -c '</dev/tcp/10.0.2.4/22' && echo SSH_PORT_OPEN || echo SSH_PORT_BLOCKED"
```

Expected:

```text
SSH_PORT_OPEN
```

---

# 7. Allow SSH from a Specific Public IP

The laptop's public IP during the lab was:

```text
103.79.249.47
```

A `/32` CIDR represents a single IPv4 address.

Create an allow rule:

```powershell
az network nsg rule create `
  -g az-nsg-lab-rg `
  --nsg-name az-nsg-lab-vm2NSG `
  --name allow-ssh-from-my-ip `
  --priority 110 `
  --direction Inbound `
  --access Allow `
  --protocol Tcp `
  --source-address-prefixes 103.79.249.47/32 `
  --destination-address-prefixes "*" `
  --destination-port-ranges 22
```

Test from the laptop:

```powershell
Test-NetConnection 40.80.87.71 -Port 22
```

Expected:

```text
TcpTestSucceeded : True
```

### NAT observation

The laptop had a private local address:

```text
192.168.8.4
```

Azure saw the public source address:

```text
103.79.249.47
```

The traffic therefore looked conceptually like:

```text
Laptop
192.168.8.4
     │
     │ NAT
     ▼
103.79.249.47
     │
     ▼
Azure VM2
```

Therefore, the NSG rule must use the source IP visible to Azure.

---

# 8. Demonstrate Explicit Deny

Create a higher-priority deny rule:

```powershell
az network nsg rule create `
  -g az-nsg-lab-rg `
  --nsg-name az-nsg-lab-vm2NSG `
  --name deny-ssh `
  --priority 100 `
  --direction Inbound `
  --access Deny `
  --protocol Tcp `
  --source-address-prefixes "*" `
  --destination-address-prefixes "*" `
  --destination-port-ranges 22
```

Because priority `100` is evaluated before priority `110` and `1000`, the deny rule wins.

Test:

```powershell
Test-NetConnection 40.80.87.71 -Port 22
```

Expected:

```text
TcpTestSucceeded : False
```

This demonstrates:

> Lower numerical priority wins when multiple rules match the same traffic.

---

# 9. Demonstrate Outbound Filtering

First establish that VM2 can communicate with VM1:

```powershell
az vm run-command invoke `
  -g az-nsg-lab-rg `
  -n az-nsg-lab-vm2 `
  --command-id RunShellScript `
  --scripts "ping -c 4 10.0.1.4"
```

Expected:

```text
4 packets transmitted
4 received
0% packet loss
```

Create an outbound deny rule:

```powershell
az network nsg rule create `
  -g az-nsg-lab-rg `
  --nsg-name az-nsg-lab-vm2NSG `
  --name deny-outbound-to-subnet1 `
  --priority 200 `
  --direction Outbound `
  --access Deny `
  --protocol "*" `
  --source-address-prefixes 10.0.2.0/24 `
  --destination-address-prefixes 10.0.1.0/24 `
  --destination-port-ranges "*"
```

Test again:

```powershell
az vm run-command invoke `
  -g az-nsg-lab-rg `
  -n az-nsg-lab-vm2 `
  --command-id RunShellScript `
  --scripts "ping -c 4 10.0.1.4"
```

Expected:

```text
4 packets transmitted
0 received
100% packet loss
```

This demonstrates that NSGs control both directions.

---

# 10. Inbound vs Outbound Priority

Inbound and outbound rule priorities are independent.

For example, an NSG can contain:

```text
Inbound:
  Priority 100 → Allow

Outbound:
  Priority 100 → Deny
```

This is valid.

Priority conflicts matter between rules in the **same NSG and same direction**.

---

# 11. Two-Sided Traffic Evaluation

A key troubleshooting concept demonstrated by this lab is that network connectivity can be blocked at either side.

For traffic:

```text
VM1 ─────────────────────> VM2
```

think about two separate decisions:

```text
VM1
 │
 │ Outbound NSG
 │
 ▼
Network
 │
 │ Inbound NSG
 ▼
VM2
```

For a connection to succeed:

```text
Source outbound → ALLOW
Destination inbound → ALLOW
```

If either side denies the traffic, the connection fails.

For example:

```text
VM1 outbound NSG
       ALLOW
         │
         ▼
       packet
         │
         ▼
VM2 inbound NSG
       DENY
         X
```

The packet can leave VM1 but is still rejected at VM2.

This is an important troubleshooting model:

> **Always check both the source-side outbound path and destination-side inbound path.**

---

# 12. Subnet NSG vs NIC NSG

An NSG can be associated with:

- A subnet
- A network interface

They are not hierarchical in the sense that one simply "overrides" the other.

When both apply, the traffic must satisfy the effective filtering imposed by both.

Conceptually:

```text
             VM
              │
          ┌───┴───┐
          │       │
       Subnet     NIC
         NSG      NSG
          │       │
          └───┬───┘
              │
          VM traffic
```

Therefore, an allow rule in one NSG does not override a deny rule in another NSG.

---

# Key Concepts Learned

### NSG

An NSG is a stateful network traffic filter used to control traffic to and from Azure resources.

### Priority

Lower numerical values have higher priority.

```text
100  → evaluated before 200
200  → evaluated before 1000
```

### `/32`

Represents a single IPv4 address.

```text
103.79.249.47/32
```

means only:

```text
103.79.249.47
```

### Inbound

Traffic entering the resource.

```text
Internet / VM1 → VM2
```

### Outbound

Traffic leaving the resource.

```text
VM2 → VM1 / Internet
```

### Default Rules

Azure NSGs contain default rules that provide baseline allow/deny behaviour.

Custom rules with higher priority can override applicable default rules.

### Source IP

The source address seen by Azure may differ from the private address of the originating machine because of NAT.

---

# Cleanup

The lab environment is configured for automatic deletion.

If manual cleanup is required:

```powershell
az group delete `
  --name az-nsg-lab-rg `
  --yes `
  --no-wait
```

---

# Lab Status

**Completed**

This lab intentionally focuses on foundational NSG behaviour.

Advanced topics such as:

- Service Tags
- Application Security Groups
- NSG flow logs
- Azure Firewall
- Advanced network security architectures

will be covered separately rather than added to this foundational lab.