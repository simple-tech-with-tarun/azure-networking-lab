# Azure Firewall – Advanced Routing & Traffic Engineering Lab

This lab explores **advanced routing and traffic engineering in Microsoft Azure using Azure Firewall, User Defined Routes (UDRs), VNet peering, private subnets, controlled Internet egress, and stateful traffic inspection**.

The goal was not simply to deploy Azure Firewall, but to understand how Azure selects routes, how UDRs force traffic through a firewall, how private workloads reach the Internet through controlled egress, and how east-west traffic can be inspected between spokes.

---

## 🎯 Objectives

- Understand Azure system routes and User Defined Routes.
- Understand longest-prefix route selection.
- Build a hub-and-spoke network architecture.
- Force spoke-to-spoke traffic through Azure Firewall.
- Configure controlled Internet egress through Azure Firewall.
- Disable implicit/default outbound access for private workloads.
- Understand Azure Firewall SNAT for Internet-bound traffic.
- Configure stateful Firewall rules for east-west traffic.
- Understand why both traffic directions require appropriate routing.
- Troubleshoot routing, Firewall policy, and application-layer failures.
- Inspect effective routes to understand the actual Azure routing decision.

---

# 🏗️ Architecture

```text
                         Internet
                            │
                            │
                    Public IP
                  20.219.254.150
                            │
                    ┌───────▼───────┐
                    │ Azure Firewall │
                    │   10.0.0.4    │
                    └───────┬───────┘
                            │
                         Hub VNet
                         10.0.0.0/16
                            │
              ┌─────────────┴─────────────┐
              │                           │
       VNet Peering                 VNet Peering
              │                           │
              ▼                           ▼
     ┌────────────────┐          ┌────────────────┐
     │    Spoke A     │          │    Spoke B     │
     │   10.1.0.0/16  │          │   10.2.0.0/16  │
     │                │          │                │
     │  VM: 10.1.1.4  │          │  VM: 10.2.1.4  │
     └────────────────┘          └────────────────┘
```

### Important

There was **no direct Spoke A ↔ Spoke B peering**.

Both spokes were connected to the hub, but Azure VNet peering is **non-transitive**.

Therefore, this path does not automatically exist:

```text
Spoke A → Hub → Spoke B
```

Instead, explicit routing through Azure Firewall was required.

---

# ☁️ Resource Layout

## Resource Group

```text
az-firewall-advanced-routing-lab-rg
```

Region:

```text
centralindia
```

---

## Hub VNet

```text
az-firewall-advanced-routing-hub-vnet
CIDR: 10.0.0.0/16
```

Subnet:

```text
AzureFirewallSubnet
10.0.0.0/24
```

Azure Firewall private IP:

```text
10.0.0.4
```

Firewall public IP:

```text
20.219.254.150
```

---

## Spoke A

```text
VNet:
az-firewall-advanced-routing-spoke-a-vnet

CIDR:
10.1.0.0/16

Subnet:
WorkloadSubnet
10.1.1.0/24

VM:
az-firewall-advanced-routing-spoke-a-vm

Private IP:
10.1.1.4
```

---

## Spoke B

```text
VNet:
az-firewall-advanced-routing-spoke-b-vnet

CIDR:
10.2.0.0/16

Subnet:
WorkloadSubnet
10.2.1.0/24

VM:
az-firewall-advanced-routing-spoke-b-vm

Private IP:
10.2.1.4
```

Both workload VMs were deployed **without public IP addresses**.

VM size used:

```text
Standard_D2s_v5
```

---

# 🔗 VNet Peering

Two-way peering was configured between the hub and each spoke:

```text
Hub ↔ Spoke A
Hub ↔ Spoke B
```

There was intentionally **no Spoke A ↔ Spoke B peering**.

The peerings allowed:

```text
Virtual network access
Forwarded traffic
```

The `allowForwardedTraffic` setting was important because Azure Firewall acts as the transit appliance.

However:

> `allowForwardedTraffic` does not turn the hub into a router.

A routing appliance such as Azure Firewall is still required to actually forward the traffic.

---

# 🔥 Azure Firewall

Firewall:

```text
az-firewall-advanced-routing-fw
```

SKU:

```text
AZFW_VNet
```

Tier:

```text
Standard
```

Private IP:

```text
10.0.0.4
```

Public IP:

```text
20.219.254.150
```

Firewall Policy:

```text
az-firewall-advanced-routing-fw-policy
```

Rule Collection Group:

```text
az-firewall-advanced-routing-fw-rcg
```

The rule collection group used priority `100` as its group priority.

---

# 🔀 User Defined Routes

## Spoke A Route Table

```text
az-firewall-advanced-routing-spoke-a-rt
```

Routes:

| Destination | Next Hop |
|---|---|
| `10.2.0.0/16` | Virtual Appliance → `10.0.0.4` |
| `0.0.0.0/0` | Virtual Appliance → `10.0.0.4` |

The first route forced Spoke A → Spoke B traffic through Azure Firewall.

The second route forced Internet-bound traffic through Azure Firewall.

---

## Spoke B Route Table

```text
az-firewall-advanced-routing-spoke-b-rt
```

Routes:

| Destination | Next Hop |
|---|---|
| `10.1.0.0/16` | Virtual Appliance → `10.0.0.4` |
| `0.0.0.0/0` | Virtual Appliance → `10.0.0.4` |

The reverse private route was required to force B → A traffic through the Firewall.

---

# 🧠 Routing Concept

Azure determines the next hop using route selection and longest-prefix matching.

For example, Spoke A had:

```text
10.2.0.0/16 → Firewall
0.0.0.0/0   → Firewall
```

Traffic destined for:

```text
10.2.1.4
```

matches both routes, but:

```text
10.2.0.0/16
```

is more specific than:

```text
0.0.0.0/0
```

Therefore, the private route wins.

This allowed us to distinguish:

```text
Private traffic
10.1.x.x → 10.2.x.x
```

from:

```text
Internet traffic
10.1.x.x → 0.0.0.0/0
```

while using Azure Firewall as the next hop.

---

# 🔐 Firewall Policy

The Firewall policy contained separate rule collections for private east-west traffic and Internet egress.

## 1. Spoke A → Spoke B

```text
Collection:
allow-spoke-a-to-spoke-b

Priority:
100

Source:
10.1.0.0/16

Destination:
10.2.0.0/16

Protocol:
TCP

Port:
8080

Action:
Allow
```

This allowed private east-west traffic from A to B.

---

## 2. Spoke B → Internet

```text
Collection:
Deny

Priority:
150

Source:
10.2.0.0/16

Destination:
*

Protocol:
TCP

Ports:
80,443

Action:
Deny
```

This explicitly denied Internet HTTP/HTTPS traffic originating from Spoke B.

---

## 3. Spoke A → Internet

```text
Collection:
allow-spoke-internet-egress

Priority:
200

Source:
10.1.0.0/16

Destination:
*

Protocol:
TCP

Ports:
80,443

Action:
Allow
```

This allowed controlled Internet egress for Spoke A.

---

## 4. Spoke B → Spoke A

```text
Collection:
allow-spoke-b-to-spoke-a

Priority:
300

Source:
10.2.0.0/16

Destination:
10.1.0.0/16

Protocol:
TCP

Port:
8080

Action:
Allow
```

A separate collection was intentionally used for the reverse direction.

---

# 🔒 Private Subnets and Default Outbound Access

Both workload subnets were explicitly configured with:

```text
defaultOutboundAccess = false
```

The workload VMs had:

- No public IP
- No implicit/default outbound Internet access
- Explicit UDR-based routing toward Azure Firewall

This creates a much cleaner private workload architecture.

### Important distinction

These are separate concepts:

```text
defaultOutboundAccess=false
```

and:

```text
0.0.0.0/0 → Azure Firewall
```

Disabling default outbound access removes the implicit Azure outbound mechanism.

The UDR can still intentionally send Internet traffic to the Firewall.

---

# 🌐 Internet Egress Testing

## Spoke B → Internet

Spoke B attempted:

```bash
curl -4 -v --connect-timeout 5 http://api.ipify.org
```

The request attempted to connect to the public IP addresses of `api.ipify.org`, but all connection attempts timed out.

Example:

```text
Trying 104.26.12.205:80...
connect ... failed: Connection timed out

Trying 172.67.74.152:80...
connect ... failed: Connection timed out

curl: (28) Failed to connect
```

### Result

```text
❌ Spoke B → Internet
```

This demonstrated that B could not establish Internet connectivity.

The traffic was intentionally routed toward Azure Firewall, where the B → Internet TCP/80 and TCP/443 traffic was denied.

---

# 🌐 Spoke A → Internet

Spoke A executed:

```bash
curl -4 -s --connect-timeout 5 https://api.ipify.org
```

Result:

```text
20.219.254.150
```

This exactly matched the public IP assigned to Azure Firewall.

### Result

```text
✅ Spoke A → Internet
```

This demonstrated that Internet-bound traffic from A was:

```text
Spoke A
10.1.1.4
   │
   │ 0.0.0.0/0
   ▼
Azure Firewall
10.0.0.4
   │
   │ SNAT
   ▼
20.219.254.150
   │
   ▼
Internet
```

Azure Firewall therefore acted as the controlled Internet egress point.

---

# 🔄 East-West Traffic

Both directions of private east-west traffic were successfully validated **before the VM lifecycle changes**.

## Spoke A → Spoke B

Traffic from:

```text
10.1.1.4
```

to:

```text
10.2.1.4:8080
```

was successfully established through Azure Firewall.

Path:

```text
Spoke A
10.1.1.4
   │
   │ UDR:
   │ 10.2.0.0/16
   ▼
Azure Firewall
10.0.0.4
   │
   │ TCP/8080
   │ Allow
   ▼
Spoke B
10.2.1.4:8080
```

Result:

```text
Connected to 10.2.1.4 port 8080
HTTP/1.0 200 OK
```

### Result

```text
✅ A → B TCP/8080
```

This confirmed that the Spoke A UDR successfully forced the traffic through Azure Firewall and that the Firewall policy permitted it.

---

## Spoke B → Spoke A

The reverse direction was also successfully validated.

Path:

```text
Spoke B
10.2.1.4
   │
   │ UDR:
   │ 10.1.0.0/16
   ▼
Azure Firewall
10.0.0.4
   │
   │ TCP/8080
   │ Allow
   ▼
Spoke A
10.1.1.4:8080
```

Result:

```text
Connected to 10.1.1.4 port 8080
HTTP/1.0 200 OK
```

The response returned the Python HTTP server directory listing.

### Result

```text
✅ B → A TCP/8080
```

This confirmed that the reverse UDR and separate B → A Firewall rule were both functioning.

---

# 🧪 Post-Restart Troubleshooting

Later in the lab, VM B was deallocated and restarted while applying the private-subnet configuration.

The Python HTTP server had originally been started manually and was not configured as a persistent service.

After the VM lifecycle change, the listener disappeared.

Checking B showed:

```bash
ss -lntp | grep :8080 || true
```

with no output.

A subsequent A → B test produced:

```text
Connection refused
```

This was **not interpreted as a routing or Firewall failure**.

`Connection refused` indicated that the destination VM was reachable but nothing was listening on TCP/8080.

The listener was recreated:

```bash
nohup python3 -m http.server 8080 \
  --bind 0.0.0.0 >/tmp/http.log 2>&1 &
```

Verification then showed:

```text
LISTEN 0 5 0.0.0.0:8080 0.0.0.0:* users:(("python3",...))
```

The lab teardown began before another A → B test could be performed after recreating the listener.

### Final east-west conclusion

| Direction | Result | Evidence |
|---|---|---|
| A → B TCP/8080 | ✅ Successful | `HTTP/1.0 200 OK` |
| B → A TCP/8080 | ✅ Successful | `HTTP/1.0 200 OK` |
| A → B after B restart | ⚠️ Not retested after listener recreation | Lab teardown started |

Therefore, **east-west routing through Azure Firewall was successfully demonstrated in both directions**.

The later `Connection refused` event was an **application-listener lifecycle issue**, not evidence that the routing design had failed.

---

# 🧭 Effective Route Analysis

Effective routes were inspected on the workload NICs.

## Spoke A

Important routes included:

```text
10.1.0.0/16 → VnetLocal
10.0.0.0/16 → VNetPeering
10.2.0.0/16 → VirtualAppliance 10.0.0.4
0.0.0.0/0   → VirtualAppliance 10.0.0.4
```

---

## Spoke B

Important routes included:

```text
10.2.0.0/16 → VnetLocal
10.0.0.0/16 → VNetPeering
10.1.0.0/16 → VirtualAppliance 10.0.0.4
0.0.0.0/0   → VirtualAppliance 10.0.0.4
```

These effective routes provided direct evidence of the routing decisions Azure was making.

---

# 📊 Traffic Matrix

| Source | Destination | Path | Result |
|---|---|---|---|
| Spoke A | Internet | A → Firewall → Internet | ✅ Allowed |
| Spoke B | Internet | B → Firewall → Internet | ❌ Denied |
| Spoke A | Spoke B TCP/8080 | A → Firewall → B | ✅ Successful |
| Spoke B | Spoke A TCP/8080 | B → Firewall → A | ✅ Successful |
| Spoke A | Hub | VNet Peering | ✅ |
| Spoke B | Hub | VNet Peering | ✅ |
| Spoke A | Spoke B direct | No direct peering | ❌ |

---

# 🧠 Key Learnings

## 1. VNet Peering Is Non-Transitive

Having:

```text
A ↔ Hub
B ↔ Hub
```

does not automatically create:

```text
A ↔ B
```

A routing appliance is required for transit.

---

## 2. UDRs Control the Forwarding Path

Azure Firewall does not automatically receive all traffic.

The UDR explicitly identifies:

```text
Destination
     ↓
Virtual Appliance
     ↓
10.0.0.4
```

as the desired next hop.

---

## 3. Routing and Firewall Policy Are Different

A useful mental model is:

```text
UDR
 ↓
Where should the packet go?
 ↓
Azure Firewall
 ↓
Should the packet be allowed?
 ↓
Forward / SNAT
 ↓
Destination
```

A correct Firewall rule cannot compensate for an incorrect route.

Likewise, a correct route does not guarantee that Firewall policy will allow the traffic.

---

## 4. Stateful Appliances Need Symmetric Routing

When A → B is intentionally forced through a stateful Firewall, the reverse B → A path should also be designed to traverse the Firewall.

Otherwise, traffic may take different paths in each direction.

The lab demonstrated this by configuring:

```text
A → B
10.2.0.0/16 → Firewall
```

and:

```text
B → A
10.1.0.0/16 → Firewall
```

---

## 5. Private Workloads Can Still Have Controlled Internet Access

A VM does not need a public IP to access the Internet.

A centralized architecture can instead use:

```text
Private VM
   ↓
UDR
   ↓
Azure Firewall
   ↓
SNAT
   ↓
Internet
```

This provides centralized outbound control.

---

## 6. `defaultOutboundAccess=false` Makes the Private Model Explicit

Disabling implicit outbound access prevents the platform-provided default outbound mechanism from becoming an unexpected source of Internet connectivity.

Explicit outbound connectivity can then be designed using services such as:

- Azure Firewall
- NAT Gateway
- Standard Load Balancer outbound rules
- Other intentional egress architectures

---

## 7. Effective Routes Are Essential for Troubleshooting

Rather than assuming how Azure will route traffic, inspect the effective routes on the NIC.

For example:

```text
10.2.0.0/16 → VirtualAppliance 10.0.0.4
```

immediately explains why Spoke A traffic destined for Spoke B is sent toward the Firewall.

---

## 8. Troubleshooting Must Be Layered

When a connection fails, investigate in order:

```text
Effective Route
      ↓
UDR / Next Hop
      ↓
Firewall Policy
      ↓
Network Connectivity
      ↓
Destination Port
      ↓
Application
```

The `Connection refused` incident was a good example.

The routing infrastructure was working, but the application listener had disappeared after the VM lifecycle change.

---

# 🧪 Experiments Performed

### Experiment 1 — Force Spoke-to-Spoke Traffic Through Firewall

Configured:

```text
A → B
10.2.0.0/16 → 10.0.0.4
```

and:

```text
B → A
10.1.0.0/16 → 10.0.0.4
```

Result:

```text
✅ Both directions successfully tested
```

---

### Experiment 2 — Disable Default Outbound Access

Configured:

```text
Spoke A WorkloadSubnet
defaultOutboundAccess=false

Spoke B WorkloadSubnet
defaultOutboundAccess=false
```

This removed implicit/default outbound access for the workload subnets after the VMs were deallocated and restarted.

---

### Experiment 3 — Controlled Internet Egress

Configured:

```text
A → Firewall → Internet
```

with TCP/80 and TCP/443 allowed.

Result:

```text
Public IP:
20.219.254.150
```

This proved Firewall-based SNAT.

---

### Experiment 4 — Explicit Internet Deny

Configured:

```text
B → Internet
TCP/80,443
Action: Deny
```

Result:

```text
❌ Internet access from B
```

This demonstrated centralized egress enforcement.

---

### Experiment 5 — Application-Layer Troubleshooting

After VM B was restarted:

```text
TCP/8080
```

was no longer listening.

The resulting:

```text
Connection refused
```

was correctly identified as an application/listener problem rather than immediately modifying routes or Firewall rules.

---

# 🧹 Lab Cleanup

After completing the experiments, the entire resource group was removed:

```text
az-firewall-advanced-routing-lab-rg
```

This removed the lab resources and prevented unnecessary Azure charges.

---

# 🏁 Conclusion

This lab demonstrated Azure Firewall as both a **security boundary and a routing appliance** in a hub-and-spoke architecture.

The most important takeaway is that Azure networking consists of multiple independent layers:

```text
VNet Peering
      +
System Routes
      +
UDRs
      +
Routing Appliance
      +
Firewall Policy
      +
SNAT
```

The lab showed how these layers interact to create:

### Controlled north-south traffic

```text
Private VM
    ↓
UDR
    ↓
Azure Firewall
    ↓
Firewall Policy
    ↓
SNAT
    ↓
Internet
```

### Controlled east-west traffic

```text
Spoke A
    ↓
UDR
    ↓
Azure Firewall
    ↓
Firewall Policy
    ↓
Spoke B
```

The lab also reinforced an important DevOps/networking troubleshooting principle:

> **Do not assume the failure is where the symptom appears. Inspect the route, next hop, firewall policy, destination port, and application independently.**

This makes Azure networking problems much easier to reason about and troubleshoot systematically.

---

## 🛠️ Technologies Used

- Microsoft Azure
- Azure Virtual Network
- VNet Peering
- Azure Firewall
- Azure Firewall Policy
- User Defined Routes
- Network Interfaces
- Azure Virtual Machines
- Azure CLI
- Linux
- Python HTTP Server
- TCP/IP Routing
- SNAT
- Network Troubleshooting