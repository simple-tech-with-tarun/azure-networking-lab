# Azure Network Watcher Lab

Hands-on Azure networking lab demonstrating **Azure Network Watcher** for network diagnostics, routing analysis, NSG verification, end-to-end connectivity troubleshooting, and network topology discovery.

---

## Overview

This lab uses Azure Network Watcher to investigate network connectivity between two Azure virtual machines inside the same Virtual Network.

The lab demonstrates how different Network Watcher capabilities answer different networking questions:

- **Next Hop** — Which route does Azure use?
- **Effective NSG** — Which security rules are actually applied?
- **IP Flow Verify** — Would a specific traffic flow be allowed or denied?
- **Connection Troubleshoot** — Can the connection actually be established?
- **Topology** — How does Azure see the relationships between networking resources?

The lab also demonstrates the difference between **network configuration** and **actual end-to-end application connectivity**.

---

## Lab Objectives

By completing this lab, you will:

- Understand the purpose and role of **Azure Network Watcher** in network diagnostics.
- Enable and configure Network Watcher for an Azure region.
- Build a simple multi-subnet Azure Virtual Network for troubleshooting exercises.
- Use **Next Hop** to analyze Azure routing decisions.
- Configure and inspect **Network Security Groups (NSGs)** and their effective rules.
- Use **IP Flow Verify** to determine whether a specific TCP flow is allowed or denied and identify the controlling NSG rule.
- Understand the distinction between **ICMP connectivity testing** and the protocols supported by Azure CLI IP Flow Verify.
- Use **Connection Troubleshoot** to diagnose end-to-end connectivity between Azure virtual machines.
- Understand when the **Network Watcher Agent** is required for VM-based diagnostics.
- Distinguish between a network path being available and an application actually listening on the destination port.
- Use **Network Topology** to visualize relationships between VNets, subnets, NICs, VMs, and NSGs.
- Develop a structured Azure network troubleshooting workflow using multiple Network Watcher capabilities.

---

## Architecture

```text
                         Azure Network Watcher
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
           Next Hop          IP Flow Verify    Connection Troubleshoot
              │                   │                   │
              └───────────────────┼───────────────────┘
                                  │
                       az-network-watcher-vnet
                          10.140.0.0/16
                                  │
                  ┌───────────────┴───────────────┐
                  │                               │
             subnet-a                        subnet-b
          10.140.1.0/24                     10.140.2.0/24
                  │                               │
                  │                               │
        az-network-watcher-vm1          az-network-watcher-vm2
             10.140.1.4                     10.140.2.4
                                                  │
                                         az-network-watcher-vm2-nsg
                                                  │
                                                  │
                  └────────── VirtualNetwork ─────┘
```

### Resource Summary

| Resource | Configuration |
|---|---|
| Resource Group | `az-network-watcher-lab-rg` |
| Location | `centralindia` |
| VNet | `az-network-watcher-vnet` |
| VNet CIDR | `10.140.0.0/16` |
| Subnet A | `10.140.1.0/24` |
| Subnet B | `10.140.2.0/24` |
| VM1 | `az-network-watcher-vm1` |
| VM1 Private IP | `10.140.1.4` |
| VM2 | `az-network-watcher-vm2` |
| VM2 Private IP | `10.140.2.4` |
| VM Size | `Standard_D2s_v5` |
| NSG | `az-network-watcher-vm2-nsg` |
| NSG Association | `subnet-b` |
| Network Watcher | `NetworkWatcher_centralindia` |

Both VMs were deployed without public IP addresses.

---

# 1. Enable Network Watcher

Network Watcher was enabled in the `centralindia` region.

Because the subscription requires tags on resource groups, the Network Watcher resource group was explicitly created first.

```powershell
az group create `
  --name NetworkWatcherRG `
  --location centralindia `
  --tags owner=tarun AutoDelete=Yes
```

Enable Network Watcher:

```powershell
az network watcher configure `
  --resource-group NetworkWatcherRG `
  --locations centralindia `
  --enabled true
```

Verify:

```powershell
az network watcher list `
  --output table
```

The resulting Network Watcher was:

```text
NetworkWatcher_centralindia
```

---

# 2. Create the Lab Resource Group

```powershell
az group create `
  --name az-network-watcher-lab-rg `
  --location centralindia `
  --tags owner=tarun AutoDelete=Yes
```

---

# 3. Create the Virtual Network

Create the VNet and first subnet:

```powershell
az network vnet create `
  --resource-group az-network-watcher-lab-rg `
  --name az-network-watcher-vnet `
  --location centralindia `
  --address-prefix 10.140.0.0/16 `
  --subnet-name subnet-a `
  --subnet-prefix 10.140.1.0/24
```

Create the second subnet:

```powershell
az network vnet subnet create `
  --resource-group az-network-watcher-lab-rg `
  --vnet-name az-network-watcher-vnet `
  --name subnet-b `
  --address-prefix 10.140.2.0/24
```

Resulting network:

```text
VNet:       10.140.0.0/16
│
├── subnet-a
│   └── 10.140.1.0/24
│
└── subnet-b
    └── 10.140.2.0/24
```

---

# 4. Deploy the Virtual Machines

Two Ubuntu 22.04 virtual machines were deployed:

```text
VM1 → subnet-a → 10.140.1.4
VM2 → subnet-b → 10.140.2.4
```

Both VMs use:

```text
Standard_D2s_v5
```

Both VMs were deployed without public IP addresses.

Azure VM Run Command was used for VM-side testing and diagnostics.

---

# 5. Baseline Connectivity

Before introducing an NSG restriction, connectivity from VM1 to VM2 was tested using ICMP.

```powershell
az vm run-command invoke `
  --resource-group az-network-watcher-lab-rg `
  --name az-network-watcher-vm1 `
  --command-id RunShellScript `
  --scripts "ping -c 4 10.140.2.4"
```

Result:

```text
4 packets transmitted
4 packets received
0% packet loss
```

This established baseline connectivity between the two subnets.

---

# 6. Network Watcher Next Hop

Network Watcher can determine which next hop Azure uses for a specific source and destination.

```powershell
az network watcher show-next-hop `
  --resource-group az-network-watcher-lab-rg `
  --vm az-network-watcher-vm1 `
  --source-ip 10.140.1.4 `
  --dest-ip 10.140.2.4
```

Result:

```json
{
  "nextHopType": "VirtualNetwork",
  "routeTableId": "System Route"
}
```

### Interpretation

Azure is routing the traffic directly through the Virtual Network using the system route.

No UDR, firewall, VPN gateway, or other network virtual appliance is involved in this path.

```text
10.140.1.4
     │
     │ System Route
     ▼
VirtualNetwork
     │
     ▼
10.140.2.4
```

---

# 7. Create a Network Security Group

An NSG was created for VM2's subnet:

```powershell
az network nsg create `
  --resource-group az-network-watcher-lab-rg `
  --name az-network-watcher-vm2-nsg `
  --location centralindia `
  --tags owner=tarun AutoDelete=Yes
```

The NSG was associated with `subnet-b`:

```powershell
az network vnet subnet update `
  --resource-group az-network-watcher-lab-rg `
  --vnet-name az-network-watcher-vnet `
  --name subnet-b `
  --network-security-group az-network-watcher-vm2-nsg
```

The resulting relationship is:

```text
az-network-watcher-vm2-nsg
          │
          ▼
      subnet-b
          │
          ▼
     VM2 NIC
          │
          ▼
         VM2
```

---

# 8. Inspect Effective NSG Rules

The effective NSG configuration for VM2 was inspected through its NIC:

```powershell
az network nic list-effective-nsg `
  --resource-group az-network-watcher-lab-rg `
  --name az-network-watcher-vm2-nic `
  --output json
```

The output confirmed that the NSG associated with `subnet-b` was effective on VM2.

This demonstrates an important Azure networking concept:

> An NSG associated with a subnet affects the network interfaces and resources connected to that subnet.

---

# 9. ICMP NSG Experiment

An ICMP deny rule was temporarily created:

```powershell
az network nsg rule create `
  --resource-group az-network-watcher-lab-rg `
  --nsg-name az-network-watcher-vm2-nsg `
  --name Deny-ICMP-From-Subnet-A `
  --priority 100 `
  --direction Inbound `
  --access Deny `
  --protocol Icmp `
  --source-address-prefixes 10.140.1.0/24 `
  --destination-address-prefixes 10.140.2.0/24 `
  --destination-port-ranges "*"
```

The result was:

```text
VM1 → VM2
ICMP
100% packet loss
```

This demonstrated that an NSG can explicitly deny ICMP traffic.

### IP Flow Verify Protocol Limitation

The Azure CLI command:

```text
az network watcher test-ip-flow
```

does **not** accept `Icmp` as a value for its `--protocol` argument.

For this lab, IP Flow Verify was therefore demonstrated using TCP.

This is a limitation of the **Azure CLI IP Flow Verify command**, not a limitation of Network Watcher as a whole or of Azure NSGs' ability to filter ICMP traffic.

The temporary ICMP rule was removed before continuing:

```powershell
az network nsg rule delete `
  --resource-group az-network-watcher-lab-rg `
  --nsg-name az-network-watcher-vm2-nsg `
  --name Deny-ICMP-From-Subnet-A
```

---

# 10. TCP/80 NSG Deny

A TCP deny rule was created to demonstrate IP Flow Verify:

```powershell
az network nsg rule create `
  --resource-group az-network-watcher-lab-rg `
  --nsg-name az-network-watcher-vm2-nsg `
  --name Deny-TCP-From-Subnet-A `
  --priority 100 `
  --direction Inbound `
  --access Deny `
  --protocol Tcp `
  --source-address-prefixes 10.140.1.0/24 `
  --destination-address-prefixes 10.140.2.0/24 `
  --destination-port-ranges 80
```

The rule blocks:

```text
Source:      10.140.1.0/24
Destination: 10.140.2.0/24
Protocol:    TCP
Port:        80
Direction:   Inbound
Priority:    100
```

---

# 11. Verify TCP/80 Is Blocked

The connection was tested directly from VM1:

```powershell
az vm run-command invoke `
  --resource-group az-network-watcher-lab-rg `
  --name az-network-watcher-vm1 `
  --command-id RunShellScript `
  --scripts "timeout 5 bash -c '</dev/tcp/10.140.2.4/80' && echo 'TCP 80 OPEN' || echo 'TCP 80 BLOCKED'"
```

Result:

```text
TCP 80 BLOCKED
```

---

# 12. IP Flow Verify — Denied

Network Watcher was then used to identify the exact NSG rule responsible for the blocked flow.

Because the NSG was associated with VM2's subnet, the test was performed from VM2's inbound perspective:

```powershell
az network watcher test-ip-flow `
  --resource-group az-network-watcher-lab-rg `
  --vm az-network-watcher-vm2 `
  --direction Inbound `
  --protocol TCP `
  --local 10.140.2.4:80 `
  --remote 10.140.1.4:50000
```

Result:

```json
{
  "access": "Deny",
  "ruleName": "securityRules/Deny-TCP-From-Subnet-A"
}
```

### Key Takeaway

IP Flow Verify can identify the **specific NSG rule** responsible for allowing or denying a particular flow.

---

# 13. Remove the TCP Deny Rule

```powershell
az network nsg rule delete `
  --resource-group az-network-watcher-lab-rg `
  --nsg-name az-network-watcher-vm2-nsg `
  --name Deny-TCP-From-Subnet-A
```

Run IP Flow Verify again:

```powershell
az network watcher test-ip-flow `
  --resource-group az-network-watcher-lab-rg `
  --vm az-network-watcher-vm2 `
  --direction Inbound `
  --protocol TCP `
  --local 10.140.2.4:80 `
  --remote 10.140.1.4:50000
```

Result:

```json
{
  "access": "Allow",
  "ruleName": "defaultSecurityRules/AllowVnetInBound"
}
```

This demonstrates the transition:

```text
Explicit Deny Rule
       │
       ▼
     Deny
```

followed by:

```text
Explicit Deny Removed
       │
       ▼
AllowVnetInBound
       │
       ▼
     Allow
```

---

# 14. Connection Troubleshoot

Network Watcher's Connection Troubleshoot capability was used to test actual connectivity between VM1 and VM2.

The initial test:

```powershell
az network watcher test-connectivity `
  --resource-group az-network-watcher-lab-rg `
  --source-resource az-network-watcher-vm1 `
  --dest-resource az-network-watcher-vm2 `
  --protocol TCP `
  --dest-port 80
```

initially returned:

```text
connectionStatus: Unreachable
```

The result included a network security rule issue.

However, VM2 was not actually listening on TCP/80 at that point.

This highlighted an important distinction:

> An NSG allowing a traffic flow does not guarantee that an application is listening on the destination port.

---

# 15. Network Watcher Agent

Connection Troubleshoot requires VM-side participation through the Network Watcher Agent extension.

The agent was installed on VM1:

```powershell
az vm extension set `
  --resource-group az-network-watcher-lab-rg `
  --vm-name az-network-watcher-vm1 `
  --name NetworkWatcherAgentLinux `
  --publisher Microsoft.Azure.NetworkWatcher `
  --version 1.4
```

Result:

```text
provisioningState: Succeeded
```

### Important Distinction

The Network Watcher Agent is **not installed on every Azure resource**.

Network Watcher has multiple capabilities that operate against Azure networking configuration without requiring an agent on each resource, including:

- Next Hop
- Effective NSG
- IP Flow Verify
- Topology

Certain VM-level diagnostics, such as Connection Troubleshoot, can require VM-side participation.

---

# 16. Verify the Destination Application

VM2 was checked for a TCP/80 listener:

```powershell
az vm run-command invoke `
  --resource-group az-network-watcher-lab-rg `
  --name az-network-watcher-vm2 `
  --command-id RunShellScript `
  --scripts "ss -lntp | grep ':80 ' || echo 'NOT LISTENING ON TCP 80'"
```

Initial result:

```text
NOT LISTENING ON TCP 80
```

A temporary Python HTTP server was started:

```powershell
az vm run-command invoke `
  --resource-group az-network-watcher-lab-rg `
  --name az-network-watcher-vm2 `
  --command-id RunShellScript `
  --scripts "nohup python3 -m http.server 80 --bind 0.0.0.0 >/tmp/http-server.log 2>&1 &"
```

The listener was verified:

```text
LISTEN 0 5 0.0.0.0:80
users:(("python3",pid=1849,fd=3))
```

VM2 was now accepting TCP connections on port 80.

---

# 17. Successful Connection Troubleshoot

The connectivity test was repeated:

```powershell
az network watcher test-connectivity `
  --resource-group az-network-watcher-lab-rg `
  --source-resource az-network-watcher-vm1 `
  --dest-resource az-network-watcher-vm2 `
  --protocol TCP `
  --dest-port 80
```

Result:

```json
{
  "avgLatencyInMs": 3,
  "connectionStatus": "Reachable",
  "probesFailed": 0,
  "probesSent": 66
}
```

Latency measurements:

```text
Minimum: 1 ms
Average: 3 ms
Maximum: 13 ms
```

The discovered path was:

```text
VM1
10.140.1.4
   │
   │ VirtualNetwork
   ▼
VM2
10.140.2.4
```

### Key Takeaway

Connection Troubleshoot validates actual connectivity rather than only examining static network configuration.

It can help identify problems involving the network path and connectivity between Azure resources.

---

# 18. Network Topology

Network Watcher can discover the relationships between resources in a resource group.

```powershell
az network watcher show-topology `
  --resource-group az-network-watcher-lab-rg `
  --location centralindia `
  --output json
```

Azure discovered the following topology:

```text
az-network-watcher-vnet
│
├── subnet-a
│    └── az-network-watcher-vm1-nic
│         └── az-network-watcher-vm1
│
└── subnet-b
     ├── az-network-watcher-vm2-nic
     │    └── az-network-watcher-vm2
     │
     └── az-network-watcher-vm2-nsg
```

The topology confirmed:

```text
VM1
 └── VM1 NIC
      └── subnet-a

VM2
 └── VM2 NIC
      └── subnet-b
           └── VM2 NSG

subnet-a
 └── az-network-watcher-vnet

subnet-b
 └── az-network-watcher-vnet
```

### Why Topology Matters

Topology discovery becomes increasingly useful as Azure environments grow.

It can help answer questions such as:

- Which subnet contains this NIC?
- Which VM owns this NIC?
- Which NSG is associated with this subnet?
- Which resources belong to this VNet?
- How are the networking resources related?

---

# 19. Network Watcher Diagnostic Workflow

The lab demonstrated several Network Watcher capabilities, each providing a different perspective.

```text
                         Network Watcher
                              │
       ┌──────────────────────┼──────────────────────┐
       │                      │                      │
    Routing                Security             Connectivity
       │                      │                      │
   Next Hop             Effective NSG       Connection Troubleshoot
                              │
                       IP Flow Verify
                              │
                           Topology
```

A practical troubleshooting workflow can therefore be:

```text
Connectivity Problem
        │
        ▼
     Topology
        │
        ▼
     Next Hop
        │
        ▼
   Effective NSG
        │
        ▼
  IP Flow Verify
        │
        ▼
Connection Troubleshoot
        │
        ▼
Application / Service Check
```

---

# Key DevOps / Networking Takeaways

## 1. Network reachability and application reachability are different

A route can exist and an NSG can allow traffic while the application is still unavailable.

```text
Route exists
     +
NSG allows
     +
Application listening
     =
Successful connection
```

---

## 2. NSGs can be associated with subnets

An NSG does not have to be attached directly to a VM NIC.

In this lab:

```text
NSG
 │
 ▼
subnet-b
 │
 ▼
VM2 NIC
```

The NSG therefore becomes effective for VM2.

---

## 3. IP Flow Verify identifies the controlling NSG rule

Instead of manually inspecting every NSG rule, Network Watcher can evaluate a specific flow and identify the rule determining the result.

Example:

```text
TCP
10.140.1.4:50000
        │
        ▼
10.140.2.4:80

Result:
Deny

Rule:
Deny-TCP-From-Subnet-A
```

---

## 4. Next Hop reveals Azure's routing decision

The lab returned:

```text
nextHopType: VirtualNetwork
routeTableId: System Route
```

This confirms that traffic between the two subnets uses Azure's VNet system routing.

---

## 5. Connection Troubleshoot validates actual connectivity

The lab demonstrated:

```text
No service listening
        │
        ▼
Unreachable

Service listening
        │
        ▼
Reachable
66 probes / 0 failures
```

This shows why troubleshooting should consider both **network configuration** and the **application endpoint**.

---

## 6. Network Watcher is a network diagnostics platform

Network Watcher is designed to diagnose and analyze Azure networking rather than act as a generic monitoring agent for every Azure resource.

Some capabilities operate from Azure's network/control-plane information, while others require VM-side participation.

---

# Technologies Used

- Azure Network Watcher
- Azure Virtual Network
- Azure Subnets
- Azure Virtual Machines
- Azure Network Security Groups
- Azure VM Run Command
- Azure VM Extensions
- Azure CLI
- Ubuntu Linux
- Python HTTP Server
- TCP/IP
- ICMP
- Network Routing
- NSG Security Rules
- Network Diagnostics

---

# Azure CLI Commands Covered

```powershell
# Resource Groups
az group create

# Network Watcher
az network watcher configure
az network watcher list
az network watcher show-next-hop
az network watcher test-ip-flow
az network watcher test-connectivity
az network watcher show-topology

# VNet / Subnets
az network vnet create
az network vnet subnet create
az network vnet subnet update

# NSG
az network nsg create
az network nsg rule create
az network nsg rule delete

# Effective NSG
az network nic list-effective-nsg

# Virtual Machines
az vm run-command invoke
az vm extension set
```

---

# Conclusion

This lab demonstrates how Azure Network Watcher can turn a vague connectivity problem into a structured troubleshooting process.

Instead of relying on a single connectivity test, we examined:

```text
Topology
   ↓
Routing
   ↓
Security
   ↓
Flow Evaluation
   ↓
End-to-End Connectivity
   ↓
Application Listener
```

The lab showed how **Next Hop**, **Effective NSG**, **IP Flow Verify**, **Connection Troubleshoot**, and **Topology** complement each other when diagnosing Azure networking issues.

The most important practical lesson is:

> **A successful network configuration does not necessarily mean a successful application connection.**

Effective troubleshooting requires understanding the entire path from **source → routing → security → destination → application**.