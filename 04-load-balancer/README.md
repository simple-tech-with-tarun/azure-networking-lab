# Azure Load Balancer Lab

A hands-on Azure networking lab demonstrating how to build and troubleshoot a **Standard Public Azure Load Balancer** using the Azure CLI.

The lab uses two Ubuntu VMs running NGINX as backend servers. The Load Balancer exposes a single public IP and distributes HTTP requests between the two VMs.

The lab also explores **outbound connectivity**, including Load Balancer outbound rules, SNAT, VM public IPs, and Azure's default outbound access behavior.

---

# Architecture

## Load Balancer Traffic

```text
                        Internet
                           |
                           |
                  Public IP: 20.207.206.109
                           |
                           v
               +-----------------------+
               |   Azure Load Balancer |
               |     Standard SKU      |
               +-----------------------+
                           |
                    Frontend: TCP/80
                           |
                           v
               +-----------------------+
               |    Backend Pool       |
               |                       |
               |  VM1        VM2       |
               | 10.10.1.4  10.10.2.4 |
               +-----------------------+
                   |             |
                   |             |
             NGINX :80     NGINX :80
                   |             |
            "Hello from     "Hello from
               VM1"             VM2"
```

## Outbound Connectivity

The lab also demonstrated the distinction between inbound Load Balancer traffic and outbound VM connectivity.

Final state:

```text
VM1: 10.10.1.4
  |
  | No VM Public IP
  | defaultOutboundAccess = false
  | LB outbound rule exists
  | Final outbound connectivity test failed
  |
  X
  |
Internet


VM2: 10.10.2.4
  |
  | No VM Public IP
  | defaultOutboundAccess = false
  | LB outbound rule exists
  | Final outbound connectivity test failed
  |
  X
  |
Internet
```

---

# Network

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
| Load Balancer Public IP | `20.207.206.109` |
| Backend Port | `80` |
| Probe | HTTP `/` |
| Frontend Port | `80` |

## Final Load Balancer Configuration

```text
Load Balancer
- Name: az-lb-lab-lb
- SKU: Standard
- Frontend: 20.207.206.109
- Frontend port: 80
- Backend port: 80
- Backend pool: az-lb-lab-lbbepool
- Health probe: HTTP :80 /
- Load-balancing rule: az-lb-lab-http-rule
- Outbound rule: az-lb-lab-outbound
- disableOutboundSnat: true
```

---

# Objectives

This lab demonstrates:

- Creating an Azure VNet and multiple subnets.
- Deploying VMs into different subnets.
- Installing and configuring NGINX on backend VMs.
- Understanding NIC-level NSG association.
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
- Verifying load-balancing behaviour.
- Simulating backend failure by stopping NGINX.
- Understanding that an unhealthy backend is removed from load-balancing consideration.
- Understanding outbound connectivity from Azure VMs.
- Understanding Load Balancer outbound rules.
- Understanding outbound SNAT.
- Understanding `disableOutboundSnat`.
- Understanding Azure default outbound access.
- Removing public IPs from backend VMs.
- Disabling default outbound access at the subnet level.
- Verifying that VMs no longer have Internet connectivity.
- Troubleshooting Azure networking layer by layer.

---

# 1. Resource Group

Created the resource group:

```powershell
az group create `
  -n az-lb-lab-rg `
  -l centralindia `
  --tags owner=tarun
```

---

# 2. Virtual Network

Created the VNet with the first subnet:

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

Verified:

```powershell
az network vnet subnet list `
  -g az-lb-lab-rg `
  --vnet-name az-lb-lab-vnet `
  --query "[].{Subnet:name,Prefix:addressPrefix}" `
  -o table
```

Expected:

```text
Subnet               Prefix
-------------------  -------------
az-lb-lab-subnet1    10.10.1.0/24
az-lb-lab-subnet2    10.10.2.0/24
```

---

# 3. Backend VMs

Created two Ubuntu 22.04 VMs.

## VM1

```powershell
az vm create `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm1 `
  -l centralindia `
  --image Ubuntu2204 `
  --size Standard_D2s_v5 `
  --vnet-name az-lb-lab-vnet `
  --subnet az-lb-lab-subnet1 `
  --admin-username azureuser `
  --generate-ssh-keys `
  --public-ip-sku Standard `
  --tags owner=tarun
```

VM1 received:

```text
Private IP: 10.10.1.4
```

## VM2

```powershell
az vm create `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm2 `
  -l centralindia `
  --image Ubuntu2204 `
  --size Standard_D2s_v5 `
  --vnet-name az-lb-lab-vnet `
  --subnet az-lb-lab-subnet2 `
  --admin-username azureuser `
  --generate-ssh-keys `
  --public-ip-sku Standard `
  --tags owner=tarun
```

VM2 received:

```text
Private IP: 10.10.2.4
```

Verified the NIC addresses:

```powershell
az network nic list `
  -g az-lb-lab-rg `
  --query "[].{NIC:name,PrivateIP:ipConfigurations[0].privateIPAddress}" `
  -o table
```

---

# 4. Install NGINX

NGINX was installed on both VMs using Azure VM Run Command.

## VM1

```powershell
az vm run-command invoke `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm1 `
  --command-id RunShellScript `
  --scripts "sudo apt-get update -y && sudo apt-get install -y nginx && echo 'Hello from VM1' | sudo tee /var/www/html/index.html"
```

## VM2

```powershell
az vm run-command invoke `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm2 `
  --command-id RunShellScript `
  --scripts "sudo apt-get update -y && sudo apt-get install -y nginx && echo 'Hello from VM2' | sudo tee /var/www/html/index.html"
```

The installation succeeded on both machines.

---

# 5. Management Access

VM configuration and troubleshooting were primarily performed through Azure CLI VM Run Command.

Example:

```powershell
az vm run-command invoke `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm1 `
  --command-id RunShellScript `
  --scripts "sudo systemctl status nginx --no-pager"
```

This allowed commands to be executed inside the VMs through Azure without requiring direct inbound SSH access to the backend VMs.

---

# 6. Verify Backend Connectivity

Before introducing the Load Balancer, backend connectivity was tested directly.

From VM1:

```powershell
az vm run-command invoke `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm1 `
  --command-id RunShellScript `
  --scripts "curl -s http://10.10.1.4 && echo && curl -s http://10.10.2.4"
```

Result:

```text
Hello from VM1

Hello from VM2
```

This confirmed that:

- Both VMs were reachable.
- NGINX was listening on port 80.
- Private VNet communication was working.
- The two VMs could communicate across their separate subnets.

---

# 7. NSG Configuration

Each VM had its own NIC-level NSG.

Initially, both NSGs contained the default SSH rule:

```text
default-allow-ssh
Priority: 1000
Direction: Inbound
Access: Allow
Protocol: TCP
Port: 22
```

HTTP access was then explicitly allowed.

## VM1

```powershell
az network nsg rule create `
  -g az-lb-lab-rg `
  --nsg-name az-lb-lab-vm1NSG `
  --name allow-http `
  --priority 200 `
  --direction Inbound `
  --access Allow `
  --protocol Tcp `
  --source-address-prefixes "*" `
  --destination-address-prefixes "*" `
  --destination-port-ranges 80
```

## VM2

```powershell
az network nsg rule create `
  -g az-lb-lab-rg `
  --nsg-name az-lb-lab-vm2NSG `
  --name allow-http `
  --priority 200 `
  --direction Inbound `
  --access Allow `
  --protocol Tcp `
  --source-address-prefixes "*" `
  --destination-address-prefixes "*" `
  --destination-port-ranges 80
```

The important lesson here is:

> **Creating a Load Balancer does not bypass NSGs.**

Traffic still has to satisfy the applicable network security rules.

---

# 8. Create the Public IP

Created a Standard static public IP:

```powershell
az network public-ip create `
  -g az-lb-lab-rg `
  -n az-lb-lab-pip `
  -l centralindia `
  --sku Standard `
  --allocation-method Static `
  --tags owner=tarun
```

Public IP:

```text
20.207.206.109
```

Verified:

```powershell
az network public-ip show `
  -g az-lb-lab-rg `
  -n az-lb-lab-pip `
  --query "{Name:name,IP:ipAddress,SKU:sku.name,Allocation:publicIPAllocationMethod}" `
  -o table
```

---

# 9. Create the Load Balancer

Created a Standard Public Load Balancer:

```powershell
az network lb create `
  -g az-lb-lab-rg `
  -n az-lb-lab-lb `
  -l centralindia `
  --sku Standard `
  --public-ip-address az-lb-lab-pip `
  --frontend-ip-name az-lb-lab-frontend `
  --tags owner=tarun
```

Azure automatically created the backend pool:

```text
az-lb-lab-lbbepool
```

---

# 10. Add Backend Servers

Added VM1:

```powershell
az network lb address-pool address add `
  -g az-lb-lab-rg `
  --lb-name az-lb-lab-lb `
  --pool-name az-lb-lab-lbbepool `
  -n vm1 `
  --vnet az-lb-lab-vnet `
  --ip-address 10.10.1.4
```

Added VM2:

```powershell
az network lb address-pool address add `
  -g az-lb-lab-rg `
  --lb-name az-lb-lab-lb `
  --pool-name az-lb-lab-lbbepool `
  -n vm2 `
  --vnet az-lb-lab-vnet `
  --ip-address 10.10.2.4
```

Verified:

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
------  --------
vm1     10.10.1.4
vm2     10.10.2.4
```

---

# 11. Create the Health Probe

Created an HTTP health probe:

```powershell
az network lb probe create `
  -g az-lb-lab-rg `
  --lb-name az-lb-lab-lb `
  -n az-lb-lab-http-probe `
  --protocol Http `
  --port 80 `
  --path /
```

Configuration:

```text
Protocol: HTTP
Port:     80
Path:     /
Interval: 15 seconds
```

The probe checks:

```text
http://<backend-ip>:80/
```

The backend must successfully respond for the Load Balancer to consider it healthy.

---

# 12. Create the Load-Balancing Rule

Created the frontend-to-backend rule:

```powershell
az network lb rule create `
  -g az-lb-lab-rg `
  --lb-name az-lb-lab-lb `
  -n az-lb-lab-http-rule `
  --frontend-ip-name az-lb-lab-frontend `
  --frontend-port 80 `
  --backend-pool-name az-lb-lab-lbbepool `
  --backend-port 80 `
  --protocol Tcp `
  --probe-name az-lb-lab-http-probe
```

The traffic flow is now:

```text
Client
  |
  | TCP/80
  v
20.207.206.109
  |
  | Load Balancer
  v
Backend Pool
  |
  +---- 10.10.1.4:80
  |
  +---- 10.10.2.4:80
```

The health probe determines which backend instances are eligible to receive traffic.

---

# 13. Initial Troubleshooting

The first attempt to access the public IP failed:

```powershell
curl.exe http://20.207.206.109
```

Result:

```text
curl: (28) Failed to connect to 20.207.206.109:80
```

Rather than assuming the Load Balancer was broken, the configuration was investigated layer by layer.

## Check NSGs

```powershell
az network nsg rule list `
  -g az-lb-lab-rg `
  --nsg-name az-lb-lab-vm1NSG `
  -o table
```

Initially, the NSG only allowed SSH.

HTTP/80 was therefore explicitly allowed on both VM NSGs.

## Check NGINX

VM1 was checked:

```powershell
az vm run-command invoke `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm1 `
  --command-id RunShellScript `
  --scripts "sudo ss -lntp | grep ':80 '"
```

Result confirmed that NGINX was listening on port 80:

```text
LISTEN ... 0.0.0.0:80 ...
LISTEN ... [::]:80 ...
```

After the NSG correction, the Load Balancer began distributing requests successfully.

---

# 14. Verify Load Balancing

Requests were sent repeatedly to the public IP:

```powershell
1..10 | ForEach-Object {
  curl.exe -s http://20.207.206.109
}
```

The responses included:

```text
Hello from VM1
Hello from VM2
Hello from VM1
Hello from VM2
Hello from VM1
Hello from VM2
...
```

This demonstrated that the Load Balancer was successfully distributing traffic between the two healthy backend servers.

---

# 15. Test Health-Based Failover

To simulate a backend failure, NGINX was stopped on VM1:

```powershell
az vm run-command invoke `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm1 `
  --command-id RunShellScript `
  --scripts "sudo systemctl stop nginx"
```

At this point VM1 still existed and remained in the backend pool, but its HTTP health probe began failing.

The important distinction is:

```text
Backend pool membership
       ≠
Healthy backend
```

A server can remain configured in the backend pool while the Load Balancer stops sending it new traffic because its health probe fails.

Repeated requests then returned only:

```text
Hello from VM2
Hello from VM2
Hello from VM2
...
```

VM1 was effectively removed from load-balancing consideration because it was unhealthy.

---

# 16. Restore VM1

NGINX was restarted:

```powershell
az vm run-command invoke `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm1 `
  --command-id RunShellScript `
  --scripts "sudo systemctl start nginx"
```

Verified locally:

```powershell
az vm run-command invoke `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm1 `
  --command-id RunShellScript `
  --scripts "curl -s http://localhost"
```

Result:

```text
Hello from VM1
```

After the health probe detected the recovered backend, VM1 became eligible for load-balanced traffic again.

---

# 17. Explore Outbound Connectivity

After completing the inbound Load Balancer configuration, the lab was extended to investigate how backend VMs obtain outbound Internet connectivity.

This introduced several different concepts:

```text
VM Public IP
      |
      +---- Direct public connectivity


Load Balancer outbound rule
      |
      +---- Outbound SNAT through LB frontend


Default outbound access
      |
      +---- Azure-provided outbound connectivity
```

These mechanisms are separate and should not be treated as the same thing.

---

# 18. Remove VM Public IPs

The original VM deployments had public IPs attached to their NICs.

The NICs were inspected:

```powershell
az network nic show `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm1VMNic `
  --query "ipConfigurations[0].{Name:name,PrivateIP:privateIPAddress,PublicIP:publicIPAddress.id}" `
  -o json
```

and:

```powershell
az network nic show `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm2VMNic `
  --query "ipConfigurations[0].{Name:name,PrivateIP:privateIPAddress,PublicIP:publicIPAddress.id}" `
  -o json
```

The public IP associations were then removed.

## VM1

```powershell
az network nic ip-config update `
  -g az-lb-lab-rg `
  --nic-name az-lb-lab-vm1VMNic `
  -n ipconfigaz-lb-lab-vm1 `
  --remove publicIPAddress
```

## VM2

```powershell
az network nic ip-config update `
  -g az-lb-lab-rg `
  --nic-name az-lb-lab-vm2VMNic `
  -n ipconfigaz-lb-lab-vm2 `
  --remove publicIPAddress
```

Verified:

```powershell
az network nic list `
  -g az-lb-lab-rg `
  --query "[].{NIC:name,PrivateIP:ipConfigurations[0].privateIPAddress,PublicIP:ipConfigurations[0].publicIPAddress.id}" `
  -o table
```

Final NIC state:

```text
NIC                    PrivateIP    PublicIP
---------------------  -----------  --------
az-lb-lab-vm1VMNic     10.10.1.4
az-lb-lab-vm2VMNic     10.10.2.4
```

The VMs no longer had directly associated public IPs.

---

# 19. Create a Load Balancer Outbound Rule

The Load Balancer was then configured with an outbound rule:

```powershell
az network lb outbound-rule create `
  -g az-lb-lab-rg `
  --lb-name az-lb-lab-lb `
  -n az-lb-lab-outbound `
  --frontend-ip-configs az-lb-lab-frontend `
  --address-pool az-lb-lab-lbbepool `
  --protocol All `
  --idle-timeout 15 `
  --enable-tcp-reset true
```

The first attempt failed with:

```text
LoadBalancingRuleMustDisableSNATSinceSameFrontendIPConfigurationIsReferencedByOutboundRule
```

This was an important learning point.

The same frontend IP configuration was being used by:

```text
Inbound Load-Balancing Rule
       +
Outbound Rule
```

The existing load-balancing rule therefore needed outbound SNAT disabled.

---

# 20. Disable Outbound SNAT on the Load-Balancing Rule

Updated the existing rule:

```powershell
az network lb rule update `
  -g az-lb-lab-rg `
  --lb-name az-lb-lab-lb `
  -n az-lb-lab-http-rule `
  --disable-outbound-snat true
```

Verified:

```powershell
az network lb rule show `
  -g az-lb-lab-rg `
  --lb-name az-lb-lab-lb `
  -n az-lb-lab-http-rule `
  --query "{Name:name,FrontendPort:frontendPort,BackendPort:backendPort,DisableOutboundSNAT:disableOutboundSnat}" `
  -o table
```

Result:

```text
Name                  FrontendPort    BackendPort    DisableOutboundSNAT
--------------------  --------------  -------------  -------------------
az-lb-lab-http-rule   80              80             True
```

The outbound rule could then be created successfully.

---

# 21. Verify Load Balancer Outbound Rule

The outbound rule was configured as:

```text
Name:             az-lb-lab-outbound
Protocol:         All
Idle timeout:     15 minutes
Allocated ports:  1024
Frontend:         az-lb-lab-frontend
Backend pool:     az-lb-lab-lbbepool
TCP reset:        Enabled
```

Verified:

```powershell
az network lb outbound-rule list `
  -g az-lb-lab-rg `
  --lb-name az-lb-lab-lb `
  --query "[].{Name:name,Protocol:protocol,IdleTimeout:idleTimeoutInMinutes,AllocatedPorts:allocatedOutboundPorts}" `
  -o table
```

Result:

```text
Name                Protocol    IdleTimeout    AllocatedPorts
------------------  ----------  -------------  ---------------
az-lb-lab-outbound  All         15             1024
```

---

# 22. Unexpected Outbound Behaviour

The VMs were tested:

```powershell
az vm run-command invoke `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm1 `
  --command-id RunShellScript `
  --scripts "curl -4 -s --max-time 10 https://api.ipify.org; echo"
```

and:

```powershell
az vm run-command invoke `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm2 `
  --command-id RunShellScript `
  --scripts "curl -4 -s --max-time 10 https://api.ipify.org; echo"
```

At this stage the VMs were still able to reach the Internet.

The observed public addresses were:

```text
VM1 → 20.207.198.126
VM2 → 20.204.43.214
```

This was an important troubleshooting discovery:

```text
No public IP on the VM NIC
       ≠
No outbound Internet connectivity
```

Removing the VM public IPs alone was not sufficient to guarantee that outbound Internet connectivity was disabled.

---

# 23. Investigate Effective Routes

The effective route table was inspected:

```powershell
az network nic show-effective-route-table `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm2VMNic `
  -o table
```

The VMs had a default route:

```text
Source    State    Address Prefix    Next Hop Type
--------  ------   ---------------   -------------
Default   Active   0.0.0.0/0         Internet
```

The VM also had its local subnet route:

```text
10.10.2.0/24 → VnetLocal
```

The default route helped explain why the VM could still reach external destinations.

---

# 24. Disable Default Outbound Access

To prevent Azure's default outbound connectivity mechanism from providing Internet access, `defaultOutboundAccess` was disabled on both subnets.

## Subnet 1

```powershell
az network vnet subnet update `
  -g az-lb-lab-rg `
  --vnet-name az-lb-lab-vnet `
  -n az-lb-lab-subnet1 `
  --default-outbound-access false
```

Result:

```text
defaultOutboundAccess: false
```

## Subnet 2

```powershell
az network vnet subnet update `
  -g az-lb-lab-rg `
  --vnet-name az-lb-lab-vnet `
  -n az-lb-lab-subnet2 `
  --default-outbound-access false
```

Result:

```text
defaultOutboundAccess: false
```

Verified:

```powershell
az network vnet subnet show `
  -g az-lb-lab-rg `
  --vnet-name az-lb-lab-vnet `
  -n az-lb-lab-subnet1 `
  --query "{Subnet:name,DefaultOutboundAccess:defaultOutboundAccess}" `
  -o json
```

and:

```powershell
az network vnet subnet show `
  -g az-lb-lab-rg `
  --vnet-name az-lb-lab-vnet `
  -n az-lb-lab-subnet2 `
  --query "{Subnet:name,DefaultOutboundAccess:defaultOutboundAccess}" `
  -o json
```

Both returned:

```json
{
  "DefaultOutboundAccess": false
}
```

---

# 25. VM Restart and Networking Behaviour

During the experiment, the VMs were restarted/deallocated and started again to ensure the updated networking state was reflected.

The VMs were restarted:

```powershell
az vm restart `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm1
```

```powershell
az vm restart `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm2
```

The VMs were also explicitly started after being deallocated during troubleshooting:

```powershell
az vm start `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm1
```

```powershell
az vm start `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm2
```

### Important lesson

During this lab, the updated outbound behaviour was not observed until the VMs had been restarted/deallocated and started again.

This should not be interpreted as a universal rule that every Azure networking change requires VM deallocation.

The deallocation/restart cycle was part of this specific experiment and troubleshooting process.

---

# 26. Final Outbound Connectivity Test

After the networking changes and VM restart/deallocation cycle, outbound connectivity was tested again.

## VM1

```powershell
az vm run-command invoke `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm1 `
  --command-id RunShellScript `
  --scripts "curl -4 -s --max-time 10 https://api.ipify.org; echo"
```

The request did not return a public IP and outbound Internet connectivity was no longer functional.

## VM2

```powershell
az vm run-command invoke `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm2 `
  --command-id RunShellScript `
  --scripts "curl -4 -s --max-time 10 https://api.ipify.org; echo"
```

The request did not return a public IP and outbound Internet connectivity was no longer functional.

This confirmed that the final VM configuration no longer provided working outbound Internet connectivity.

---

# Key Lessons

## 1. A Load Balancer is not an application-layer reverse proxy

Azure Load Balancer operates primarily at Layer 4.

It distributes network connections rather than performing application-layer HTTP routing.

```text
Azure Load Balancer
→ Layer 4
→ TCP/UDP
→ Connection distribution
→ Health-based backend selection
```

This differs from an application-layer service such as Azure Application Gateway:

```text
Application Gateway
→ Layer 7
→ HTTP/HTTPS
→ URL/host/header-based routing
→ WAF capabilities
```

---

## 2. Backend pool membership does not mean healthy

VM1 and VM2 remained members of:

```text
az-lb-lab-lbbepool
```

But when NGINX stopped on VM1:

```text
VM1
 |
 X HTTP probe fails
 |
Unhealthy
```

The Load Balancer stopped sending traffic to it.

---

## 3. Health probes are critical

The health probe was:

```text
HTTP
Port 80
Path /
```

If the backend cannot successfully answer that probe, the Load Balancer considers it unhealthy.

This means an application-level failure can affect traffic distribution even when:

- The VM is running.
- The NIC exists.
- The backend is in the pool.
- The IP address is correct.

---

## 4. NSGs still matter

The Load Balancer does not override NSG security.

The traffic path must satisfy the applicable network security rules.

In this lab:

```text
Internet
  |
  v
Load Balancer
  |
  v
VM NIC
  |
  v
NSG
  |
  v
NGINX :80
```

Every layer matters.

---

## 5. Troubleshooting should be layered

When the public IP initially failed, the investigation followed the traffic path:

```text
Public IP
   ↓
Load Balancer
   ↓
Frontend configuration
   ↓
Load-balancing rule
   ↓
Backend pool
   ↓
Health probe
   ↓
NIC / NSG
   ↓
VM
   ↓
NGINX :80
```

This is a much more reliable troubleshooting method than randomly changing settings.

---

## 6. Inbound and outbound Load Balancer traffic are different

The Load Balancer can have:

```text
Inbound load-balancing rules
```

and:

```text
Outbound rules
```

These serve different purposes.

Inbound:

```text
Internet
  |
  v
LB Frontend
  |
  v
Backend VM
```

Outbound:

```text
Backend VM
  |
  v
LB outbound SNAT
  |
  v
LB Frontend Public IP
  |
  v
Internet
```

---

## 7. Outbound SNAT matters

When an outbound rule uses the same frontend IP configuration as a load-balancing rule, the inbound rule may need:

```text
disableOutboundSnat = true
```

This was demonstrated by the Azure error:

```text
LoadBalancingRuleMustDisableSNATSinceSameFrontendIPConfigurationIsReferencedByOutboundRule
```

The existing rule was therefore updated before the outbound rule could be created.

---

## 8. Removing a VM public IP does not necessarily eliminate outbound connectivity

One of the most important lessons from this experiment was:

```text
NIC Public IP = None
       ≠
VM has no Internet access
```

Azure can provide outbound connectivity through other mechanisms.

Therefore, when troubleshooting unexpected outbound connectivity, check:

- VM public IP
- Load Balancer outbound rules
- SNAT
- NAT Gateway, if present
- subnet `defaultOutboundAccess`
- effective routes
- NSGs
- other Azure networking components

---

## 9. Default outbound access is separate from Load Balancer outbound SNAT

The lab demonstrated that these should not be treated as interchangeable:

```text
Default outbound access
       ≠
Load Balancer outbound SNAT
       ≠
VM Public IP
```

A proper Azure network design should deliberately decide which outbound mechanism is intended.

---

## 10. Explicit outbound design is preferable

For production-style architectures, outbound Internet access should be deliberately designed rather than relying on implicit/default behaviour.

Possible architectures can include:

```text
VM
 |
 +--> NAT Gateway --> Internet
```

or:

```text
VM
 |
 +--> Azure Load Balancer outbound rule --> Internet
```

or:

```text
VM
 |
 +--> No outbound path
```

The appropriate choice depends on the architecture and security requirements.

---

## 11. Deallocation is not deletion

During troubleshooting, the VMs were deallocated and restarted.

Deallocation does not mean the VM was deleted and recreated.

The VM resource, disks, configuration, private networking configuration, and operating-system data remain associated with the VM.

For this lab, deallocation/restart was used as part of the troubleshooting process to observe the intended outbound networking behaviour.

---

# Final State

```text
Resource Group
└── az-lb-lab-rg
    |
    ├── VNet
    │   └── az-lb-lab-vnet
    │       |
    │       ├── 10.10.1.0/24
    │       │   └── VM1
    │       │       └── 10.10.1.4
    │       |
    │       └── 10.10.2.0/24
    │           └── VM2
    │               └── 10.10.2.4
    |
    ├── VM1 Public IP
    │   └── None
    |
    ├── VM2 Public IP
    │   └── None
    |
    ├── Subnet 1
    │   └── defaultOutboundAccess = false
    |
    ├── Subnet 2
    │   └── defaultOutboundAccess = false
    |
    ├── Public IP
    │   └── 20.207.206.109
    |
    └── Standard Load Balancer
        |
        ├── Frontend
        │   └── 20.207.206.109:80
        |
        ├── Backend Pool
        │   ├── VM1: 10.10.1.4
        │   └── VM2: 10.10.2.4
        |
        ├── Health Probe
        │   └── HTTP :80 /
        |
        ├── Load-Balancing Rule
        │   └── TCP :80 → TCP :80
        │
        │   disableOutboundSnat = true
        |
        └── Outbound Rule
            └── az-lb-lab-outbound
```

## Final inbound state

```text
Internet
    |
    v
20.207.206.109:80
    |
    v
Azure Standard Load Balancer
    |
    +----> VM1: 10.10.1.4:80
    |
    +----> VM2: 10.10.2.4:80
```

## Final outbound state

```text
VM1 10.10.1.4
 |
 | No VM Public IP
 | defaultOutboundAccess = false
 | LB outbound rule exists
 | Final outbound connectivity test failed
 |
 X
 |
Internet


VM2 10.10.2.4
 |
 | No VM Public IP
 | defaultOutboundAccess = false
 | LB outbound rule exists
 | Final outbound connectivity test failed
 |
 X
 |
Internet
```

Final verification:

```text
VM1 → Internet: BLOCKED
VM2 → Internet: BLOCKED

Internet → Load Balancer: ALLOWED
Load Balancer → VM1: ALLOWED
Load Balancer → VM2: ALLOWED
```

---

# Cleanup

Because the lab resources are contained in a dedicated resource group, the entire environment can be removed with:

```powershell
az group delete --name az-lb-lab-rg
```

Verify deletion:

```powershell
az group exists --name az-lb-lab-rg
```

Expected:

```text
false
```

---

# What This Lab Demonstrated

By completing this lab, the following Azure networking concepts were covered:

- Azure Virtual Network
- Multiple subnets
- Private IP addressing
- Ubuntu VMs
- NGINX
- NIC-level NSGs
- Standard Public Load Balancer
- Static Standard Public IP
- Frontend IP configuration
- Backend address pool
- Backend IP addresses
- Health probes
- Load-balancing rules
- NSG interaction
- Backend health
- Failure detection
- Automatic backend exclusion
- Backend recovery
- Load distribution
- Outbound SNAT
- Load Balancer outbound rules
- `disableOutboundSnat`
- Default outbound access
- `defaultOutboundAccess`
- VM public IP removal
- Effective route inspection
- VM restart/deallocation behaviour
- Outbound Internet troubleshooting
- Layered Azure network troubleshooting

---

# Lab Takeaway

The main lesson from this lab is that Azure networking is composed of multiple independent layers.

For inbound traffic:

```text
Internet
  ↓
Public IP
  ↓
Load Balancer Frontend
  ↓
Load-Balancing Rule
  ↓
Backend Pool
  ↓
Health Probe
  ↓
NIC / NSG
  ↓
VM
  ↓
Application
```

For outbound traffic:

```text
VM
  ↓
NIC
  ↓
Subnet
  ↓
Outbound mechanism
  ↓
Internet
```

A networking problem should therefore be investigated by following the actual traffic path rather than changing configuration randomly.

The lab also demonstrated that **having no public IP on a VM does not automatically mean the VM has no Internet access**. Outbound connectivity must be intentionally designed and verified.

The final configuration deliberately separates:

```text
Public Load Balancer
       ↓
Inbound application traffic

from

VM outbound connectivity
       ↓
Explicitly disabled
```

This provides a clearer foundation for building more advanced Azure networking architectures.