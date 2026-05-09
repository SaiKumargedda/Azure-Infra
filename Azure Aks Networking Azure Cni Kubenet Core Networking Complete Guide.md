# Complete Azure AKS Networking, Azure CNI, Kubenet, CoreDNS, kube-proxy, NSG, UDR, Firewall and Kubernetes Networking Guide

# 1. Introduction

This document explains complete AKS and Azure networking architecture including:

* Azure CNI
* Kubenet
* Azure CNI Overlay
* AKS networking internals
* CoreDNS
* kube-proxy
* iptables
* NSGs
* UDRs
* Azure Firewall
* VNet/Subnet communication
* Pod-to-pod communication
* Service networking
* Network policies
* Headless services
* Same namespace communication
* Cross namespace communication
* Cross cluster networking
* Private networking
* Azure resource security
* Enterprise AKS networking patterns

This guide is useful for:

* AKS interviews
* DevOps interviews
* Platform engineering
* Kubernetes networking preparation
* Azure networking preparation

---

# 2. What is Azure CNI?

Azure CNI is Azure's native Kubernetes networking model.

With Azure CNI:

* Pods receive IP addresses directly from Azure VNet subnet
* Pods become first-class citizens inside Azure network
* Pods can directly communicate with:

  * VMs
  * Databases
  * Private endpoints
  * Other VNets

without NAT.

---

# 3. Azure CNI Traffic Flow

```text
Pod
 |
Azure VNet IP
 |
Azure SDN
 |
Other Azure Resources
```

---

# 4. Azure CNI Characteristics

| Feature             | Azure CNI |
| ------------------- | --------- |
| Pod gets VNet IP    | Yes       |
| Direct pod routing  | Yes       |
| Works with NSGs     | Yes       |
| Works with UDR      | Yes       |
| Enterprise friendly | Yes       |
| IP consumption      | High      |

---

# 5. What is Kubenet?

Kubenet is basic Kubernetes networking.

Pods get:

* Internal pod CIDR IPs
* NOT Azure VNet IPs

Traffic gets NATed through node IP.

---

# 6. Kubenet Flow

```text
Pod
 |
Node NAT
 |
Node IP
 |
Azure Network
```

---

# 7. Kubenet Characteristics

| Feature                   | Kubenet |
| ------------------------- | ------- |
| Pod gets VNet IP          | No      |
| Pod gets internal CIDR IP | Yes     |
| Uses NAT                  | Yes     |
| Simpler                   | Yes     |
| IP efficient              | Yes     |
| Enterprise flexibility    | Lower   |

---

# 8. Azure CNI vs Kubenet

| Feature          | Azure CNI         | Kubenet               |
| ---------------- | ----------------- | --------------------- |
| Pod IP Source    | Azure VNet        | Internal Pod CIDR     |
| NAT Required     | No                | Yes                   |
| Pod Reachability | Direct            | Through Node          |
| NSG Support      | Better            | Limited               |
| Scalability      | Good with Overlay | Better IP efficiency  |
| Enterprise Use   | Preferred         | Small/simple clusters |

---

# 9. Why Azure CNI is Preferred in Enterprises

Because:

* Better integration with Azure networking
* Easier security control
* Easier firewall integration
* Better observability
* Better private networking support
* Easier routing

---

# 10. Azure CNI Overlay

## Modern AKS Recommendation

Azure CNI Overlay solves IP exhaustion problems.

Pods:

* Use overlay pod CIDR
* Still integrate with Azure networking

Benefits:

* Better scalability
* Reduced subnet IP usage
* Enterprise networking support

---

# 11. Azure CNI Overlay Flow

```text
Pod CIDR Overlay
     |
Node Routing
     |
Azure SDN
     |
Azure Network
```

---

# 12. Where Azure CNI is Enabled

Enabled during AKS creation.

Terraform:

```hcl
network_profile {
  network_plugin = "azure"
}
```

For Kubenet:

```hcl
network_profile {
  network_plugin = "kubenet"
}
```

---

# 13. Which Resources Use Azure CNI?

Azure CNI impacts:

* AKS pods
* AKS services
* Pod routing
* NSGs
* Firewall routing
* Private endpoint communication
* VNet communication

Not used directly by:

* Azure SQL
* Storage
* Key Vault

But pods communicate to them through Azure networking.

---

# 14. How Azure Networking Works Internally

Azure networking uses:

* Azure SDN (Software Defined Networking)
* Virtual Networks
* Subnets
* Route tables
* NSGs
* Azure backbone routing

Azure automatically manages:

* Routing
* Encapsulation
* Backend SDN flows

---

# 15. Core Azure Networking Components

| Component        | Purpose            |
| ---------------- | ------------------ |
| VNet             | Network boundary   |
| Subnet           | Segmentation       |
| NSG              | Packet filtering   |
| UDR              | Custom routing     |
| Firewall         | Traffic inspection |
| Private Endpoint | Private access     |
| DNS              | Name resolution    |

---

# 16. Same Subnet Communication

## Example

```text
VM1 --> VM2
```

Inside same subnet.

Azure automatically routes locally.

No UDR needed.

---

# 17. Same VNet Different Subnet Communication

## Example

```text
AKS Subnet --> DB Subnet
```

Azure system routes handle routing automatically.

Flow:

```text
Source NIC
   |
VNet Router
   |
Destination Subnet
```

---

# 18. Different VNet Communication

Requires:

* VNet peering
* VPN
* ExpressRoute

---

# 19. Different VNet Flow

```text
AKS VNet
   |
VNet Peering
   |
Shared Services VNet
```

Traffic remains private.

---

# 20. How NSGs Work

NSG = Network Security Group.

Acts like packet filtering firewall.

Can be attached to:

* Subnets
* NICs

---

# 21. NSG Processing Flow

```text
Packet
  |
NSG Rules Evaluated
  |
Allow or Deny
```

---

# 22. Example NSG Rule

Allow:

```text
AKS Subnet --> SQL Port 1433
```

Deny:

```text
Internet --> DB Subnet
```

---

# 23. How UDR Works

UDR = User Defined Route.

Overrides Azure default routing.

---

# 24. Example UDR Flow

```text
AKS
 |
Route Table
 |
Azure Firewall
 |
Internet
```

---

# 25. Why Use UDR?

To:

* Force traffic through firewall
* Centralize security
* Inspect outbound traffic
* Control routing

---

# 26. Azure Firewall Flow

```text
Pod
 |
Node
 |
UDR
 |
Azure Firewall
 |
Destination
```

---

# 27. kube-proxy Explained

kube-proxy manages Kubernetes service networking.

Responsible for:

* Service virtual IPs
* Load balancing
* iptables rules
* Traffic forwarding

Runs on every node.

---

# 28. kube-proxy Flow

```text
Service IP
   |
kube-proxy
   |
iptables Rules
   |
Backend Pods
```

---

# 29. What is iptables?

iptables is Linux packet filtering/routing system.

Used for:

* NAT
* Port forwarding
* Traffic routing
* Kubernetes service networking

---

# 30. Kubernetes Service Networking

## ClusterIP Example

```text
Application
    |
Service IP
    |
kube-proxy
    |
Backend Pod
```

---

# 31. How Service Discovery Works

Pods communicate using:

```text
service-name.namespace.svc.cluster.local
```

Resolved through CoreDNS.

---

# 32. What is CoreDNS?

CoreDNS provides internal Kubernetes DNS.

Responsible for:

* Service discovery
* Pod DNS resolution
* Cluster DNS

---

# 33. CoreDNS Flow

```text
Application Pod
      |
DNS Query
      |
CoreDNS
      |
Kubernetes API
      |
Service IP Returned
```

---

# 34. Example Service Resolution

Request:

```text
backend.default.svc.cluster.local
```

CoreDNS returns:

```text
10.0.0.15
```

---

# 35. Same Namespace Communication

Pods can directly communicate using:

```text
service-name
```

because same namespace automatically resolves.

---

# 36. Different Namespace Communication

Need full DNS.

Example:

```text
backend.prod.svc.cluster.local
```

---

# 37. Same Cluster Pod Communication

Pods communicate:

* Using pod IPs
* Using services
* Through CNI networking

---

# 38. Different AKS Cluster Communication

Requires:

* VNet peering
* Service mesh
* Gateway
* Multi-cluster ingress

---

# 39. What is Headless Service?

Normal service:

```text
Service IP --> Pods
```

Headless service:

No ClusterIP.

DNS directly returns pod IPs.

---

# 40. Headless Service Example

```yaml
clusterIP: None
```

---

# 41. Headless Service Flow

```text
Application
   |
CoreDNS
   |
Direct Pod IPs Returned
```

---

# 42. Why Use Headless Services?

Common for:

* StatefulSets
* Databases
* Kafka
* Cassandra
* Elasticsearch

---

# 43. Kubernetes Network Policies

Network Policies control pod communication.

Like firewall for pods.

---

# 44. Example Policy

Allow only frontend namespace to access backend.

---

# 45. Network Policy Flow

```text
Packet
 |
CNI Plugin
 |
Policy Check
 |
Allow or Deny
```

---

# 46. Example Network Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      app: backend

  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
```

---

# 47. AKS Private Networking

Private networking integrates with:

* Azure CNI
* NSGs
* UDRs
* Firewalls
* Private Endpoints

---

# 48. AKS to Private Endpoint Flow

```text
Pod
 |
CoreDNS
 |
Private DNS
 |
Private Endpoint IP
 |
Azure Private Link
 |
Azure SQL/Storage/KV
```

---

# 49. How Applications Reach Each Other

## Same Namespace

```text
frontend --> backend
```

Simple service name.

---

## Different Namespace

```text
frontend --> backend.prod.svc.cluster.local
```

---

## Different VNet

```text
AKS VNet --> VNet Peering --> Shared Services VNet
```

---

# 50. Packet Flow Inside AKS

```text
Application Pod
      |
Pod Network Namespace
      |
veth Pair
      |
Linux Bridge
      |
CNI Plugin
      |
Node Interface
      |
Azure SDN
```

---

# 51. How kube-proxy Uses iptables

kube-proxy dynamically creates:

* NAT rules
* Service forwarding rules
* Load balancing rules

Example:

```text
Service IP --> Backend Pod IP
```

---

# 52. Pod-to-Service Communication

```text
Pod
 |
Service VIP
 |
kube-proxy
 |
iptables
 |
Backend Pod
```

---

# 53. NodePort Flow

```text
External Request
      |
Node IP:Port
      |
kube-proxy
      |
Backend Pod
```

---

# 54. LoadBalancer Service Flow

```text
Internet
   |
Azure Load Balancer
   |
Node
   |
kube-proxy
   |
Pod
```

---

# 55. Ingress Flow

```text
User
 |
Ingress Controller
 |
Service
 |
Pod
```

---

# 56. Azure Application Gateway + AGIC Flow

```text
User
 |
Application Gateway
 |
Ingress
 |
Service
 |
Pods
```

---

# 57. How Azure Firewall and NSG Work Together

## NSG

Controls subnet/NIC traffic.

---

## Firewall

Provides:

* Deep inspection
* Threat intelligence
* URL filtering
* Central security

---

# 58. Enterprise Security Pattern

```text
Pods
 |
NSG
 |
UDR
 |
Azure Firewall
 |
Internet
```

---

# 59. How Azure Resources are Secured

## Common Controls

| Security Control | Purpose            |
| ---------------- | ------------------ |
| NSGs             | Packet filtering   |
| Firewall         | Traffic inspection |
| Private Endpoint | No public exposure |
| RBAC             | Identity security  |
| Managed Identity | Secretless auth    |
| Network Policies | Pod isolation      |
| WAF              | HTTP protection    |
| Defender         | Threat detection   |

---

# 60. Real Enterprise AKS Networking Architecture

```text
Internet Users
      |
Azure Front Door
      |
Application Gateway WAF
      |
AKS Cluster
      |
Pods
      |
Private Endpoint
      |
SQL / Storage / Key Vault
```

---

# 61. Common Interview Questions

## Q1: Difference between Azure CNI and Kubenet?

Answer:

Azure CNI assigns Azure VNet IP addresses directly to pods, allowing direct communication with Azure resources. Kubenet uses internal pod CIDRs and performs NAT through node IPs.

---

## Q2: Why is Azure CNI preferred?

Answer:

Better Azure integration, direct pod routing, easier security management, NSG support, firewall integration, and enterprise networking support.

---

## Q3: What does kube-proxy do?

Answer:

kube-proxy manages Kubernetes service networking using iptables rules to forward traffic from service IPs to backend pods.

---

## Q4: What is Headless Service?

Answer:

Headless service does not allocate ClusterIP. DNS directly returns pod IP addresses.

---

## Q5: How does CoreDNS work?

Answer:

CoreDNS provides internal Kubernetes DNS resolution for services and pods by querying Kubernetes API objects.

---

## Q6: Difference between NSG and Network Policy?

| NSG                  | Network Policy              |
| -------------------- | --------------------------- |
| Azure network level  | Kubernetes pod level        |
| Subnet/NIC filtering | Pod communication filtering |
| Azure managed        | Kubernetes managed          |

---

# 62. Common Production Best Practices

## Recommended

* Use Azure CNI Overlay
* Use Network Policies
* Use Private Endpoints
* Use NSGs
* Use Azure Firewall
* Use Private AKS
* Use AGIC
* Use centralized logging
* Use HPA
* Use topology spread constraints

---

# 63. Common Mistakes

| Mistake                | Problem                |
| ---------------------- | ---------------------- |
| No Network Policies    | Open pod communication |
| Poor subnet sizing     | IP exhaustion          |
| No Firewall            | Uncontrolled outbound  |
| Public DB access       | Security risk          |
| No NSGs                | Lateral movement       |
| Using Kubenet at scale | Routing limitations    |

---

# 64. End-to-End AKS Networking Flow

```text
User
 |
Front Door
 |
Application Gateway
 |
Ingress
 |
Service
 |
kube-proxy
 |
iptables
 |
Pod
 |
Private Endpoint
 |
Azure SQL
```

---

# 65. Final Interview Summary Answer

"In production we used Azure CNI Overlay for AKS networking because it provides enterprise-grade Azure integration while solving IP exhaustion problems. Pods communicate through Azure SDN and integrate with NSGs, UDRs, Azure Firewall, and Private Endpoints. kube-proxy manages Kubernetes service routing using iptables rules, while CoreDNS handles internal DNS resolution for services and pods. Network Policies secure pod-to-pod communication, and Azure Firewall with UDRs centralizes outbound traffic inspection. Private Endpoints and Private DNS Zones ensure secure private connectivity between AKS workloads and Azure services like SQL Database, Storage Accounts, Key Vault, and ACR."

---

# Conclusion

This architecture provides:

* Enterprise-grade AKS networking
* Secure pod communication
* Advanced traffic control
* Private networking integration
* Scalable Azure networking
* Production-grade security
* Strong Kubernetes networking understanding

These concepts are heavily used in real enterprise Azure AKS environments and are very important for DevOps, SRE, Platform Engineering, and Cloud Architect interviews.
