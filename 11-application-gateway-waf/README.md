# Azure Application Gateway + WAF + Path-Based Routing

This lab demonstrates how to deploy and configure an **Azure Application Gateway with Web Application Firewall (WAF)** and **path-based routing** to expose multiple private backend applications through a single public endpoint.

The lab uses two backend virtual machines hosted on a private subnet. Azure Application Gateway provides the public entry point, distributes requests based on URL paths, performs backend health monitoring, and protects the applications using an OWASP-based WAF policy.

---

## Architecture

```text
                         Internet
                            |
                            | HTTP :80
                            v
                 +-----------------------+
                 |  Public IP            |
                 | 40.80.85.137          |
                 +-----------+-----------+
                             |
                             v
                 +-----------------------+
                 | Azure Application     |
                 | Gateway               |
                 | az-appgw-waf          |
                 | WAF_v2                |
                 +-----------+-----------+
                             |
                    WAF Inspection
                             |
                 +-----------+-----------+
                 |                       |
             "/" |                       | "/app2/*"
                 |                       |
                 v                       v
       +-------------------+    +-------------------+
       | app1BackendPool   |    | app2BackendPool   |
       | 10.90.2.4         |    | 10.90.2.5         |
       +---------+---------+    +---------+---------+
                 |                        |
                 v                        v
       +-------------------+    +-------------------+
       | Backend VM 1      |    | Backend VM 2      |
       | App 1             |    | App 2             |
       | 10.90.2.4:8080    |    | 10.90.2.5:8080    |
       +-------------------+    +-------------------+
```

### Traffic flow

```text
http://40.80.85.137/
        |
        +--> WAF
              |
              +--> Default route
                    |
                    +--> App 1
                         10.90.2.4:8080


http://40.80.85.137/app2/
        |
        +--> WAF
              |
              +--> /app2/* path rule
                    |
                    +--> App 2
                         10.90.2.5:8080
```

A malicious request matching the WAF's managed rules is blocked before it reaches the backend.

---

## Objectives

By completing this lab, the following Azure networking and application-delivery concepts are demonstrated:

- Azure Application Gateway
- Application Gateway WAF_v2
- Azure Web Application Firewall
- OWASP managed WAF rules
- WAF Prevention mode
- Path-based routing
- Backend address pools
- Backend HTTP settings
- Custom health probes
- Private backend virtual machines
- Application Gateway backend health monitoring
- Public-to-private application publishing
- Basic WAF validation

---

## Azure Resources

| Resource | Name | Configuration |
|---|---|---|
| Resource Group | `az-appgw-waf-lab-rg` | Central India |
| Virtual Network | `az-appgw-waf-vnet` | `10.90.0.0/16` |
| Application Gateway Subnet | `appgw-subnet` | `10.90.1.0/24` |
| Backend Subnet | `backend-subnet` | `10.90.2.0/24` |
| Application Gateway | `az-appgw-waf` | WAF_v2, capacity 2 |
| Public IP | `az-appgw-waf-pip` | Standard, Static |
| WAF Policy | `az-appgw-waf-policy` | OWASP 3.2, Prevention |
| Backend VM 1 | `az-appgw-backend1-vm` | `10.90.2.4` |
| Backend VM 2 | `az-appgw-backend2-vm` | `10.90.2.5` |
| Backend Pool 1 | `app1BackendPool` | `10.90.2.4` |
| Backend Pool 2 | `app2BackendPool` | `10.90.2.5` |
| Backend Probe 1 | `app1BackendProbe` | HTTP `/` |
| Backend Probe 2 | `app2BackendProbe` | HTTP `/` |

All resources were created in **Central India** and tagged:

```text
owner=tarun
AutoDelete=Yes
```

---

# 1. Prerequisites

The following are required:

- Azure subscription
- Azure CLI
- PowerShell
- SSH key
- Sufficient Azure permissions to create networking and compute resources

Verify Azure CLI:

```powershell
az version
```

Verify the active subscription:

```powershell
az account show
```

---

# 2. Create the Resource Group

```powershell
az group create `
  --name az-appgw-waf-lab-rg `
  --location centralindia `
  --tags owner=tarun AutoDelete=Yes
```

---

# 3. Create the Virtual Network

The lab uses a single VNet with separate subnets for the Application Gateway and backend workloads.

```powershell
az network vnet create `
  --resource-group az-appgw-waf-lab-rg `
  --name az-appgw-waf-vnet `
  --location centralindia `
  --address-prefix 10.90.0.0/16 `
  --subnet-name appgw-subnet `
  --subnet-prefix 10.90.1.0/24
```

Create the backend subnet:

```powershell
az network vnet subnet create `
  --resource-group az-appgw-waf-lab-rg `
  --vnet-name az-appgw-waf-vnet `
  --name backend-subnet `
  --address-prefixes 10.90.2.0/24
```

The Application Gateway subnet is dedicated to Application Gateway.

The backend subnet contains the private application servers.

---

# 4. Backend Virtual Machines

Two Ubuntu virtual machines are used as simple backend applications.

## Backend VM 1

Private IP:

```text
10.90.2.4
```

Application:

```text
App 1
```

Response:

```text
Hello from Backend VM 1 - 10.90.2.4
```

## Backend VM 2

Private IP:

```text
10.90.2.5
```

Application:

```text
App 2
```

Response:

```text
Application 2 - Admin Portal - Backend VM 10.90.2.5
```

Both applications run a simple Python HTTP server:

```powershell
python3 -m http.server 8080 --directory /opt/appgw --bind 0.0.0.0
```

The applications therefore listen on:

```text
TCP 8080
```

The backend VMs do **not** require public IP addresses because Application Gateway provides the public entry point.

---

# 5. Application Gateway Public IP

The Application Gateway uses a Standard Static public IP:

```text
az-appgw-waf-pip
```

Final public IP:

```text
40.80.85.137
```

The public listener accepts HTTP traffic on:

```text
TCP 80
```

---

# 6. WAF Policy

The Web Application Firewall policy is:

```text
az-appgw-waf-policy
```

Configuration:

```text
Type:             OWASP
Rule set:         OWASP 3.2
Mode:             Prevention
State:            Enabled
Request body:     Enabled
```

The policy is attached directly to the Application Gateway.

### WAF mode

The lab uses **Prevention** mode.

In Prevention mode, requests matching enabled WAF rules can be blocked instead of merely logged.

During policy creation Azure also reported an automatically computed disabled rule associated with the managed **Known-CVEs** rule group. This was retained as part of the managed WAF configuration.

---

# 7. Application Gateway

Application Gateway:

```text
az-appgw-waf
```

SKU:

```text
WAF_v2
```

Capacity:

```text
2
```

The Application Gateway is deployed into:

```text
appgw-subnet
10.90.1.0/24
```

The backend servers reside in:

```text
backend-subnet
10.90.2.0/24
```

---

# 8. Backend Pools

Separate backend pools are used for each application.

## App 1

```text
app1BackendPool
    |
    +--> 10.90.2.4
```

## App 2

```text
app2BackendPool
    |
    +--> 10.90.2.5
```

This separation makes the path-routing configuration explicit and allows each application to have its own backend settings and health probe.

---

# 9. Backend HTTP Settings

Each backend application has dedicated HTTP settings.

## App 1

```text
Name:              app1BackendHttpSettings
Protocol:          HTTP
Port:              8080
Cookie affinity:   Disabled
Timeout:            30 seconds
Probe:             app1BackendProbe
```

## App 2

```text
Name:              app2BackendHttpSettings
Protocol:          HTTP
Port:              8080
Cookie affinity:   Disabled
Timeout:            30 seconds
Probe:             app2BackendProbe
```

---

# 10. Backend Health Probes

Dedicated probes are configured for each backend.

## App 1 Probe

```text
app1BackendProbe
```

Configuration:

```text
Protocol:          HTTP
Host:              10.90.2.4
Path:              /
Interval:          30 seconds
Timeout:           30 seconds
Unhealthy threshold: 3
Expected status:   200-399
```

## App 2 Probe

```text
app2BackendProbe
```

Configuration:

```text
Protocol:          HTTP
Host:              10.90.2.5
Path:              /
Interval:          30 seconds
Timeout:           30 seconds
Unhealthy threshold: 3
Expected status:   200-399
```

Using dedicated probes keeps each application's health monitoring independent.

---

# 11. Path-Based Routing

The Application Gateway uses a path-based routing rule:

```text
rule1
```

Routing type:

```text
PathBasedRouting
```

The URL path map is:

```text
appPathMap
```

### Default route

Requests that do not match a specific path rule are sent to App 1:

```text
/  --> app1BackendPool
```

### App 2 route

Requests matching:

```text
/app2/*
```

are sent to:

```text
app2BackendPool
```

Therefore:

| Request | Destination |
|---|---|
| `/` | `10.90.2.4:8080` |
| `/anything` | `10.90.2.4:8080` |
| `/app2/` | `10.90.2.5:8080` |
| `/app2/test` | `10.90.2.5:8080` |

The important point is that **the same public IP and listener expose both applications**.

---

# 12. Verify Application Gateway Configuration

Check the Application Gateway:

```powershell
az network application-gateway show `
  --resource-group az-appgw-waf-lab-rg `
  --name az-appgw-waf `
  --query "{name:name,sku:sku.name,provisioningState:provisioningState,operationalState:operationalState,wafPolicy:firewallPolicy.id}" `
  --output json
```

Final result:

```text
name:               az-appgw-waf
sku:                WAF_v2
operationalState:   Running
provisioningState:  Succeeded
wafPolicy:          az-appgw-waf-policy
```

This confirms that the Application Gateway is operational and the WAF policy is attached.

---

# 13. Verify Backend Health

Run:

```powershell
az network application-gateway show-backend-health `
  --resource-group az-appgw-waf-lab-rg `
  --name az-appgw-waf `
  --query "backendAddressPools[].{pool:backendAddressPool.id,servers:backendHttpSettingsCollection[].servers[].{address:address,health:health}}" `
  --output json
```

Final backend health:

```text
10.90.2.4  Healthy
10.90.2.5  Healthy
```

Both backend applications are therefore reachable from Application Gateway and passing their health probes.

---

# 14. Test App 1 Routing

The Application Gateway public IP is:

```text
40.80.85.137
```

Test the default route:

```powershell
curl.exe -i http://40.80.85.137/
```

Expected response:

```text
HTTP/1.1 200 OK
```

Response body:

```text
Hello from Backend VM 1 - 10.90.2.4
```

This confirms that the default path is routed to App 1.

Repeated requests continued to return the App 1 response, confirming that the default route is associated with the App 1 backend pool.

---

# 15. Test App 2 Path-Based Routing

Test:

```powershell
curl.exe -i http://40.80.85.137/app2/
```

Expected response:

```text
HTTP/1.1 200 OK
```

Response body:

```text
Application 2 - Admin Portal - Backend VM 10.90.2.5
```

This confirms that:

```text
/app2/*
```

is routed to:

```text
10.90.2.5:8080
```

---

# 16. Test WAF Protection

A SQL-injection-style query string was sent to the App 2 endpoint:

```powershell
curl.exe -i "http://40.80.85.137/app2/?id=1%27%20OR%20%271%27%3D%271"
```

The Application Gateway returned:

```text
HTTP/1.1 403 Forbidden
```

The response identified the Application Gateway:

```text
Server: Microsoft-Azure-Application-Gateway/v2
```

The important result is:

```text
403 Forbidden
```

This demonstrates that the WAF detected and blocked the malicious request before it reached the backend application.

---

# 17. Final Validation

The completed environment was verified with the following state:

```text
Application Gateway
-------------------
Name:               az-appgw-waf
SKU:                WAF_v2
Operational state:  Running
Provisioning state: Succeeded
WAF policy:         az-appgw-waf-policy


Backend health
--------------
App 1:              10.90.2.4  Healthy
App 2:              10.90.2.5  Healthy


Routing
-------
/                   -> App 1
/app2/*             -> App 2


WAF
---
Normal application request -> 200 OK
SQL-injection-style test   -> 403 Forbidden
```

---

# 18. Configuration Cleanup

During the lab, the original shared backend pool, HTTP settings, and probe were replaced with dedicated resources.

The final Application Gateway configuration contains only:

```text
Backend pools
-------------
app1BackendPool
app2BackendPool


HTTP settings
-------------
app1BackendHttpSettings
app2BackendHttpSettings


Health probes
-------------
app1BackendProbe
app2BackendProbe
```

The obsolete shared resources were removed after confirming that the active routing configuration no longer referenced them.

This is important because it leaves the Application Gateway configuration clean rather than retaining unused resources from the initial configuration.

---

# 19. Cleanup Azure Resources

When the lab is no longer required, delete the entire resource group:

```powershell
az group delete `
  --name az-appgw-waf-lab-rg `
  --yes `
  --no-wait
```

This removes the Application Gateway, WAF policy, public IP, virtual machines, NICs, VNet, subnets, and associated resources.

Verify deletion:

```powershell
az group show `
  --name az-appgw-waf-lab-rg
```

A successful deletion results in the resource group no longer being found.

---

# 20. Key Learning Outcomes

This lab demonstrates an important Azure application-delivery pattern:

```text
Public Internet
      |
      v
Application Gateway
      |
      +---- WAF inspection
      |
      +---- Path-based routing
              |
              +---- Application 1
              |
              +---- Application 2
```

The backend servers do not need individual public IP addresses.

Instead, Application Gateway acts as the controlled public entry point while providing:

- Layer 7 HTTP routing
- URL path-based routing
- Backend health monitoring
- WAF protection
- Centralized application publishing
- Separation between public and private network components

The lab also demonstrates why dedicated backend pools, HTTP settings, and health probes are useful when an Application Gateway hosts multiple applications.

---

## Technologies Used

- Azure Virtual Network
- Azure Application Gateway
- Azure Web Application Firewall
- WAF_v2
- OWASP Core Rule Set 3.2
- Path-based routing
- Azure Virtual Machines
- Azure CLI
- PowerShell
- Ubuntu Linux
- Python HTTP Server

---

## Repository

This lab is part of the Azure networking labs repository and is implemented on the:

```text
feat/application-gateway-waf
```

branch.

---

## Summary

The final environment provides two private backend applications through one public Application Gateway endpoint:

```text
http://40.80.85.137/
        |
        +--> App 1
             10.90.2.4:8080


http://40.80.85.137/app2/
        |
        +--> App 2
             10.90.2.5:8080
```

Both backends are healthy, routing is handled by Application Gateway, and WAF protection successfully blocks a SQL-injection-style request with HTTP `403 Forbidden`.

This provides a practical foundation for understanding how **Azure Application Gateway, WAF, private backend workloads, health probes, and Layer 7 path-based routing** work together in a real-world architecture.