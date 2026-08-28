Absolutely. For the README, I’d document this as a **hands-on Azure Load Balancer lab**, including the troubleshooting journey because that’s where a lot of the useful learning happened.

# Azure Load Balancer Lab

A hands-on Azure networking lab demonstrating how to build and troubleshoot a **Standard Public Azure Load Balancer** using the Azure CLI.

The lab uses two Ubuntu VMs running NGINX as backend servers. The Load Balancer exposes a single public IP and distributes HTTP requests between the two VMs.

---

## Architecture

```text
                         Internet
                            |
                            |
                    Public IP: 20.207.206.109
                            |
                            v
                 +-----------------------+
                 |   Azure Load Balancer |
                 |      Standard SKU     |
                 +-----------------------+
                            |
                     Frontend: TCP/80
                            |
                            v
                 +-----------------------+
                 |   Backend Pool        |
                 |                       |
                 |  VM1       VM2        |
                 | 10.10.1.4  10.10.2.4 |
                 +-----------------------+
                    |             |
                    |             |
              NGINX :80      NGINX :80
                    |             |
              "Hello from     "Hello from
                 VM1"             VM2"
```

### Network

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
| Public IP | `20.207.206.109` |
| Backend Port | `80` |
| Probe | HTTP `/` |

---

## Objectives

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
- Verifying load balancing behaviour.
- Simulating backend failure by stopping NGINX.
- Understanding that an unhealthy backend is removed from load-balancing consideration.

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
Subnet             Prefix
-----------------  -------------
az-lb-lab-subnet1  10.10.1.0/24
az-lb-lab-subnet2  10.10.2.0/24
```

---

# 3. Backend VMs

Created two Ubuntu 22.04 VMs.

### VM1

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

### VM2

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

NGINX was installed on both VMs using VM Run Command.

### VM1

```powershell
az vm run-command invoke `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm1 `
  --command-id RunShellScript `
  --scripts "sudo apt-get update -y && sudo apt-get install -y nginx && echo 'Hello from VM1' | sudo tee /var/www/html/index.html"
```

### VM2

```powershell
az vm run-command invoke `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm2 `
  --command-id RunShellScript `
  --scripts "sudo apt-get update -y && sudo apt-get install -y nginx && echo 'Hello from VM2' | sudo tee /var/www/html/index.html"
```

The installation succeeded on both machines.

---

# 5. Verify Backend Connectivity

Before introducing the Load Balancer, backend connectivity was tested directly.

From VM2:

```powershell
az vm run-command invoke `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm2 `
  --command-id RunShellScript `
  --scripts "curl -s http://10.10.2.4 && echo && curl -s http://10.10.1.4"
```

Result:

```text
Hello from VM2

Hello from VM1
```

This confirmed that:

- Both VMs were reachable.
- NGINX was listening on port 80.
- The VNet/subnet configuration allowed private communication.

---

# 6. NSG Configuration

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

The same rule was created on VM2:

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

The important lesson here is that **creating a Load Balancer does not bypass NSGs**.

The traffic still has to satisfy the applicable network security rules.

---

# 7. Create the Public IP

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

# 8. Create the Load Balancer

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

# 9. Add Backend Servers

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
  --query "[].{Name:name,IP:backendAddress.ipAddress}" `
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

# 10. Create the Health Probe

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

# 11. Create the Load-Balancing Rule

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

# 12. Initial Troubleshooting

The first attempt to access the public IP failed:

```powershell
curl.exe http://20.207.206.109
```

Result:

```text
curl: (28) Failed to connect to 20.207.206.109:80
```

Rather than assuming the Load Balancer was broken, the configuration was investigated layer by layer.

### Check NSGs

```powershell
az network nic list `
  -g az-lb-lab-rg `
  --query "[].{NIC:name,PrivateIP:ipConfigurations[0].privateIPAddress,NSG:networkSecurityGroup.id}" `
  -o table
```

Both NICs had NSGs attached.

The NSGs initially allowed SSH only, so HTTP/80 was explicitly allowed.

### Check NGINX

VM1 was checked:

```powershell
az vm run-command invoke `
  -g az-lb-lab-rg `
  -n az-lb-lab-vm1 `
  --command-id RunShellScript `
  --scripts "sudo ss -lntp | grep ':80 '"
```

Result:

```text
LISTEN ... 0.0.0.0:80 ...
LISTEN ... [::]:80 ...
```

This confirmed that NGINX was actually listening on port 80.

After the NSG correction, the Load Balancer began distributing requests successfully.

---

# 13. Verify Load Balancing

Requests were sent repeatedly to the public IP:

```powershell
1..10 | ForEach-Object {
    curl.exe -s http://20.207.206.109
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

This demonstrated that the Load Balancer was successfully distributing traffic between the two healthy backend servers.

---

# 14. Test Health-Based Failover

To simulate a backend failure, NGINX was stopped on VM1.

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

# 15. Restore VM1

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

# Key Lessons

## 1. A Load Balancer is not a proxy

The Load Balancer does not simply forward traffic blindly.

It maintains knowledge of backend health and sends traffic only to eligible backend instances.

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

The traffic path must satisfy network security rules.

In this lab, the initial failure helped demonstrate this:

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

When the public IP failed, the investigation followed the traffic path:

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

# Final State

```text
Resource Group
└── az-lb-lab-rg
    │
    ├── VNet
    │   └── az-lb-lab-vnet
    │       ├── 10.10.1.0/24
    │       │   └── VM1: 10.10.1.4
    │       │
    │       └── 10.10.2.0/24
    │           └── VM2: 10.10.2.4
    │
    ├── Public IP
    │   └── 20.207.206.109
    │
    └── Standard Load Balancer
        ├── Frontend
        │   └── 20.207.206.109:80
        │
        ├── Backend Pool
        │   ├── VM1: 10.10.1.4
        │   └── VM2: 10.10.2.4
        │
        ├── Health Probe
        │   └── HTTP :80 /
        │
        └── Load-Balancing Rule
            └── TCP :80 → TCP :80
```

## Cleanup

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

## What This Lab Demonstrated

By completing this lab, the following Azure Load Balancer concepts were covered:

- Public Load Balancer
- Standard SKU
- Static Public IP
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
- Layered network troubleshooting