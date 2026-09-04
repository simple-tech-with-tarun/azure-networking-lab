# Azure DNS Private Resolver — Inbound and Outbound

This lab demonstrates how to build and validate **Azure DNS Private Resolver** for both inbound and outbound DNS resolution.

The lab covers two complementary DNS flows:

* **Inbound DNS** — DNS queries entering Azure through an inbound endpoint
* **Outbound DNS** — DNS queries leaving Azure through an outbound endpoint and forwarding ruleset

The lab also demonstrates how DNS Private Resolver works together with:

* Private DNS Zones
* DNS records
* VNet Links
* Forwarding Rulesets
* Forwarding Rules
* VNet Peering
* Private VMs
* NAT Gateway
* Azure platform DNS

The lab ends with successful DNS validation for both inbound and outbound scenarios.

---

# Architecture

```text
                                      Azure
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                            │
│                         Resolver VNet                                      │
│                         10.50.0.0/16                                       │
│                                                                            │
│   ┌────────────────┐   ┌─────────────────┐   ┌────────────────────────┐   │
│   │ inbound-subnet │   │ outbound-subnet │   │     test-subnet        │   │
│   │ 10.50.1.0/28   │   │ 10.50.2.0/28    │   │     10.50.3.0/24       │   │
│   │                │   │                 │   │                        │   │
│   │ Inbound        │   │ Outbound        │   │      Test VM           │   │
│   │ Endpoint       │   │ Endpoint        │   │      Private IP        │   │
│   │ 10.50.1.4      │   │                 │   │      No Public IP      │   │
│   └───────┬────────┘   └────────┬────────┘   └────────────────────────┘   │
│           │                     │                                         │
│           │                     │                                         │
│           │                     ▼                                         │
│           │             Forwarding Ruleset                                │
│           │             az-dns-forwarding-ruleset                         │
│           │                     │                                         │
│           │                     ▼                                         │
│           │               example.com.                                     │
│           │                     │                                         │
│           │                     ▼                                         │
│           │                 8.8.8.8:53                                    │
│           │                                                                │
│           ▼                                                                │
│     Private DNS Zone                                                       │
│       inbound.example                                                      │
│           │                                                                │
│           ▼                                                                │
│   app.inbound.example                                                      │
│       → 10.50.10.10                                                        │
│                                                                            │
└───────────┬────────────────────────────────────────────────────────────────┘
            │
            │ VNet Peering
            │
            ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                         Client VNet                                        │
│                         10.60.0.0/16                                       │
│                                                                            │
│                     ┌────────────────────┐                                 │
│                     │   client-subnet    │                                 │
│                     │   10.60.1.0/24     │                                 │
│                     │                    │                                 │
│                     │    Client VM       │                                 │
│                     │    10.60.1.4       │                                 │
│                     │    No Public IP    │                                 │
│                     └────────────────────┘                                 │
│                                                                            │
│                         NAT Gateway                                        │
│                              │                                             │
│                              ▼                                             │
│                           Internet                                         │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

# What This Lab Demonstrates

The lab demonstrates the two directions supported by Azure DNS Private Resolver.

```text
                         DNS Private Resolver
                                  │
                   ┌──────────────┴──────────────┐
                   │                             │
                INBOUND                       OUTBOUND
                   │                             │
                   ▼                             ▼
          Inbound Endpoint              Outbound Endpoint
                   │                             │
                   ▼                             ▼
              Azure DNS                  Forwarding Ruleset
                   │                             │
                   ▼                             ▼
           Private DNS Zone                External DNS
```

The key distinction is:

```text
Inbound:
DNS → Azure

Outbound:
Azure → External DNS
```

---

# Resources

## Resource Group

```text
az-dns-resolver-lab-rg
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

# Resolver VNet

```text
Name:          az-dns-resolver-vnet
Address space: 10.50.0.0/16
```

The resolver VNet contains three subnets:

```text
inbound-subnet
10.50.1.0/28

outbound-subnet
10.50.2.0/28

test-subnet
10.50.3.0/24
```

The inbound and outbound subnets are delegated to:

```text
Microsoft.Network/dnsResolvers
```

---

# DNS Private Resolver

```text
Name:  az-dns-private-resolver
State: Connected
```

The resolver is associated with:

```text
az-dns-resolver-vnet
```

The resolver provides the managed DNS infrastructure used for both inbound and outbound DNS scenarios.

---

# Inbound DNS

## Inbound Endpoint

```text
Name:       inbound-endpoint
Private IP: 10.50.1.4
Subnet:     inbound-subnet
```

The inbound endpoint provides a private IP address that DNS clients can use to send DNS queries into Azure.

The inbound subnet:

```text
10.50.1.0/28
```

is delegated to:

```text
Microsoft.Network/dnsResolvers
```

---

## Client VNet

A separate VNet was created to simulate a remote DNS client network.

```text
Name:          az-dns-inbound-client-vnet
Address space: 10.60.0.0/16
```

Client subnet:

```text
Name:    client-subnet
Address: 10.60.1.0/24
```

The subnet uses:

```text
defaultOutboundAccess: false
```

---

## VNet Peering

The client VNet and resolver VNet were connected using two-way VNet peering.

Client → Resolver:

```text
client-to-resolver
State: Connected
```

Resolver → Client:

```text
resolver-to-client
State: Connected
```

This provides private connectivity between:

```text
10.60.0.0/16
```

and:

```text
10.50.0.0/16
```

---

# Private DNS Zone

The following Private DNS Zone was created:

```text
inbound.example
```

The zone contains a custom A record:

```text
Name: app
FQDN: app.inbound.example
IP:   10.50.10.10
TTL:  3600
```

Therefore:

```text
app.inbound.example → 10.50.10.10
```

The IP address is a test address and does not represent an actual application.

---

# VNet Link

The Private DNS Zone was linked to:

```text
az-dns-resolver-vnet
```

Link:

```text
resolver-vnet-link
```

Configuration:

```text
Registration enabled: false
Link state:           Completed
```

Automatic registration was disabled because the DNS record was created manually.

The VNet Link makes the Private DNS Zone available to resources in the linked VNet.

---

# Client VM

The client VM was created without a public IP.

```text
Name:       az-dns-inbound-client-vm
Private IP: 10.60.1.4
Public IP:  None
SKU:        Standard_D2s_v5
```

The VM uses an explicitly created NIC:

```text
az-dns-inbound-client-nic
```

This keeps the network configuration explicit and avoids unwanted public IP creation.

---

# Inbound DNS Flow

The inbound DNS flow is:

```text
Client VM
10.60.1.4
     │
     │ DNS query
     ▼
VNet Peering
     │
     ▼
Inbound Endpoint
10.50.1.4
     │
     ▼
Azure DNS
     │
     ▼
Private DNS Zone
inbound.example
     │
     ▼
app.inbound.example
     │
     ▼
10.50.10.10
```

---

# Inbound Validation

A DNS query was sent directly to the inbound endpoint:

```powershell
az vm run-command invoke `
  --resource-group az-dns-resolver-lab-rg `
  --name az-dns-inbound-client-vm `
  --command-id RunShellScript `
  --scripts "dig @10.50.1.4 app.inbound.example +time=3 +tries=1"
```

The query returned:

```text
status: NOERROR

ANSWER: 1

app.inbound.example. 1800 IN A 10.50.10.10

SERVER: 10.50.1.4#53
```

This proves that:

1. The client VNet can reach the resolver VNet.
2. VNet peering is functioning.
3. The client can reach the inbound endpoint.
4. The inbound endpoint accepts DNS queries.
5. Azure DNS can resolve the Private DNS Zone.
6. The expected DNS record is returned.

---

# Outbound DNS

The same DNS Private Resolver was also configured for outbound DNS forwarding.

---

# Outbound Subnet

```text
Name:       outbound-subnet
Address:    10.50.2.0/28
Delegation: Microsoft.Network/dnsResolvers
```

The subnet is dedicated to the DNS Private Resolver outbound endpoint.

---

# Outbound Endpoint

```text
Name: outbound-endpoint
```

The outbound endpoint is associated with:

```text
outbound-subnet
```

The outbound endpoint provides the egress path used by DNS Private Resolver when forwarding DNS queries to external DNS servers.

Unlike the inbound endpoint, the outbound endpoint does not expose a directly assigned private IP address in the same way.

---

# DNS Forwarding Ruleset

```text
Name: az-dns-forwarding-ruleset
```

The forwarding ruleset contains conditional DNS forwarding rules.

Its purpose is to determine:

```text
Which DNS suffix should be forwarded?
            │
            ▼
Which DNS server should receive the query?
```

---

# Forwarding Rule

A forwarding rule was created for:

```text
Domain: example.com.
```

Target:

```text
8.8.8.8:53
```

Therefore:

```text
example.com.
      │
      ▼
8.8.8.8:53
```

The trailing dot represents the fully qualified DNS domain.

---

# Forwarding Ruleset VNet Link

The forwarding ruleset was linked to:

```text
az-dns-resolver-vnet
```

The VNet Link determines which VNets can use the forwarding ruleset.

This is separate from the outbound endpoint.

```text
Forwarding Ruleset
        │
        ├── Forwarding Rules
        │
        └── VNet Link
                │
                ▼
              VNet
```

The outbound endpoint provides the forwarding path, while the VNet Link makes the forwarding rules available to DNS clients in the linked VNet.

---

# Outbound Test VM

The outbound scenario used a private test VM in:

```text
test-subnet
10.50.3.0/24
```

The VM was configured without a public IP.

The subnet uses:

```text
defaultOutboundAccess: false
```

This prevents the test from relying on implicit outbound internet connectivity.

---

# NAT Gateway

A NAT Gateway was configured to provide controlled outbound connectivity.

The NAT Gateway provides:

* Explicit outbound internet access
* Predictable public source IP
* No public IP directly on the VM
* Connectivity required for Azure Run Command

The NAT Gateway is **not the DNS forwarding mechanism**.

The DNS forwarding path is:

```text
Test VM
   │
   ▼
Azure Platform DNS
168.63.129.16
   │
   ▼
DNS Private Resolver
   │
   ▼
Outbound Endpoint
   │
   ▼
Forwarding Ruleset
   │
   ▼
8.8.8.8:53
```

---

# Outbound DNS Flow

The complete outbound DNS flow is:

```text
Private Test VM
      │
      │ DNS Query
      ▼
Azure Platform DNS
168.63.129.16
      │
      ▼
DNS Private Resolver
      │
      ▼
Outbound Endpoint
      │
      ▼
Forwarding Ruleset
      │
      │ example.com.
      ▼
8.8.8.8:53
      │
      ▼
DNS Response
```

---

# Outbound Validation

From the private test VM:

```powershell
az vm run-command invoke `
  --resource-group az-dns-resolver-lab-rg `
  --name az-dns-resolver-test-vm `
  --command-id RunShellScript `
  --scripts "dig @168.63.129.16 example.com"
```

The query successfully returned DNS records for:

```text
example.com
```

The DNS server shown by the VM was:

```text
168.63.129.16
```

This is the Azure platform DNS address used by Azure VMs.

The important forwarding path occurs behind this Azure DNS interface.

---

# Controlled Failure Test

To validate the forwarding configuration, the forwarding target was temporarily changed from:

```text
8.8.8.8
```

to the documentation-only TEST-NET address:

```text
192.0.2.1
```

The DNS query was then tested again.

The query timed out.

The forwarding target was restored to:

```text
8.8.8.8:53
```

A subsequent DNS query succeeded again.

This controlled failure/recovery test provides practical evidence that the forwarding rule and target configuration were active.

---

# Inbound vs Outbound

The two resolver scenarios solve opposite DNS problems.

## Inbound

Inbound allows DNS clients to query Azure DNS through the resolver.

```text
Remote DNS Client
        │
        ▼
Inbound Endpoint
        │
        ▼
Azure DNS
        │
        ▼
Private DNS Zone
```

Example:

```text
app.inbound.example
        │
        ▼
10.50.10.10
```

---

## Outbound

Outbound allows Azure DNS queries to be forwarded to external DNS servers.

```text
Azure DNS Client
       │
       ▼
Outbound Endpoint
       │
       ▼
Forwarding Ruleset
       │
       ▼
External DNS
```

Example:

```text
example.com.
      │
      ▼
8.8.8.8:53
```

---

# Inbound vs Outbound Summary

```text
                    DNS Private Resolver
                             │
               ┌─────────────┴─────────────┐
               │                           │
            INBOUND                     OUTBOUND
               │                           │
               ▼                           ▼
      Inbound Endpoint             Outbound Endpoint
               │                           │
               ▼                           ▼
          Azure DNS                Forwarding Ruleset
               │                           │
               ▼                           ▼
       Private DNS Zone              External DNS
```

The easiest way to remember the difference:

```text
Inbound:
DNS → Azure

Outbound:
Azure → External DNS
```

---

# Private DNS Zone vs DNS Private Resolver

These components solve different problems.

## Private DNS Zone

A Private DNS Zone stores private DNS records.

```text
Private DNS Zone
       │
       ▼
app.inbound.example
       │
       ▼
10.50.10.10
```

## DNS Private Resolver

DNS Private Resolver provides managed DNS resolution and forwarding capabilities.

```text
DNS Private Resolver
        │
        ├── Inbound Endpoint
        │
        └── Outbound Endpoint
```

Therefore:

```text
Private DNS Zone
        ≠
DNS Private Resolver
```

They can work together but are not replacements for each other.

---

# VNet Link vs Forwarding Ruleset

These concepts are easy to confuse.

## Private DNS Zone VNet Link

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

## Forwarding Ruleset VNet Link

Answers:

> Which VNet can use these DNS forwarding rules?

```text
Forwarding Ruleset
       │
       ▼
   VNet Link
       │
       ▼
      VNet
```

They serve different DNS components.

---

# Inbound Endpoint vs Outbound Endpoint

## Inbound Endpoint

Provides a private DNS endpoint that clients can query.

```text
Client
  │
  ▼
Inbound Endpoint
  │
  ▼
Azure DNS
```

It has a private IP address.

Example:

```text
10.50.1.4
```

## Outbound Endpoint

Provides the egress path for DNS forwarding.

```text
Azure DNS
    │
    ▼
Outbound Endpoint
    │
    ▼
External DNS
```

The outbound endpoint does not have a directly assigned IP address in the same way as an inbound endpoint.

---

# NAT Gateway vs DNS Private Resolver

NAT Gateway and DNS Private Resolver solve completely different problems.

## NAT Gateway

```text
Private VM
    │
    ▼
NAT Gateway
    │
    ▼
Internet
```

Provides outbound network connectivity.

## DNS Private Resolver

```text
DNS Client
    │
    ▼
DNS Private Resolver
    │
    ▼
DNS Server
```

Provides DNS resolution and forwarding.

Therefore:

```text
NAT Gateway
      ≠
DNS Private Resolver
```

The NAT Gateway does not determine where DNS queries are forwarded.

---

# Important Networking Concept

**DNS resolution does not provide network connectivity.**

For example:

```text
app.inbound.example
        │
        ▼
10.50.10.10
```

only provides an IP address.

Actual connectivity depends on:

* VNet addressing
* Subnets
* Routing
* VNet Peering
* Network security controls
* The destination service
* The destination port

The DNS layer and network layer are separate.

```text
DNS
 │
 └── hostname → IP
                │
                ▼
           Network Path
                │
                ▼
        Destination Service
```

---

# Why `168.63.129.16` Appears in Azure

Azure VMs commonly use:

```text
168.63.129.16
```

as the Azure platform DNS address.

Therefore an outbound test can look like:

```text
dig @168.63.129.16 example.com
```

The VM sends the query to the Azure platform DNS service.

The configured DNS forwarding behavior then determines how matching DNS queries are handled.

The address:

```text
168.63.129.16
```

does **not** mean that the VM is directly querying `8.8.8.8`.

The external forwarding happens through the Azure DNS Private Resolver configuration.

---

# Key Takeaways

1. **DNS Private Resolver** provides managed DNS resolution and forwarding capabilities.
2. An **Inbound Endpoint** allows DNS queries to enter Azure.
3. An **Outbound Endpoint** provides the egress path for DNS forwarding.
4. A **Private DNS Zone** stores private DNS records.
5. A **DNS Record** maps a hostname to an IP address.
6. A **Private DNS Zone VNet Link** makes a Private DNS Zone available to a VNet.
7. A **Forwarding Ruleset** contains conditional DNS forwarding rules.
8. A **Forwarding Ruleset VNet Link** makes those forwarding rules available to a VNet.
9. An **Inbound Endpoint** has a private IP address.
10. An **Outbound Endpoint** is associated with a dedicated subnet but does not expose a directly assigned IP in the same way as an inbound endpoint.
11. VNet Peering can provide connectivity between a remote client VNet and the resolver VNet.
12. NAT Gateway provides controlled outbound network connectivity but does not perform DNS forwarding.
13. `168.63.129.16` is the Azure platform DNS address used by Azure VMs.
14. DNS resolution and network connectivity are separate concerns.
15. Inbound and outbound DNS resolver configurations solve complementary hybrid DNS requirements.
16. Controlled failure testing can provide practical evidence that DNS forwarding configuration is active.

---

# Cleanup

When the lab is complete, delete the entire resource group:

```powershell
az group delete `
  --name az-dns-resolver-lab-rg `
  --yes `
  --no-wait
```

This removes:

* DNS Private Resolver
* Inbound Endpoint
* Outbound Endpoint
* Forwarding Ruleset
* Forwarding Rules
* Private DNS Zone
* VNet Links
* Resolver VNet
* Client VNet
* VNet Peering
* Subnets
* Test VMs
* NICs
* NAT Gateway
* NAT Public IP
* Other resources created inside the lab resource group

---

# Lab Status

**Completed** ✅

This lab successfully demonstrated both major DNS Private Resolver patterns:

```text
INBOUND
Remote DNS Client
       │
       ▼
Inbound Endpoint
       │
       ▼
Azure Private DNS
```

and:

```text
OUTBOUND
Azure DNS Client
       │
       ▼
Outbound Endpoint
       │
       ▼
Forwarding Ruleset
       │
       ▼
External DNS
```

Together, these patterns form the foundation for **hybrid DNS architectures in Azure**.