# 01 — Azure Networking Foundation

**Status:** 🟢 Completed  
**Location:** `centralindia`  
**Tools:** Azure CLI, PowerShell, Linux

This lab establishes the basic networking foundation in Azure and
demonstrates how Network Security Groups control traffic between
subnets.

The lab was intentionally performed as a hands-on exercise, including
failed commands, configuration changes, traffic tests, and
troubleshooting.

---

## 🎯 Objectives

By completing this lab, we learned how to:

- Create and inspect Azure Resource Groups
- Create an Azure Virtual Network
- Design and create subnets
- Understand CIDR and address spaces
- Create Network Security Groups
- Understand default NSG security rules
- Create custom NSG rules
- Associate NSGs with subnets
- Understand inbound and outbound traffic filtering
- Understand NSG rule priorities
- Test connectivity between Azure VMs
- Use Azure VM Run Command for remote Linux testing
- Inspect Azure resources using Azure CLI and JMESPath

---

## 🏗️ Architecture

The final lab environment consists of one VNet with two subnets.

```text
Azure Subscription
│
└── Resource Group
    │
    └── az-net-lab-vnet
        │
        │ 10.0.0.0/16
        │
        ├── az-net-lab-subnet
        │   │
        │   ├── 10.0.1.0/24
        │   ├── NSG: az-net-lab-nsg1
        │   └── VM1: 10.0.1.4
        │
        └── az-net-lab-subnet2
            │
            ├── 10.0.2.0/24
            ├── NSG: az-net-lab-nsg2
            └── VM2: 10.0.2.4
```

---

## 📦 Resources

| Resource | Name | Configuration |
|---|---|---|
| Resource Group | `az-net-lab-rg` | `centralindia` |
| Virtual Network | `az-net-lab-vnet` | `10.0.0.0/16` |
| Subnet 1 | `az-net-lab-subnet` | `10.0.1.0/24` |
| Subnet 2 | `az-net-lab-subnet2` | `10.0.2.0/24` |
| NSG 1 | `az-net-lab-nsg1` | Associated with Subnet 1 |
| NSG 2 | `az-net-lab-nsg2` | Associated with Subnet 2 |
| VM 1 | `az-net-lab-vm1` | `10.0.1.4` |
| VM 2 | `az-net-lab-vm2` | `10.0.2.4` |

---

## 🌐 VNet and Subnet Design

The VNet uses:

```text
10.0.0.0/16
```

This provides the overall address space for the virtual network.

Two `/24` subnets were created inside that address space:

```text
10.0.1.0/24
10.0.2.0/24
```

A subnet must be contained within the VNet address space.

### Troubleshooting Example

The initial subnet creation attempt used:

```text
10.1.0.0/24
```

Azure rejected the configuration because this range is outside:

```text
10.0.0.0/16
```

The subnet was corrected to:

```text
10.0.1.0/24
```

This was the first practical demonstration of Azure validating
subnet boundaries against the VNet address space.

---

# 🔐 Network Security Groups

Two NSGs were created:

```text
az-net-lab-nsg1
az-net-lab-nsg2
```

They were associated with the two subnets:

```text
az-net-lab-subnet
        │
        └── az-net-lab-nsg1

az-net-lab-subnet2
        │
        └── az-net-lab-nsg2
```

---

## Default NSG Rules

A newly created NSG contains default security rules.

### Inbound

| Priority | Rule | Access |
|---:|---|---|
| 65000 | `AllowVnetInBound` | Allow |
| 65001 | `AllowAzureLoadBalancerInBound` | Allow |
| 65500 | `DenyAllInBound` | Deny |

### Outbound

| Priority | Rule | Access |
|---:|---|---|
| 65000 | `AllowVnetOutBound` | Allow |
| 65001 | `AllowInternetOutBound` | Allow |
| 65500 | `DenyAllOutBound` | Deny |

Custom rules are separate from these built-in defaults.

Azure CLI exposes them separately as:

```text
securityRules
defaultSecurityRules
```

---

# 🧪 NSG Traffic Experiment

The main experiment was designed to demonstrate traffic filtering
between the two subnets.

```text
VM1
10.0.1.4
   │
   │ TCP/443
   ▼
VM2
10.0.2.4
```

A listener was started on TCP/443 on VM2.

Connectivity was then tested from VM1 using:

```bash
timeout 5 nc -vz 10.0.2.4 443
```

The connection succeeded when the NSG configuration allowed the traffic.

---

## 🔢 NSG Rule Priority

NSG custom rule priorities range from:

```text
100 - 4096
```

Lower numerical values are evaluated first.

For example:

```text
Priority 100 → Allow
Priority 300 → Deny
```

The priority `100` rule is evaluated before the priority `300` rule.

### Priority Experiment

The lab deliberately attempted several priority changes.

Attempting a priority below `100` failed because:

```text
SecurityRuleInvalidPriority
```

Attempting to assign an already-used priority resulted in:

```text
SecurityRuleConflict
```

This demonstrated that custom NSG rules:

1. Must use priorities between `100` and `4096`.
2. Cannot share the same priority when the rules have the same
   direction.

---

# 🔄 Rule Reordering

The original configuration contained:

```text
Priority 100 → Deny
Priority 200 → Allow
```

The rules were then reordered to:

```text
Priority 100 → Allow
Priority 300 → Deny
```

The connectivity test subsequently succeeded.

This demonstrated the practical effect of NSG rule evaluation order.

---

# ↔️ Direction Matters

The lab also demonstrated the difference between:

```text
Outbound
```

and:

```text
Inbound
```

For traffic:

```text
VM2 → VM1
```

an inbound deny rule on VM1 can block the connection even when VM2's
outbound traffic is otherwise permitted.

An example rule created during the experiment was:

```text
Deny-Subnet2-to-Subnet1-443

Direction:       Inbound
Access:          Deny
Protocol:        TCP
Source:          10.0.2.0/24
Destination:     10.0.1.0/24
Destination Port: 443
Priority:        100
```

The reverse connectivity test from VM2 to VM1 was blocked.

---

# 🖥️ VM Connectivity Testing

Direct SSH access from the local Windows workstation was not used as
the primary testing mechanism because the VM's SSH connectivity was
affected by the NSG configuration.

Instead, Azure VM Run Command was used to execute commands inside the
Linux VMs.

Example:

```powershell
az vm run-command invoke `
  -g az-net-lab-rg `
  -n az-net-lab-vm1 `
  --command-id RunShellScript `
  --scripts "timeout 5 nc -vz 10.0.2.4 443"
```

This allowed network connectivity to be tested independently of the
local workstation's SSH connection.

---

# 🔎 Resource Inspection

Azure CLI was used extensively to inspect the resulting
configuration.

For example:

```powershell
az network nic list `
  -g az-net-lab-rg `
  --query "[].{NIC:name,PrivateIP:ipConfigurations[0].privateIPAddress,Subnet:ipConfigurations[0].subnet.id,NSG:networkSecurityGroup.id}" `
  -o table
```

This confirmed:

```text
VM1 NIC → 10.0.1.4 → Subnet 1 → az-net-lab-nsg1
VM2 NIC → 10.0.2.4 → Subnet 2
```

The NSG was subsequently associated with VM1's NIC when required for
the traffic experiment.

---

# 🧠 JMESPath

Azure CLI supports JMESPath queries through:

```text
--query
```

For example:

```powershell
az network vnet show `
  --resource-group az-net-lab-rg `
  --name az-net-lab-vnet `
  --query "{Name:name, AddressSpace:addressSpace.addressPrefixes, Subnets:subnets[].{Name:name, Prefix:addressPrefix}}" `
  -o json
```

An important lesson was that the names on the left side of the
projection are user-defined aliases.

For example:

```text
{Owner:tags.owner}
```

means:

```text
Owner      → output name
tags.owner → Azure property
```

---

# 🛠️ Troubleshooting Lessons

Several failed operations were intentionally retained as part of the
learning process.

### Invalid subnet range

```text
10.1.0.0/24
```

was outside:

```text
10.0.0.0/16
```

### Invalid CLI argument

A typo in:

```text
--source-address-prefixes
```

was rejected by Azure CLI.

### Invalid priority

A priority below `100` was rejected.

### Priority conflict

Two rules could not use the same priority in the same direction.

### Rule rename

Updating an NSG rule does not rename it.

The practical approach is:

```text
Delete old rule
       ↓
Create new rule
```

---

# 📋 Final NSG Configuration

## NSG 1 — `az-net-lab-nsg1`

| Priority | Name | Direction | Access | Source | Destination | Port |
|---:|---|---|---|---|---|---:|
| 100 | `Allow-Subnet1-to-Subnet2-443` | Outbound | Allow | `10.0.1.0/24` | `10.0.2.0/24` | 443 |
| 300 | `Deny-Subnet1-to-Subnet2-443` | Outbound | Deny | `10.0.1.0/24` | `10.0.2.0/24` | 443 |

## NSG 2 — `az-net-lab-nsg2`

| Priority | Name | Direction | Access | Source | Destination | Port |
|---:|---|---|---|---|---|---:|
| 100 | `Allow-Subnet2-443` | Inbound | Allow | `10.0.1.0/24` | `10.0.2.0/24` | 443 |

---

# 💡 Key Takeaways

1. A subnet must be contained within its VNet address space.
2. A VNet can contain multiple subnets.
3. NSGs can be associated with subnets or network interfaces.
4. NSGs contain both default and custom security rules.
5. Custom rules are evaluated by priority.
6. Lower numerical priority is evaluated first.
7. Custom NSG priorities range from `100` to `4096`.
8. Rules cannot share a priority when their direction conflicts.
9. Inbound and outbound filtering are separate considerations.
10. Connectivity testing should identify which direction of traffic is
    being evaluated.
11. Azure VM Run Command is useful for testing Linux workloads without
    relying on SSH.
12. Azure CLI's `--query` option is useful for extracting specific
    resource properties.
13. Azure CLI command groups are hierarchical.
14. JSON output is useful when inspecting complete resource objects.
15. Failed operations are valuable troubleshooting exercises.

---

# 📝 Session Log

The chronological record of commands, results, failures, and
observations is preserved separately:

[`session.log`](./session.log)

The session log is intentionally kept as a raw chronological record,
while this README presents the lab in a structured form.

---

# 🧹 Cleanup

When the lab is no longer required, remove the resource group:

```powershell
az group delete `
  --name az-net-lab-rg `
  --yes
```

This removes the resources created as part of the lab.

> **Note:** Azure resources can incur charges. Verify the resource
> group and its contents before deleting anything.

---

## 🔜 Next Lab

The next stage will build on this networking foundation and explore
Azure routing and traffic paths.
```

### One correction from our earlier structure

I deliberately changed one thing from the earlier draft: **NSGs belong in Lab 01**, not Lab 02. That's much more accurate given what we actually did.

And I would keep `session.log` **raw and chronological**. The README above is the polished explanation; the log is our evidence trail.

So the directory becomes:

```text
azure-networking-lab/
│
├── README.md
│
└── 01-network-foundation/
    ├── README.md
    └── session.log
```