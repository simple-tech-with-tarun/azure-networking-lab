# Azure Private DNS — Custom Internal DNS

This lab demonstrates how to build and validate a **custom Azure Private DNS namespace** for internal application name resolution.

Unlike a Private Endpoint lab, this setup does **not** use a Private Endpoint or Private DNS Zone Group. Instead, it focuses on the fundamental building blocks of Azure Private DNS:

* Private DNS Zone
* DNS Record
* VNet Link
* Private network connectivity

The lab ends with a real HTTP request from one VM to another using the custom DNS name.

---

## Architecture

```text
                           Azure VNet
                    10.40.0.0/16
                           │
             ┌─────────────┴─────────────┐
             │                           │
       Test Subnet                 dns-test-subnet
       10.40.1.0/24                     │
             │                           │
             │                 ┌─────────┴─────────┐
             │                 │                   │
      Test VM                  │              App VM
      10.40.1.4                │              10.40.1.10
             │                 │                   │
             │                 │            Python HTTP
             │                 │              :8080
             │                 │                   │
             └─────────┬───────┴───────────────────┘
                       │
                  VNet Link
                       │
                       ▼
              Private DNS Zone
                internal.example
                       │
                       └── app → 10.40.1.10
```

Both VMs use private IP addresses and have **no public IP addresses**.

---

## What This Lab Demonstrates

The lab demonstrates the relationship between:

```text
Private DNS Zone
        │
        ├── DNS Record
        │
        └── VNet Link
                │
                ▼
               VNet
```

The final application flow is:

```text
Test VM
   │
   │ http://app.internal.example:8080
   ▼
Private DNS
   │
   │ app.internal.example → 10.40.1.10
   ▼
App VM
   │
   │ TCP 8080
   ▼
Python HTTP Server
```

---

## Resources

### Resource Group

```text
az-private-dns-custom-lab-rg
```

Location:

```text
centralindia
```

Tags:

```text
owner=tarun
AutoDelete=Yes
```

> `AutoDelete=Yes` is a resource tag only. It does not automatically delete the resource group without separate automation.

---

### Virtual Network

```text
Name:          az-private-dns-custom-vnet
Address space: 10.40.0.0/16
```

Subnet:

```text
Name:          dns-test-subnet
Address space: 10.40.1.0/24
```

The subnet was configured with:

```text
defaultOutboundAccess: false
```

This keeps the lab focused on private networking rather than relying on implicit outbound internet access.

---

## Private DNS Zone

```text
Zone: internal.example
```

Azure Private DNS zones provide a private DNS namespace that can be used by resources connected to associated VNets.

The zone itself does not provide network connectivity. It provides **name resolution**.

---

## DNS Record

A custom A record was created:

```text
Name: app
FQDN: app.internal.example
IP:   10.40.1.10
TTL:  300
```

Therefore:

```text
app.internal.example → 10.40.1.10
```

The IP address was intentionally chosen before the application VM was created so that the DNS record could point to a real private application endpoint.

---

## VNet Link

The Private DNS Zone was linked to:

```text
az-private-dns-custom-vnet
```

Configuration:

```text
Registration enabled: false
Link state:           Completed
```

The VNet Link makes the Private DNS zone available for name resolution from resources in the associated VNet.

Automatic registration was disabled because this lab uses manually created custom DNS records.

---

## Test VM

```text
Name:       az-private-dns-test-vm
Private IP: 10.40.1.4
Public IP:  None
```

The VM was created using an explicitly created NIC:

```text
NIC: az-private-dns-test-nic
```

This was done to keep the networking configuration explicit and prevent Azure from creating an unwanted public IP.

---

## Application VM

```text
Name:       az-private-dns-app-vm
Private IP: 10.40.1.10
Public IP:  None
```

NIC:

```text
az-private-dns-app-nic
```

The NIC was configured with a static private IP:

```text
10.40.1.10
```

This matches the DNS record:

```text
app.internal.example → 10.40.1.10
```

---

## Application

A simple Python HTTP server was started on the application VM:

```bash
python3 -m http.server 8080 --bind 10.40.1.10
```

The application therefore listens on:

```text
10.40.1.10:8080
```

---

## Validation

### 1. Validate DNS resolution

From the test VM:

```powershell
az vm run-command invoke `
  --resource-group az-private-dns-custom-lab-rg `
  --name az-private-dns-test-vm `
  --command-id RunShellScript `
  --scripts "getent hosts app.internal.example"
```

Result:

```text
10.40.1.10    app.internal.example
```

This proves that the test VM can resolve the custom Private DNS name through the VNet Link.

---

### 2. Validate application connectivity

From the test VM:

```powershell
az vm run-command invoke `
  --resource-group az-private-dns-custom-lab-rg `
  --name az-private-dns-test-vm `
  --command-id RunShellScript `
  --scripts "curl -sS --max-time 10 http://app.internal.example:8080"
```

The request returned the Python HTTP server's directory listing.

This proves the complete path:

```text
DNS resolution
      ↓
10.40.1.10
      ↓
VNet routing
      ↓
TCP 8080
      ↓
HTTP application
```

---

# Private DNS vs Private Endpoint DNS

A key learning from this lab is that **Private DNS and Private Endpoints are separate concepts**.

## Standalone Private DNS

The basic setup is:

```text
Private DNS Zone
       │
       ├── DNS Record
       │
       └── VNet Link
```

For example:

```text
app.internal.example → 10.40.1.10
```

No Private Endpoint is required.

---

## Private Endpoint + Private DNS

When a Private Endpoint is used, an additional integration component can be used:

```text
Private Endpoint
       │
       ▼
Private DNS Zone Group
       │
       ▼
Private DNS Zone
       │
       ▼
DNS Record
```

The **Private DNS Zone Group** connects the Private Endpoint with the Private DNS Zone and allows Azure to manage the DNS integration for the endpoint.

It is not a replacement for the DNS zone or DNS record.

---

## VNet Link vs DNS Zone Group

These two are easy to confuse.

### VNet Link

Answers:

> Which VNet can use this Private DNS Zone?

```text
Private DNS Zone
       │
       ▼
VNet Link
       │
       ▼
VNet
```

### Private DNS Zone Group

Answers:

> Which Private DNS Zone is associated with this Private Endpoint?

```text
Private Endpoint
       │
       ▼
Zone Group
       │
       ▼
Private DNS Zone
```

---

# Important Networking Concept

**DNS does not provide network connectivity.**

The DNS record:

```text
app.internal.example → 10.40.1.10
```

only provides the destination IP address.

The actual application connection depends on:

* VNet addressing
* Subnet configuration
* Routing
* Network security controls
* The application listening on the destination port

This lab demonstrates both layers:

```text
DNS
 │
 └── app.internal.example → 10.40.1.10
                              │
                              ▼
                         Network path
                              │
                              ▼
                       HTTP :8080
```

---

# Key Takeaways

1. A **Private DNS Zone** provides a private DNS namespace.
2. A **DNS record** maps a hostname to an IP address.
3. A **VNet Link** makes the zone available to a VNet.
4. A Private DNS Zone does **not** require a Private Endpoint.
5. A **Private DNS Zone Group** is used for Private Endpoint DNS integration.
6. DNS resolution and network connectivity are separate concerns.
7. Private-only VMs can communicate using custom DNS names without public IP addresses.
8. Explicit NIC creation provides tighter control over private IP allocation and public IP exposure.

---

# Cleanup

When the lab is complete, delete the entire resource group:

```powershell
az group delete `
  --name az-private-dns-custom-lab-rg `
  --yes `
  --no-wait
```

This removes:

* Virtual Network
* Subnet
* Private DNS Zone
* VNet Link
* Test VM
* Application VM
* NICs
* Other resources created inside the lab resource group
