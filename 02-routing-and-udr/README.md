# Lab 2 — Azure Routing & User-Defined Routes

## Overview

This lab explores how Azure routes traffic between subnets and how **User-Defined Routes (UDRs)** can be used to direct traffic through an Azure VM acting as a **virtual appliance**.

The lab uses two VMs. Due to the subscription's regional CPU quota, a third VM could not be deployed. VM2 therefore uses two network interfaces and acts as both the **transit device** and the **destination** for the routing demonstration.

---

## Objectives

- Understand Azure's default VNet routing.
- Understand how UDRs override default routing behavior.
- Configure a route using `VirtualAppliance` as the next-hop type.
- Configure an Azure VM to forward network traffic.
- Understand Azure NIC IP forwarding versus Linux IP forwarding.
- Configure a multi-NIC VM.
- Inspect effective routes applied to an Azure NIC.
- Verify traffic flow through a virtual appliance.

---

## Architecture

```text
                         Azure VNet
                       10.0.0.0/16
                            |
          +-----------------+------------------+
          |                 |                  |
          |                 |                  |
     Subnet 1           Subnet 2           Subnet 3
     10.0.1.0/24        10.0.2.0/24        10.0.3.0/24
          |                 |                  |
          |                 |                  |
       VM1               VM2                  |
     10.0.1.4       eth0: 10.0.2.4            |
                       eth1: 10.0.3.4 <-------+
                            |
                       IP Forwarding
```

### Final traffic path

```text
VM1
10.0.1.4
   |
   | Destination: 10.0.3.4
   |
   | UDR:
   | 10.0.3.0/24
   | → VirtualAppliance
   | → 10.0.2.4
   |
   v
VM2 eth0
10.0.2.4
   |
   | Linux IP forwarding
   |
   v
VM2 eth1
10.0.3.4
```

VM2 therefore demonstrates both roles:

- **Virtual appliance / transit router**
- **Destination**

---

## Azure Resources

### Resource Group

```text
az-routing-lab-rg
```

### Virtual Network

```text
az-routing-lab-vnet
Address space: 10.0.0.0/16
```

### Subnets

| Subnet | Address Space |
|---|---|
| `az-routing-lab-subnet1` | `10.0.1.0/24` |
| `az-routing-lab-subnet2` | `10.0.2.0/24` |
| `az-routing-lab-subnet3` | `10.0.3.0/24` |

### Virtual Machines

| VM | Interface | IP | Role |
|---|---|---|---|
| VM1 | Primary NIC | `10.0.1.4` | Source |
| VM2 | `eth0` | `10.0.2.4` | Transit / Virtual Appliance |
| VM2 | `eth1` | `10.0.3.4` | Destination |

---

# 1. Azure Default Routing

Azure automatically provides a VNet-local route for the VNet address space.

For this lab:

```text
10.0.0.0/16 → VnetLocal
```

This allows resources in different subnets of the same VNet to communicate without requiring a custom route.

The effective route table can be inspected with:

```powershell
az network nic show-effective-route-table `
  --resource-group az-routing-lab-rg `
  --name az-routing-lab-vm1VMNic `
  -o table
```

---

# 2. User-Defined Route

A route table was created:

```text
az-routing-lab-rt1
```

The lab route was configured as:

```text
Destination:   10.0.3.0/24
Next Hop Type: VirtualAppliance
Next Hop IP:   10.0.2.4
```

Conceptually:

```text
10.0.3.0/24
      |
      v
VirtualAppliance
      |
      v
10.0.2.4
```

This tells Azure to send traffic destined for Subnet3 to VM2.

---

# 3. Understanding `NextHopType None`

A temporary test route was also used with:

```text
10.0.3.0/24 → None
```

This demonstrated that a UDR can intentionally prevent traffic from reaching a destination.

The effective route appeared as:

```text
User   Active   10.0.3.0/24   None
```

This was a useful demonstration of route precedence and how a custom route can alter Azure's normal VNet-local behavior.

The test route was subsequently removed.

---

# 4. Configuring VM2 as a Virtual Appliance

For Azure to allow VM2 to forward traffic, IP forwarding was enabled on its NIC.

The NIC was configured with:

```text
enableIPForwarding = true
```

Linux IP forwarding was also enabled inside VM2:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Verified with:

```bash
sysctl net.ipv4.ip_forward
```

Expected result:

```text
net.ipv4.ip_forward = 1
```

### Two layers of forwarding

Both layers are important:

```text
Azure NIC
   |
   | IP forwarding enabled
   v
Linux VM
   |
   | net.ipv4.ip_forward = 1
   v
Traffic can be forwarded
```

---

# 5. Multi-NIC VM

Because a third VM could not be deployed within the subscription's regional CPU quota, VM2 was configured with a second NIC.

### VM2 networking

```text
eth0 → 10.0.2.4/24
eth1 → 10.0.3.4/24
```

Inside the VM:

```bash
ip -br addr
```

showed both interfaces.

The routing table included:

```text
10.0.2.0/24 → eth0
10.0.3.0/24 → eth1
```

This allowed VM2 to provide connectivity between the two subnets.

---

# 6. Effective Route Verification

The effective route table on VM1 was inspected after applying the UDR.

The important entry was:

```text
User   Active   10.0.3.0/24   VirtualAppliance   10.0.2.4
```

This is the authoritative Azure-side confirmation that traffic destined for `10.0.3.0/24` is being sent to VM2.

---

# 7. Connectivity Verification

From VM1, the destination was tested:

```bash
ping -c 4 10.0.3.4
```

The test succeeded:

```text
4 packets transmitted
4 received
0% packet loss
```

This confirmed the complete routing path.

---

# Key Concepts

## Default Route vs UDR

Azure provides a default VNet-local route:

```text
10.0.0.0/16 → VnetLocal
```

A UDR can provide a more specific route:

```text
10.0.3.0/24 → VirtualAppliance → 10.0.2.4
```

The more specific route is selected for traffic destined for `10.0.3.0/24`.

---

## Azure IP Forwarding vs Linux IP Forwarding

Configuring a VM as a router requires both:

**Azure:**

```text
NIC → enableIPForwarding = true
```

**Linux:**

```text
net.ipv4.ip_forward = 1
```

These operate at different layers and both are required for this scenario.

---

## Effective Routes

The effective route table shows the routes Azure has actually applied to a NIC.

This is one of the most useful tools for troubleshooting Azure routing problems.

---

# Important Limitation

The original design called for three VMs:

```text
VM1 → VM2 (transit) → VM3 (destination)
```

The subscription's Central India regional core quota was limited to four cores, preventing deployment of the third VM.

The final lab therefore uses:

```text
VM1 → VM2 (transit + destination)
```

This still demonstrates:

- UDR behavior
- Virtual Appliance next-hop routing
- Azure NIC IP forwarding
- Linux IP forwarding
- Multi-NIC routing
- Effective route inspection

A separate transit-to-independent-destination scenario can be explored later when additional compute capacity is available.

---

# Lab Status

**Completed**

### Completed

- [x] Azure VNet and subnet routing
- [x] Default VNet-local routing
- [x] Route table creation
- [x] User-Defined Route
- [x] `NextHopType None`
- [x] `VirtualAppliance` next hop
- [x] Azure NIC IP forwarding
- [x] Linux IP forwarding
- [x] Multi-NIC VM
- [x] Effective route verification
- [x] End-to-end connectivity verification

### Deferred

- [ ] Three-VM transit routing scenario
- [ ] Linux network namespace / veth experiment

---

## Conclusion

This lab demonstrated how Azure's routing system can be extended using User-Defined Routes and how an Azure VM can function as a virtual network appliance.

The most important takeaway is:

```text
UDR
 ↓
Virtual Appliance
 ↓
Azure NIC IP Forwarding
 ↓
Linux IP Forwarding
 ↓
Second Network Interface
 ↓
Destination
```

This provides the foundation for understanding more advanced Azure networking concepts such as network virtual appliances, hub-and-spoke architectures, Azure Firewall, peering, and centralized routing.