# Azure VNet Peering Lab

## Overview

This lab demonstrates **Azure Virtual Network (VNet) Peering**, including bidirectional connectivity, peering access controls, effective routing, and the **non-transitive nature of VNet peering**.

The lab uses a hub-and-spoke topology:

```text
                    Hub VNet
                 10.110.0.0/16
                  /          \
                 /            \
                ↕              ↕
               /                \
        Spoke-1 VNet         Spoke-2 VNet
        10.120.0.0/16       10.130.0.0/16
```

The Hub was peered with both Spoke VNets, while Spoke-1 and Spoke-2 were intentionally **not directly peered**.

---

## Architecture

| VNet | Address Space | Subnet | VM |
|---|---|---|---|
| Hub | `10.110.0.0/16` | `10.110.1.0/24` | Removed during quota management |
| Spoke-1 | `10.120.0.0/16` | `10.120.1.0/24` | `10.120.1.4` |
| Spoke-2 | `10.130.0.0/16` | `10.130.1.0/24` | `10.130.1.4` |

### Peering Relationships

```text
Hub ↔ Spoke-1
Hub ↔ Spoke-2

Spoke-1 ↛ Spoke-2
```

All peering connections were configured with:

- Virtual network access: Enabled
- Forwarded traffic: Disabled
- Gateway transit: Disabled
- Remote gateway: Disabled

---

## What Was Tested

### 1. Hub ↔ Spoke-1 Connectivity

Connectivity was verified in both directions using ICMP.

```text
Hub → Spoke-1     SUCCESS
Spoke-1 → Hub     SUCCESS
```

This confirmed that the peering connection was operational.

---

### 2. VNet Peering Access Control

The Hub-side peering was temporarily configured with:

```text
allowVirtualNetworkAccess = false
```

Although the effective route remained visible, traffic between the VNets was blocked.

After restoring:

```text
allowVirtualNetworkAccess = true
```

connectivity was restored.

### Key Lesson

**Routing information and traffic permission are separate concepts.**

A route can exist in the effective route table while the peering configuration prevents traffic from traversing the connection.

---

## 3. Non-Transitive VNet Peering

The most important test was:

```text
Spoke-1 → Spoke-2
```

The expected path would have been:

```text
Spoke-1 → Hub → Spoke-2
```

However, Azure VNet peering is **non-transitive**.

The test produced:

```text
4 packets transmitted
0 packets received
100% packet loss
```

Therefore:

```text
Spoke-1 ❌→ Spoke-2
```

even though both VNets were independently peered with the Hub.

---

## 4. Effective Route Table Verification

The effective route table of the Spoke-1 NIC showed:

```text
10.120.0.0/16    VnetLocal
10.110.0.0/16    VNetPeering
```

There was **no route for**:

```text
10.130.0.0/16
```

This provided Azure-side confirmation of why Spoke-1 could not reach Spoke-2.

### Important Distinction

The guest operating system's routing table does not necessarily show Azure VNet peering routes.

Azure's **effective route table** represents the routing decisions applied to the NIC by Azure's networking fabric.

---

## Why Doesn't the Hub Automatically Forward Traffic?

VNet peering does not turn the Hub VNet into a router.

Having:

```text
Spoke-1 ↔ Hub
Spoke-2 ↔ Hub
```

does not automatically create:

```text
Spoke-1 ↔ Spoke-2
```

For controlled transit between spokes, an actual forwarding device is required.

For example:

```text
Spoke-1
    |
    | UDR
    ↓
Azure Firewall
    |
    | Forward / inspect
    ↓
Spoke-2
```

A UDR can specify the firewall as the next hop, but the firewall must actually receive, inspect, and forward the traffic.

This is fundamentally different from simply having VNet peering.

---

## Key Takeaways

- VNet peering provides **private connectivity between VNets**.
- Peering is **non-transitive**.
- Two spokes connected to the same Hub cannot communicate through the Hub automatically.
- `allowVirtualNetworkAccess` controls whether traffic can use the peering connection.
- The presence of a route does not necessarily mean traffic is permitted.
- Azure's **effective route table** is useful for troubleshooting Azure-side routing.
- A **UDR determines the next hop**, but a forwarding device such as Azure Firewall or a network virtual appliance must actually forward the traffic.
- Hub-and-spoke transit architectures therefore commonly use **Azure Firewall or another network virtual appliance** when controlled inter-spoke communication is required.

---

## Technologies Used

- Azure Virtual Network
- Azure VNet Peering
- Azure CLI
- Azure VM
- Azure Effective Route Tables
- User Defined Routes (UDR) — conceptual analysis
- Azure Firewall — conceptual transit architecture