# Azure Load Balancer Lab

A hands-on Azure networking lab demonstrating how to build, configure, troubleshoot, and validate an **Azure Standard Public Load Balancer** using the Azure CLI.

The lab uses two Ubuntu VMs running NGINX as backend servers. The Load Balancer exposes a single public frontend and distributes HTTP traffic between the backend VMs.

The lab also explores **NSGs, health probes, backend health, outbound connectivity, SNAT, and Azure's default outbound access behavior**.

---

## Architecture

```text
                              Internet
                                 |
                                 |
                         Public IP / Frontend
                                 |
                                 | TCP :80
                                 v
                    +-------------------------+
                    |   Azure Load Balancer   |
                    |       Standard SKU      |
                    +-------------------------+
                                 |
                         Backend Address Pool
                                 |
                    +------------+------------+
                    |                         |
                    v                         v
             +-------------+           +-------------+
             |     VM1     |           |     VM2     |
             | 10.10.1.4  |           | 10.10.2.4  |
             |   NGINX     |           |   NGINX     |
             |    :80      |           |    :80      |
             +-------------+           +-------------+
                    |                         |
              Subnet 1                  Subnet 2
             10.10.1.0/24              10.10.2.0/24
                    \                         /
                     \                       /
                      +---------------------+
                      |  VNet 10.10.0.0/16 |
                      +---------------------+
```

---

## Lab Resources

| Resource | Configuration |
|---|---|
| Resource Group | `az-lb-lab-rg` |
| Region | `centralindia` |
| VNet | `az-lb-lab-vnet` |
| VNet CIDR | `10.10.0.0/16` |
| Subnet 1 | `10.10.1.0/24` |
| Subnet 2 | `10.10.2.0/24` |
| VM1 | `10.10.1.4` |
| VM2 | `10.10.2.4` |
| Load Balancer | `az-lb-lab-lb` |
| LB SKU | Standard |
| LB Frontend | `az-lb-lab-frontend` |
| Backend Pool | `az-lb-lab-lbbepool` |
| Backend Port | `80` |
| Health Probe | HTTP `/` |
| Load-Balancing Rule | TCP `80 -> 80` |
| Outbound Rule | `az-lb-lab-outbound` |
| Outbound Protocol | `All` |
| Outbound Idle Timeout | `15` minutes |
| Allocated Outbound Ports | `1024` |

---

# 1. Objectives

This lab demonstrates:

- Creating an Azure VNet and multiple subnets.
- Deploying backend VMs into different subnets.
- Installing and configuring NGINX.
- Understanding NIC-level NSGs.
- Creating a Standard Public Load Balancer.
- Creating a frontend public IP configuration.
- Creating and populating a backend address pool.
- Creating an HTTP health probe.
- Creating a TCP load-balancing rule.
- Understanding the relationship between:
  - Frontend IP
  - Frontend port
  - Backend pool
  - Backend port
  - Health probe
  - NSGs
- Testing traffic distribution.
- Simulating backend failure.
- Understanding health-probe behavior.
- Configuring Load Balancer outbound connectivity.
- Understanding outbound SNAT.
- Removing public IPs from backend VMs.
- Disabling subnet default outbound access.
- Understanding the difference between Load Balancer outbound connectivity and Azure default outbound access.

---

# 2. Resource Group

Created the resource group:

```powershell
az group create `
  -n az-lb-lab-rg `
  -l centralindia `
  --tags owner=tarun
```

---

# 3. Virtual Network

Created the VNet:

```powershell
az network vnet create `
  -g az-lb-lab-rg `
  -l centralindia `
  -n az-lb-lab-vnet `
  --address-prefixes 10.10.0.0/16 `
  --subnet-name az-lb-lab-subnet1 `
  --subnet-prefixes 10.10.1.0/24 `
  --tags owner=tarun
```

Created the second subnet:

```powershell
az network vnet subnet create `
  -g az-lb-lab-rg `
  --vnet-name az-lb-lab-vnet `
  -n az-lb-lab-subnet2 `
  --address-prefixes 10.10.2.0/24
```

Final subnet layout:

```text
VNet: 10.10.0.0/16

Subnet 1:
10.10.1.0/24
    |
    +-- VM1: 10.10.1.4

Subnet 2:
10.10.2.0/24
    |
    +-- VM2: 10.10.2.4
```

The VMs were intentionally placed in different subnets while remaining inside the same VNet.

---

# 4. Backend VMs

Created two Ubuntu VMs.

## VM1

```text
Name:      az-lb-lab-vm1
Private IP: 10.10.1.4
Subnet:    az-lb-lab-subnet1
NIC:       az-lb-lab-vm1VMNic
```

## VM2

```text
Name:      az-lb-lab-vm2
Private IP: 10.10.2.4
Subnet:    az-lb-lab-subnet2
NIC:       az-lb-lab-vm2VMNic
```

Both VMs initially had public IP addresses for administration.

Those public IPs were later removed so that the VMs could be tested using the Load Balancer's outbound connectivity instead.

---

# 5. NGINX Backend Services

NGINX was installed on both VMs.

Different responses were configured so that Load Balancer traffic distribution could be observed:

```text
VM1 -> Hello from VM1
VM2 -> Hello from VM2
```

Direct connectivity was verified from inside the VNet:

```bash
curl http://10.10.1.4
```

```text
Hello from VM1
```

and:

```bash
curl http://10.10.2.4
```

```text
Hello from VM2
```

This confirmed that both backend applications were reachable over the VNet.

---

# 6. Network Security Groups

Both VMs had NSGs associated with their NICs.

Initially, the NSGs contained the default SSH rule:

```text
Name:       default-allow-ssh
Priority:   1000
Direction:  Inbound
Access:     Allow
Protocol:   TCP
Port:       22
```

When the Load Balancer was first tested, requests to the public frontend timed out.

The reason was that NGINX was listening on port 80, but the VM NSGs were not allowing inbound HTTP traffic.

An HTTP rule was therefore added to both NSGs:

```text
Name:       allow-http
Priority:   200
Direction:  Inbound
Access:     Allow
Protocol:   TCP
Port:       80
Source:     *
Destination: *
```

This allowed the Load Balancer to reach the backend NGINX services.

### Important lesson

The Load Balancer does not bypass the backend VM's network security.

The traffic path is conceptually:

```text
Client
  |
  v
Load Balancer
  |
  v
VM NIC / NSG
  |
  v
NGINX :80
```

The NSG must permit the required backend traffic.

---

# 7. Load Balancer Public IP

Created a Standard static public IP:

```text
Name:       az-lb-lab-pip
SKU:        Standard
Allocation: Static
Region:     centralindia
```

This public IP became the frontend entry point of the Load Balancer.

---

# 8. Azure Load Balancer

Created:

```text
Name:      az-lb-lab-lb
SKU:       Standard
Region:    centralindia
```

The frontend configuration:

```text
Frontend:
    az-lb-lab-frontend

Public IP:
    az-lb-lab-pip
```

The Load Balancer automatically created the backend address pool:

```text
az-lb-lab-lbbepool
```

Backend members:

```text
vm1 -> 10.10.1.4
vm2 -> 10.10.2.4
```

Verified with:

```powershell
az network lb address-pool address list `
  -g az-lb-lab-rg `
  --lb-name az-lb-lab-lb `
  --pool-name az-lb-lab-lbbepool `
  --query "[].{Name:name,IP:ipAddress}" `
  -o table
```

Result:

```text
Name    IP
vm1     10.10.1.4
vm2     10.10.2.4
```

---

# 9. Health Probe

Created an HTTP health probe:

```text
Name:     az-lb-lab-http-probe
Protocol: HTTP
Port:     80
Path:     /
```

The probe runs periodically against the backend.

The probe configuration included:

```text
Interval:       15 seconds
Number Probes:  2
Probe Threshold: 1
```

The health probe was associated with the Load Balancing Rule.

### Important concept

Backend pool membership and backend health are different concepts.

A VM can remain registered in the backend pool while being considered unhealthy.

Conceptually:

```text
Backend Pool
    |
    +-- VM1
    |
    +-- VM2

Health Probe
    |
    +-- VM1 -> Healthy / Unhealthy
    |
    +-- VM2 -> Healthy / Unhealthy
```

Only healthy backends are eligible to receive new Load Balancer traffic.

---

# 10. Load-Balancing Rule

Created:

```text
Name:          az-lb-lab-http-rule
Protocol:      TCP

Frontend:
    Port 80

Backend:
    Port 80

Backend Pool:
    az-lb-lab-lbbepool

Health Probe:
    az-lb-lab-http-probe
```

Traffic flow:

```text
Internet Client
      |
      | TCP :80
      v
Load Balancer Frontend
      |
      v
Backend Pool
   /       \
  v         v
VM1       VM2
:80       :80
```

---

# 11. Initial Load Balancer Test

The first request to the public Load Balancer IP failed:

```powershell
curl.exe http://<load-balancer-public-ip>
```

The connection timed out.

Troubleshooting showed that NGINX was running and listening on port 80.

The problem was the backend NSGs.

Initially they allowed SSH but not HTTP.

After adding:

```text
allow-http
TCP
Port 80
```

to both NSGs, the Load Balancer became reachable.

### Troubleshooting lesson

When an Azure Load Balancer request fails, check the entire path:

```text
Client
  |
  v
Public IP
  |
  v
Load Balancer Frontend
  |
  v
Load-Balancing Rule
  |
  v
Health Probe
  |
  v
Backend Pool
  |
  v
NSG
  |
  v
Application
```

A failure at any layer can result in the client seeing a timeout.

---

# 12. Traffic Distribution Test

Repeated requests were sent to the Load Balancer:

```powershell
1..10 | ForEach-Object {
    curl.exe -s http://<load-balancer-public-ip>
}
```

The responses alternated between:

```text
Hello from VM1
Hello from VM2
Hello from VM1
Hello from VM2
...
```

This confirmed that both backend VMs were receiving traffic.

### Important clarification

Azure Load Balancer should not be thought of as a simple round-robin mechanism.

The default distribution is flow-based and uses a hash of connection characteristics.

Repeated independent requests may therefore produce a distribution that looks round-robin, but the fundamental behavior is not simply:

```text
Request 1 -> VM1
Request 2 -> VM2
Request 3 -> VM1
```

---

# 13. Backend Failure Experiment

NGINX was deliberately stopped on VM1:

```bash
sudo systemctl stop nginx
```

VM1 remained registered in the backend pool.

However, the HTTP health probe could no longer successfully reach:

```text
VM1:80 /
```

After the probe detected the failure, new Load Balancer requests were sent only to VM2.

Repeated requests produced:

```text
Hello from VM2
Hello from VM2
Hello from VM2
...
```

### Key lesson

The Load Balancer does not remove the VM from the backend pool.

Instead:

```text
VM1
 |
 +-- Backend Pool membership: YES
 |
 +-- Health: UNHEALTHY
 |
 +-- Receives new LB traffic: NO
```

---

# 14. Backend Recovery

NGINX was started again:

```bash
sudo systemctl start nginx
```

Local verification:

```bash
curl http://localhost
```

Result:

```text
Hello from VM1
```

Once the health probe detected the recovered service, VM1 became eligible to receive Load Balancer traffic again.

---

# 15. Load Balancer Outbound Connectivity

The next part of the lab explored outbound connectivity.

The goal was to understand how backend VMs can reach the Internet through the Load Balancer rather than having individual public IP addresses.

The VMs' public IP associations were removed.

VM1:

```text
Private IP: 10.10.1.4
Public IP:  None
```

VM2:

```text
Private IP: 10.10.2.4
Public IP:  None
```

Verified using:

```powershell
az network nic show `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm1VMNic
```

and:

```powershell
az network nic show `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm2VMNic
```

Both NICs showed no public IP association.

---

# 16. Outbound Rule

An outbound rule was created on the Load Balancer:

```text
Name:             az-lb-lab-outbound
Protocol:         All
Frontend:         az-lb-lab-frontend
Backend Pool:     az-lb-lab-lbbepool
Idle Timeout:     15 minutes
Allocated Ports:  1024
TCP Reset:        Enabled
```

The outbound rule associates:

```text
Backend Pool
     |
     v
Load Balancer Frontend Public IP
```

This allows backend instances to use the Load Balancer frontend for outbound SNAT.

---

# 17. SNAT and the Outbound Rule

The outbound rule provides outbound SNAT.

Conceptually:

```text
VM1
10.10.1.4
   |
   | outbound connection
   v
Load Balancer
   |
   | SNAT
   v
Public Frontend IP
```

The external destination therefore sees the Load Balancer's public IP rather than the VM's private address.

The same applies to VM2.

### Important distinction

Inbound load balancing and outbound SNAT are separate functions.

```text
INBOUND

Internet
   |
   v
LB Public IP
   |
   v
Backend Pool
   |
   +--> VM1
   |
   +--> VM2
```

versus:

```text
OUTBOUND

VM1 / VM2
    |
    v
Backend Pool
    |
    v
LB Outbound Rule
    |
    v
LB Public IP
    |
    v
Internet
```

---

# 18. Disabling Default Outbound Access

To ensure that outbound Internet connectivity came specifically from the Load Balancer configuration, default outbound access was disabled on both subnets.

Subnet 1:

```powershell
az network vnet subnet update `
  -g az-lb-lab-rg `
  --vnet-name az-lb-lab-vnet `
  -n az-lb-lab-subnet1 `
  --default-outbound-access false
```

Subnet 2:

```powershell
az network vnet subnet update `
  -g az-lb-lab-rg `
  --vnet-name az-lb-lab-vnet `
  -n az-lb-lab-subnet2 `
  --default-outbound-access false
```

Verified:

```text
Subnet 1
defaultOutboundAccess = false

Subnet 2
defaultOutboundAccess = false
```

This is an important security configuration because it prevents relying on Azure's default outbound Internet connectivity.

---

# 19. Deallocation and Network Configuration Changes

During the experiment, the subnet outbound configuration was changed while the VMs were already running.

The VMs continued to demonstrate outbound Internet connectivity even after:

```text
Public IP = None
defaultOutboundAccess = false
```

This led to an important troubleshooting exercise.

The VMs were subsequently deallocated and started again.

After that, the expected outbound behavior took effect.

### Important lesson

Do **not** interpret this experiment as:

> "Every Azure networking change requires VM deallocation."

That is not generally true.

Instead, the important lesson is:

> Some Azure networking changes can depend on the VM's existing networking state, and when behavior does not match the new configuration, VM restart/deallocation can be an important troubleshooting step.

For Azure networking changes, always verify the actual effective configuration instead of assuming that the control-plane configuration immediately reflects the complete data-plane behavior.

---

# 20. Deallocation vs Restart

A VM deallocation is different from deleting the VM.

When a VM is deallocated:

```text
VM resource       -> remains
OS disk           -> remains
Data disks        -> remain
NIC                -> remains
Private IP config  -> remains
Configuration      -> remains
```

The VM is simply stopped and its compute allocation is released.

It can later be started again.

A restart is closer to rebooting a physical machine.

A deallocation goes further because Azure releases the underlying compute allocation.

Neither operation means:

```text
Delete VM
+
Create brand-new VM
```

---

# 21. Final Outbound Connectivity Test

After the required networking changes and VM restart/deallocation cycle, outbound Internet access was tested from both VMs.

The test command was:

```bash
curl -4 -s --max-time 10 https://api.ipify.org
```

The final experiment then disabled the Load Balancer outbound rule and removed the remaining outbound path.

After the VMs were deallocated and started again, the same test produced no public IP response.

This demonstrated that the VMs no longer had working Internet egress.

---

# 22. Final Network Security Model

The final lab state intentionally separates inbound application access from outbound Internet access.

### Inbound

```text
Internet
   |
   v
Load Balancer Public IP
   |
   | TCP :80
   v
Backend Pool
   |
   +------ VM1 :80
   |
   +------ VM2 :80
```

The VM NSGs permit HTTP traffic required by the application.

### Outbound

The Load Balancer outbound rule was used to demonstrate SNAT-based Internet egress.

It was subsequently disabled as part of the final security experiment.

The backend VMs were left without public IPs and with subnet default outbound access disabled.

Final result:

```text
VM1 ----X----> Internet
VM2 ----X----> Internet
```

while inbound application access through the Load Balancer remained conceptually separate:

```text
Internet
   |
   v
Load Balancer
   |
   +--> VM1
   |
   +--> VM2
```

---

# 23. Key Concepts Learned

## Load Balancer frontend

The frontend is the IP/port clients connect to.

```text
Public IP + TCP 80
```

---

## Backend pool

The backend pool identifies the resources that can receive traffic.

```text
VM1 -> 10.10.1.4
VM2 -> 10.10.2.4
```

---

## Health probe

The probe determines whether a backend is healthy.

```text
HTTP GET /
Port 80
```

Backend pool membership does not guarantee that a VM receives traffic.

---

## Load-balancing rule

The rule connects the frontend to the backend pool.

```text
Frontend TCP 80
        |
        v
Backend TCP 80
```

---

## NSG

The NSG controls whether traffic is allowed to reach the VM.

A correctly configured Load Balancer cannot make an NSG rule allowing traffic unnecessary.

---

## Outbound rule

The outbound rule provides a controlled outbound SNAT path through the Load Balancer frontend.

```text
Private VM
    |
    v
Outbound Rule
    |
    v
LB Public IP
    |
    v
Internet
```

---

## SNAT

SNAT changes the source address of an outbound connection so that the external destination sees a public source address.

In this lab, the Load Balancer frontend public IP was used for outbound connectivity.

---

## Default outbound access

Azure can provide default outbound Internet connectivity for VMs under certain configurations.

For a more controlled architecture, this should not be treated as an intentional application egress mechanism.

In this lab we explicitly disabled:

```text
defaultOutboundAccess = false
```

on both backend subnets.

---

# 24. Troubleshooting Lessons

### Problem 1 — Load Balancer timed out

Cause:

```text
VM NSGs did not allow TCP/80
```

Fix:

```text
Add allow-http rule
```

---

### Problem 2 — Backend stopped receiving traffic

Cause:

```text
NGINX stopped
```

Result:

```text
Health probe failed
VM became unhealthy
Load Balancer stopped sending new traffic
```

---

### Problem 3 — Outbound rule creation failed

The outbound rule initially failed because the existing Load Balancing Rule used the same frontend IP configuration and had outbound SNAT enabled.

Azure reported:

```text
LoadBalancingRuleMustDisableSNATSinceSameFrontendIPConfigurationIsReferencedByOutboundRule
```

The fix was:

```powershell
az network lb rule update `
  -g az-lb-lab-rg `
  --lb-name az-lb-lab-lb `
  -n az-lb-lab-http-rule `
  --disable-outbound-snat true
```

Then the outbound rule could be created successfully.

### Lesson

Inbound Load Balancing and explicit outbound SNAT configuration can interact when they share the same frontend IP configuration.

---

### Problem 4 — VMs continued to show Internet access after networking changes

The NICs showed:

```text
PublicIP = null
```

and the subnets showed:

```text
defaultOutboundAccess = false
```

Yet outbound connectivity continued temporarily.

The VMs were subsequently deallocated and started again.

After the networking state was refreshed, the expected behavior was observed.

### Lesson

When changing Azure networking behavior, distinguish between:

```text
Control-plane configuration
```

and:

```text
Effective data-plane behavior
```

Always verify from the VM rather than relying solely on the configuration shown by the CLI.

---

# 25. Useful Verification Commands

### Check backend pool

```powershell
az network lb address-pool address list `
  -g az-lb-lab-rg `
  --lb-name az-lb-lab-lb `
  --pool-name az-lb-lab-lbbepool `
  --query "[].{Name:name,IP:ipAddress}" `
  -o table
```

### Check health probe

```powershell
az network lb probe list `
  -g az-lb-lab-rg `
  --lb-name az-lb-lab-lb `
  --query "[].{Name:name,Protocol:protocol,Port:port,Path:requestPath}" `
  -o table
```

### Check Load Balancing Rule

```powershell
az network lb rule list `
  -g az-lb-lab-rg `
  --lb-name az-lb-lab-lb `
  --query "[].{Name:name,Protocol:protocol,FrontendPort:frontendPort,BackendPort:backendPort,Probe:probe.id,DisableOutboundSNAT:disableOutboundSnat}" `
  -o table
```

### Check outbound rule

```powershell
az network lb outbound-rule list `
  -g az-lb-lab-rg `
  --lb-name az-lb-lab-lb `
  --query "[].{Name:name,Protocol:protocol,IdleTimeout:idleTimeoutInMinutes,AllocatedPorts:allocatedOutboundPorts}" `
  -o table
```

### Check VM NIC public IP association

```powershell
az network nic show `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm1VMNic `
  --query "ipConfigurations[].{Name:name,PrivateIP:privateIPAddress,PublicIP:publicIPAddress.id}" `
  -o table
```

### Check subnet outbound configuration

```powershell
az network vnet subnet show `
  -g az-lb-lab-rg `
  --vnet-name az-lb-lab-vnet `
  -n az-lb-lab-subnet1 `
  --query "{Subnet:name,DefaultOutboundAccess:defaultOutboundAccess}" `
  -o json
```

### Test Internet egress from a VM

```powershell
az vm run-command invoke `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm1 `
  --command-id RunShellScript `
  --scripts "curl -4 -s --max-time 10 https://api.ipify.org; echo"
```

---

# 26. Final Takeaways

The most important concepts from this lab are:

1. **A Load Balancer frontend is not the same thing as a backend VM public IP.**

2. **Backend VMs can use only private IPs while still being reachable through a public Load Balancer.**

3. **The backend NSG must allow the required application traffic.**

4. **A backend pool defines membership; the health probe determines eligibility for traffic.**

5. **Health probes allow the Load Balancer to automatically stop sending traffic to unhealthy backends.**

6. **Inbound load balancing and outbound connectivity are separate concepts.**

7. **Outbound SNAT allows private backend VMs to initiate Internet connections using a Load Balancer frontend public IP.**

8. **An explicit outbound rule provides controlled outbound behavior rather than relying on default outbound access.**

9. **Removing a VM's public IP does not automatically mean the VM has no Internet connectivity. Other outbound mechanisms may still exist.**

10. **`defaultOutboundAccess=false` disables subnet default outbound connectivity, but effective behavior should always be verified from the VM.**

11. **VM deallocation does not delete the VM or its disks. It releases the compute allocation and can be useful when troubleshooting networking-state changes.**

12. **Do not assume that every Azure networking change requires VM deallocation. Verify the effective data-plane behavior and use restart/deallocation when the specific configuration requires or benefits from it.**

---

# 27. Lab Status

The Load Balancer lab has successfully demonstrated:

```text
[✓] Resource Group
[✓] VNet
[✓] Multiple subnets
[✓] Two backend VMs
[✓] NGINX backend services
[✓] NIC-level NSGs
[✓] HTTP NSG rules
[✓] Standard Public Load Balancer
[✓] Frontend public IP
[✓] Backend address pool
[✓] HTTP health probe
[✓] TCP load-balancing rule
[✓] Load distribution
[✓] Backend failure detection
[✓] Backend recovery
[✓] VM public IP removal
[✓] Load Balancer outbound rule
[✓] Outbound SNAT
[✓] Disabled subnet default outbound access
[✓] Verified outbound behavior
[✓] Disabled final Internet egress
```

The lab now provides a practical foundation for understanding Azure Load Balancer behavior before moving into more advanced Azure networking topics.