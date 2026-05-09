# Complete Azure App Service Domain, Azure DNS, Front Door and App Gateway Flow Guide

# 1. Introduction

This document explains the complete end-to-end flow for:

* Buying custom domains in Azure
* Azure App Service Domains
* Azure DNS Zones
* Name servers
* DNS records
* CNAME configuration
* Azure Front Door integration
* Application Gateway integration
* AKS ingress routing
* DNS resolution flow
* SSL/TLS setup
* Real production architecture
* Interview concepts

This guide is useful for:

* Azure networking interviews
* DevOps interviews
* AKS production architecture
* DNS understanding
* Front Door and App Gateway configuration

---

# 2. Important Azure Services Difference

| Azure Service            | Purpose                    |
| ------------------------ | -------------------------- |
| Azure App Service Domain | Buy/manage internet domain |
| Azure DNS Zone           | Host/manage DNS records    |
| Azure Front Door         | Global routing and CDN     |
| Application Gateway      | Regional Layer 7 ingress   |
| Azure AD Domain Services | LDAP/Kerberos/domain join  |

---

# 3. What is Azure App Service Domain?

Azure App Service Domain allows purchasing internet domains directly from Azure portal.

Example:

```text
contoso.com
```

Azure internally works with registrars.

Benefits:

* Easier Azure integration
* Centralized management
* Easier DNS configuration
* Easy SSL integration

---

# 4. End-to-End Production Architecture

```text
User Browser
      |
app.contoso.com
      |
Azure DNS Zone
      |
CNAME --> myapp.azurefd.net
      |
Azure Front Door
      |
Application Gateway
      |
AKS Ingress
      |
Service
      |
Pods
```

---

# 5. Step 1 - Purchase Domain in Azure

In Azure Portal:

```text
Azure Portal
   |
App Service Domains
   |
Purchase Domain
```

Example:

```text
contoso.com
```

Azure registers and manages the domain.

---

# 6. Step 2 - Create Azure DNS Zone

Create:

```text
Azure DNS Zone
```

Example:

```text
contoso.com
```

Azure automatically creates:

```text
ns1-01.azure-dns.com
ns2-01.azure-dns.net
ns3-01.azure-dns.org
ns4-01.azure-dns.info
```

These are authoritative name servers.

---

# 7. What are Name Servers?

Name servers answer DNS queries for your domain.

Example:

```text
Who knows about contoso.com?
```

Azure DNS name servers respond with DNS records.

---

# 8. How Name Servers Work

```text
Client Browser
      |
Recursive DNS Resolver
      |
Root DNS Server
      |
TLD Server (.com)
      |
Azure DNS Name Servers
      |
DNS Records Returned
```

---

# 9. Step 3 - Link Domain with Azure DNS

Because domain is purchased within Azure ecosystem:

Azure can automatically or semi-automatically configure:

* NS delegation
* DNS management

Azure DNS becomes authoritative DNS provider.

---

# 10. Step 4 - Create Azure Front Door

Create:

```text
Azure Front Door
```

Azure Front Door generates default endpoint:

```text
myapp.azurefd.net
```

---

# 11. What Azure Front Door Does

Azure Front Door provides:

* Global routing
* CDN
* WAF
* SSL offloading
* Multi-region failover
* Health probes
* Latency-based routing

---

# 12. Step 5 - Configure Front Door Backend

Backend can be:

* Application Gateway
* AKS ingress
* App Service
* VM
* Internal Load Balancer

Production pattern:

```text
Front Door
    |
Application Gateway
    |
AKS
```

---

# 13. Step 6 - Add Application Gateway as Backend Origin

Inside Front Door:

```text
Origin Group
   |
Add Origin
```

Backend example:

```text
appgw-eastus.contoso.com
```

OR

```text
20.50.10.5
```

---

# 14. How Front Door Reaches App Gateway

Flow:

```text
Client
  |
Azure Front Door
  |
Application Gateway Public IP/FQDN
  |
AKS Ingress
  |
Pods
```

---

# Important Concept

Front Door is global.

App Gateway is regional.

---

# 15. Front Door vs App Gateway

| Front Door            | App Gateway              |
| --------------------- | ------------------------ |
| Global service        | Regional service         |
| CDN                   | Ingress                  |
| Multi-region failover | Backend balancing        |
| Edge POPs             | Regional Layer 7 routing |
| Global WAF            | Regional WAF             |

---

# 16. Step 7 - Add Custom Domain in Front Door

Add:

```text
app.contoso.com
```

inside Front Door custom domain configuration.

---

# 17. Step 8 - Create CNAME Record

Inside Azure DNS Zone:

Create:

```text
app.contoso.com
```

pointing to:

```text
myapp.azurefd.net
```

---

# 18. CNAME Flow

```text
app.contoso.com
      |
CNAME
      |
myapp.azurefd.net
      |
Front Door IP
```

---

# 19. Why Use CNAME?

Benefits:

* Front Door IPs may change
* Easier management
* Dynamic backend handling
* Better Azure integration

---

# 20. Difference Between A Record and CNAME

| A Record             | CNAME             |
| -------------------- | ----------------- |
| Domain --> IP        | Domain --> Domain |
| Static mapping       | Alias mapping     |
| Direct IP dependency | Dynamic target    |

---

# 21. Step 9 - Domain Validation

Azure Front Door asks for TXT validation.

Example:

```text
asuid.app --> validation-token
```

Used to verify ownership.

---

# 22. Step 10 - Enable HTTPS

Azure Front Door can:

* Generate managed certificate
* Automatically renew SSL

HTTPS enabled for:

```text
https://app.contoso.com
```

---

# 23. Detailed DNS Resolution Flow

```text
Browser
  |
Local DNS Cache
  |
ISP Recursive Resolver
  |
Root DNS Server
  |
TLD Server (.com)
  |
Azure DNS Name Server
  |
CNAME Returned
  |
myapp.azurefd.net
  |
Front Door IP Returned
```

---

# 24. How Client Finally Reaches Application

```text
User Browser
      |
app.contoso.com
      |
Azure DNS
      |
Azure Front Door
      |
Application Gateway
      |
AKS Ingress
      |
Service
      |
Pods
```

---

# 25. How App Gateway Reaches AKS

Application Gateway integrates with AKS using:

```text
AGIC
```

AGIC = Application Gateway Ingress Controller.

---

# 26. AGIC Responsibilities

AGIC dynamically updates:

* Backend pools
* Routing rules
* Health probes
* SSL listeners
* URL path rules

---

# 27. App Gateway to AKS Flow

```text
Application Gateway
      |
Ingress Rules
      |
Kubernetes Service
      |
Pods
```

---

# 28. Real Enterprise Production Architecture

```text
Global Users
      |
Azure Front Door
      |
Regional Application Gateway
      |
AKS Cluster
      |
Pods
      |
Private Endpoint
      |
Azure SQL / Storage / Key Vault
```

---

# 29. Public DNS vs Private DNS

| Public DNS      | Private DNS                      |
| --------------- | -------------------------------- |
| Internet-facing | Internal only                    |
| app.contoso.com | privatelink.database.windows.net |
| Used by clients | Used inside VNets                |

---

# 30. How Internal AKS DNS Works

Inside AKS:

```text
backend.default.svc.cluster.local
```

resolved using:

```text
CoreDNS
```

External Azure DNS is separate from Kubernetes DNS.

---

# 31. Important Interview Concepts

## Why Use Front Door + App Gateway Together?

### Front Door

Provides:

* Global routing
* Multi-region failover
* CDN
* Global WAF
* Edge acceleration

### App Gateway

Provides:

* Regional ingress
* AKS integration
* Backend balancing
* Path-based routing
* AGIC integration

Together:

```text
Front Door = Global Layer
App Gateway = Regional Layer
```

---

# 32. Important Clarification About Azure AD Domain Services

Azure AD Domain Services is different.

AAD DS is used for:

* LDAP
* Kerberos
* NTLM
* Windows authentication
* Domain join

AAD DS is NOT used for:

* Public website domains
* Azure Front Door DNS
* Internet DNS hosting

---

# 33. Terraform Azure DNS Zone Example

```hcl
resource "azurerm_dns_zone" "main" {
  name                = "contoso.com"
  resource_group_name = azurerm_resource_group.main.name
}
```

---

# 34. Terraform Azure Front Door Backend Configuration with Application Gateway

This section shows exactly where Azure Front Door is configured to send traffic to Application Gateway.

---

# 34.1 Front Door Profile

```hcl
resource "azurerm_cdn_frontdoor_profile" "fd" {
  name                = "prod-frontdoor"
  resource_group_name = azurerm_resource_group.main.name
  sku_name            = "Premium_AzureFrontDoor"
}
```

---

# 34.2 Front Door Endpoint

```hcl
resource "azurerm_cdn_frontdoor_endpoint" "fd_endpoint" {
  name                     = "prod-endpoint"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.fd.id
}
```

---

# 34.3 Front Door Origin Group

Origin Group contains backend origins.

```hcl
resource "azurerm_cdn_frontdoor_origin_group" "appgw_group" {
  name                     = "appgw-origin-group"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.fd.id

  health_probe {
    interval_in_seconds = 30
    path                = "/health"
    protocol            = "Https"
    request_type        = "GET"
  }

  load_balancing {
    sample_size                        = 4
    successful_samples_required        = 3
  }
}
```

---

# 34.4 Front Door Origin (Application Gateway Backend)

THIS is the exact place where Front Door is connected to Application Gateway.

```hcl
resource "azurerm_cdn_frontdoor_origin" "appgw_origin" {
  name                           = "eastus-appgw"
  cdn_frontdoor_origin_group_id  = azurerm_cdn_frontdoor_origin_group.appgw_group.id

  host_name          = azurerm_public_ip.appgw_pip.fqdn
  origin_host_header = azurerm_public_ip.appgw_pip.fqdn

  http_port  = 80
  https_port = 443
  priority   = 1
  weight     = 1000
  enabled    = true

  certificate_name_check_enabled = true
}
```

---

# Important Understanding

In this block:

```hcl
host_name = azurerm_public_ip.appgw_pip.fqdn
```

Front Door backend target is:

```text
Application Gateway Public IP/FQDN
```

This is how Front Door forwards traffic to App Gateway.

---

# 34.5 Front Door Route

```hcl
resource "azurerm_cdn_frontdoor_route" "route" {
  name                          = "app-route"
  cdn_frontdoor_endpoint_id     = azurerm_cdn_frontdoor_endpoint.fd_endpoint.id
  cdn_frontdoor_origin_group_id = azurerm_cdn_frontdoor_origin_group.appgw_group.id

  supported_protocols = ["Http", "Https"]
  patterns_to_match   = ["/*"]
  forwarding_protocol = "HttpsOnly"

  https_redirect_enabled = true
  link_to_default_domain = true
}
```

---

# 34.6 Full Traffic Flow

```text
User
  |
Azure Front Door
  |
Origin Group
  |
Application Gateway Public FQDN/IP
  |
AKS Ingress
  |
Service
  |
Pods
```

---

# 34.7 Multi-Region Backend Example

```hcl
resource "azurerm_cdn_frontdoor_origin" "westus_appgw" {
  name                          = "westus-appgw"
  cdn_frontdoor_origin_group_id = azurerm_cdn_frontdoor_origin_group.appgw_group.id

  host_name = azurerm_public_ip.westus_appgw_pip.fqdn

  priority = 2
  weight   = 500
}
```

---

# How Failover Works

* Front Door continuously sends health probes
* If East US App Gateway fails
* Front Door routes traffic to West US App Gateway

---

# 34.8 Health Probe Flow

```text
Front Door
    |
HTTPS Probe
    |
Application Gateway
    |
AKS Ingress
    |
/health Endpoint
```

---

# 34.9 Production Pattern

```text
Global Users
      |
Azure Front Door
      |
Regional Application Gateways
      |
AKS Clusters
      |
Pods
```

---

# 34. Terraform CNAME Example

```hcl
resource "azurerm_dns_cname_record" "fd" {
  name                = "app"
  zone_name           = azurerm_dns_zone.main.name
  resource_group_name = azurerm_resource_group.main.name
  ttl                 = 300
  record              = "myapp.azurefd.net"
}
```

---

# 35. Validation Commands

## Check DNS

```bash
nslookup app.contoso.com
```

---

## Check CNAME

```bash
dig app.contoso.com
```

---

## Check Name Servers

```bash
nslookup -type=ns contoso.com
```

---

## Validate HTTPS

```bash
curl https://app.contoso.com
```

---

# 36. Common Mistakes

| Mistake                       | Problem                     |
| ----------------------------- | --------------------------- |
| Wrong CNAME                   | Front Door unreachable      |
| Missing TXT validation        | SSL/domain validation fails |
| Wrong NS delegation           | DNS failure                 |
| Using A record for Front Door | Hard IP management          |
| Missing health probes         | Traffic failures            |

---

# 37. Best Practices

## Recommended

* Use Azure App Service Domain
* Use Azure DNS Zone
* Use CNAME for Front Door
* Use managed SSL certificates
* Use Front Door WAF
* Use AGIC with App Gateway
* Use health probes
* Use multi-region backend architecture

---

# 38. Final Interview Summary Answer

"In production we purchased and managed domains using Azure App Service Domains and hosted DNS records in Azure DNS Zones. Azure DNS became authoritative using Azure-generated name servers. We configured CNAME records pointing application domains like app.contoso.com to Azure Front Door endpoints such as myapp.azurefd.net. Azure Front Door handled global routing, WAF, SSL, CDN, and multi-region failover, while Application Gateway provided regional ingress routing into AKS using AGIC. Client DNS resolution flows through root servers, TLD servers, and Azure DNS name servers before resolving Azure Front Door endpoints and routing traffic to healthy backend regions."

---

# Conclusion

This architecture provides:

* Enterprise-grade DNS hosting
* Global application delivery
* Secure custom domains
* Multi-region routing
* AKS ingress integration
* SSL/TLS automation
* Production-grade scalability

These concepts are heavily used in enterprise Azure environments and are commonly discussed in DevOps, SRE, platform engineering, and cloud architect interviews.
