# End-to-End Highly Available Azure AKS Architecture Across Availability Zones and Regions

## Overview

This document explains how to design and deploy a highly available, production-grade Azure Kubernetes Service (AKS) platform across:

* Multiple Availability Zones (AZ)
* Multiple Azure Regions
* Azure Front Door for global routing
* Application Gateway + AGIC for ingress
* Zone-aware Kubernetes scheduling
* Pod Topology Spread Constraints
* Health probes and failover
* Terraform-based Infrastructure as Code
* High availability interview concepts

---

# 1. High-Level Architecture

## Architecture Components

### Region 1 (Primary)

* AKS Cluster
* Multiple node pools spread across AZ1, AZ2, AZ3
* Azure Application Gateway
* AGIC (Application Gateway Ingress Controller)
* Azure Container Registry
* Azure Key Vault
* Azure Monitor + Log Analytics
* NAT Gateway

### Region 2 (Secondary / DR)

* Same architecture replicated
* Independent AKS cluster
* Independent App Gateway
* Same application deployed

### Global Entry

* Azure Front Door
* Health probes
* Priority/Latency routing
* Automatic failover
* WAF policies

---

# 2. Availability Zones vs Regions

| Feature          | Availability Zone        | Region                   |
| ---------------- | ------------------------ | ------------------------ |
| Scope            | Within same Azure region | Separate Azure geography |
| Protects Against | Datacenter failure       | Entire regional outage   |
| Latency          | Very low                 | Higher                   |
| Example          | East US Zone 1/2/3       | East US vs Central US    |
| Used For         | HA                       | DR + Global HA           |

---

# 3. Production Traffic Flow

1. User accesses application URL
2. DNS resolves to Azure Front Door
3. Azure Front Door performs health checks
4. Front Door selects healthy backend region
5. Traffic forwarded to regional Application Gateway
6. AGIC updates App Gateway backend pool dynamically
7. App Gateway routes traffic to AKS ingress/service
8. Kubernetes service routes to healthy pods

---

# 4. Terraform Folder Structure

```text
terraform/
├── modules/
│   ├── network/
│   ├── aks/
│   ├── appgw/
│   ├── frontdoor/
│   ├── monitoring/
│   └── keyvault/
│
├── env/
│   ├── prod/
│   │   ├── main.tf
│   │   ├── provider.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── outputs.tf
│
└── backend/
```

---

# 5. Provider Configuration

## provider.tf

```hcl
terraform {
  required_version = ">=1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.100"
    }
  }

  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "tfstateprod001"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}

provider "azurerm" {
  features {}
}
```

---

# 6. Resource Group

```hcl
resource "azurerm_resource_group" "main" {
  name     = "prod-aks-rg"
  location = "East US"
}
```

---

# 7. Virtual Network

## Multi-subnet Design

```hcl
resource "azurerm_virtual_network" "main" {
  name                = "prod-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_subnet" "aks" {
  name                 = "aks-subnet"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_subnet" "appgw" {
  name                 = "appgw-subnet"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.2.0/24"]
}
```

---

# 8. Highly Available AKS Cluster

## Key HA Features

* Multiple availability zones
* System and user node pools
* Autoscaling
* Managed identity
* Azure CNI
* Private cluster optional
* Uptime SLA
* Availability zone aware scheduling

## AKS Terraform

```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "prod-aks"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = "prodaks"
  kubernetes_version  = "1.30"

  sku_tier = "Paid"

  default_node_pool {
    name                 = "systempool"
    vm_size              = "Standard_D4s_v5"
    node_count           = 3
    auto_scaling_enabled = true
    min_count            = 3
    max_count            = 10

    vnet_subnet_id = azurerm_subnet.aks.id

    zones = ["1", "2", "3"]

    type = "VirtualMachineScaleSets"
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin = "azure"
    network_policy = "azure"
    load_balancer_sku = "standard"
  }

  oms_agent {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.law.id
  }
}
```

---

# 9. Additional User Node Pool

```hcl
resource "azurerm_kubernetes_cluster_node_pool" "userpool" {
  name                  = "userpool"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.aks.id
  vm_size               = "Standard_D4s_v5"
  mode                  = "User"

  enable_auto_scaling = true
  min_count           = 2
  max_count           = 20

  node_taints = [
    "workload=apps:NoSchedule"
  ]

  zones = ["1", "2", "3"]

  vnet_subnet_id = azurerm_subnet.aks.id
}
```

---

# 10. Application Gateway

## Why App Gateway?

* Layer 7 load balancing
* SSL termination
* WAF support
* URL/path routing
* Cookie affinity
* Integration with AGIC

## Terraform

```hcl
resource "azurerm_public_ip" "appgw" {
  name                = "appgw-pip"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

resource "azurerm_application_gateway" "appgw" {
  name                = "prod-appgw"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  sku {
    name     = "WAF_v2"
    tier     = "WAF_v2"
    capacity = 2
  }

  gateway_ip_configuration {
    name      = "gateway-ip-config"
    subnet_id = azurerm_subnet.appgw.id
  }

  frontend_ip_configuration {
    name                 = "frontend-ip"
    public_ip_address_id = azurerm_public_ip.appgw.id
  }

  frontend_port {
    name = "https-port"
    port = 443
  }

  backend_address_pool {
    name = "backend-pool"
  }

  backend_http_settings {
    name                  = "backend-setting"
    cookie_based_affinity = "Disabled"
    path                  = "/"
    port                  = 80
    protocol              = "Http"
    request_timeout       = 60
  }

  http_listener {
    name                           = "https-listener"
    frontend_ip_configuration_name = "frontend-ip"
    frontend_port_name             = "https-port"
    protocol                       = "Https"
    ssl_certificate_name           = "ssl-cert"
  }
}
```

---

# 11. Enable AGIC (Application Gateway Ingress Controller)

```hcl
ingress_application_gateway {
  gateway_id = azurerm_application_gateway.appgw.id
}
```

## What AGIC Does

AGIC watches Kubernetes ingress resources and dynamically configures:

* Backend pools
* Health probes
* Listeners
* Routing rules
* SSL settings

---

# 12. Azure Front Door Configuration

## Why Azure Front Door?

* Global traffic distribution
* Multi-region failover
* Anycast network
* Health probes
* WAF
* SSL offloading
* CDN acceleration

---

# 13. Front Door Health Check Flow

Azure Front Door continuously sends health probes to backend endpoints.

Example:

```text
/health
/ready
/status
```

If Region 1 becomes unhealthy:

* Front Door marks backend unhealthy
* Stops routing traffic
* Routes users automatically to Region 2

---

# 14. Azure Front Door Terraform

```hcl
resource "azurerm_cdn_frontdoor_profile" "fd" {
  name                = "global-frontdoor"
  resource_group_name = azurerm_resource_group.main.name
  sku_name            = "Standard_AzureFrontDoor"
}

resource "azurerm_cdn_frontdoor_endpoint" "endpoint" {
  name                     = "prod-endpoint"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.fd.id
}

resource "azurerm_cdn_frontdoor_origin_group" "og" {
  name                     = "aks-origin-group"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.fd.id

  health_probe {
    interval_in_seconds = 30
    path                = "/health"
    protocol            = "Https"
    request_type        = "GET"
  }

  load_balancing {
    sample_size                 = 4
    successful_samples_required = 3
  }
}
```

---

# 15. Multiple Region Backends

```hcl
resource "azurerm_cdn_frontdoor_origin" "region1" {
  name                          = "eastus-origin"
  cdn_frontdoor_origin_group_id = azurerm_cdn_frontdoor_origin_group.og.id

  enabled                      = true
  host_name                    = "eastus-appgw.contoso.com"
  http_port                    = 80
  https_port                   = 443
  origin_host_header           = "eastus-appgw.contoso.com"
  priority                     = 1
  weight                       = 1000
  certificate_name_check_enabled = true
}

resource "azurerm_cdn_frontdoor_origin" "region2" {
  name                          = "centralus-origin"
  cdn_frontdoor_origin_group_id = azurerm_cdn_frontdoor_origin_group.og.id

  enabled                      = true
  host_name                    = "centralus-appgw.contoso.com"
  http_port                    = 80
  https_port                   = 443
  origin_host_header           = "centralus-appgw.contoso.com"
  priority                     = 2
  weight                       = 500
  certificate_name_check_enabled = true
}
```

---

# 16. How Front Door Sends Traffic

## Priority-Based Routing

| Region     | Priority |
| ---------- | -------- |
| East US    | 1        |
| Central US | 2        |

Traffic always goes to East US unless unhealthy.

---

## Latency-Based Routing

Front Door sends users to nearest healthy region.

Example:

* India users → Southeast Asia
* Europe users → West Europe
* US users → East US

---

# 17. Kubernetes High Availability Scheduling

## Problem

Without topology constraints:

* Pods may schedule into same node
* Same AZ failure may kill all pods

---

# 18. Pod Topology Spread Constraints

## Goal

Distribute pods evenly across:

* Availability zones
* Nodes
* Regions

---

# 19. Kubernetes Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 6

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: nginx

      containers:
      - name: nginx
        image: nginx
```

---

# 20. How maxSkew Works

## Example

6 replicas across 3 AZs:

| Zone   | Pods |
| ------ | ---- |
| Zone 1 | 2    |
| Zone 2 | 2    |
| Zone 3 | 2    |

If scheduler tries:

| Zone   | Pods |
| ------ | ---- |
| Zone 1 | 4    |
| Zone 2 | 1    |
| Zone 3 | 1    |

Kubernetes rejects because skew > 1.

---

# 21. Node Affinity

## Example

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: agentpool
          operator: In
          values:
          - userpool
```

---

# 22. Pod Anti-Affinity

## Prevent Same Node Placement

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchExpressions:
      - key: app
        operator: In
        values:
        - nginx
    topologyKey: kubernetes.io/hostname
```

---

# 23. Multi-Region Kubernetes Strategy

Kubernetes itself does NOT schedule pods across regions automatically.

Common pattern:

| Region     | Cluster       |
| ---------- | ------------- |
| East US    | AKS Cluster 1 |
| Central US | AKS Cluster 2 |

Applications deployed independently.

Azure Front Door performs global traffic distribution.

---

# 24. Recommended Production Setup

## Region 1

* AKS Cluster
* 3 AZs
* App Gateway
* AGIC
* Front Door backend

## Region 2

* Same setup
* Warm standby or active-active

---

# 25. Active-Active vs Active-Passive

| Mode           | Description                 |
| -------------- | --------------------------- |
| Active-Active  | Both regions serve traffic  |
| Active-Passive | DR only used during failure |

---

# 26. Stateful Applications

## Database Considerations

Use:

* Azure SQL Geo-replication
* Cosmos DB multi-region
* Azure Redis geo replication
* Storage account GRS

---

# 27. AKS Upgrade Strategy

## Best Practice

* Use surge upgrades
* Upgrade one node pool at a time
* Use PodDisruptionBudgets

Example:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: nginx
```

---

# 28. HPA Configuration

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-app

  minReplicas: 3
  maxReplicas: 20

  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

# 29. Cluster Autoscaler

## Terraform

```hcl
auto_scaling_enabled = true
min_count = 3
max_count = 20
```

Cluster Autoscaler adds/removes nodes automatically.

---

# 30. Monitoring

## Components

* Azure Monitor
* Container Insights
* Log Analytics
* Application Insights
* Prometheus + Grafana

---

# 31. Azure Monitor Terraform

```hcl
resource "azurerm_log_analytics_workspace" "law" {
  name                = "prod-law"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "PerGB2018"
}
```

---

# 32. Key Vault

## Why?

* TLS certificates
* Database passwords
* Secrets
* API keys

---

# 33. NAT Gateway

## Purpose

Stable outbound internet connectivity.

Benefits:

* Static outbound IP
* SNAT scaling
* Better security

---

# 34. Network Security Best Practices

## Recommendations

* Private AKS cluster
* NSGs
* Azure Firewall
* WAF enabled
* Network policies
* Private endpoints
* RBAC
* Managed identities

---

# 35. Ingress Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
spec:
  rules:
  - host: app.contoso.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

---

# 36. Service Manifest

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx

  ports:
  - port: 80
    targetPort: 80

  type: ClusterIP
```

---

# 37. Readiness and Liveness Probes

## Why Important?

Azure Front Door and App Gateway rely on healthy applications.

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 80

readinessProbe:
  httpGet:
    path: /ready
    port: 80
```

---

# 38. Interview Questions and Answers

## Q1: How do you achieve HA in AKS?

Answer:

* Deploy node pools across multiple AZs
* Use multiple replicas
* Use topology spread constraints
* Configure HPA + cluster autoscaler
* Use App Gateway + AGIC
* Use Azure Front Door across regions
* Use health probes and PodDisruptionBudgets

---

## Q2: How does Azure Front Door failover work?

Answer:

Front Door continuously probes backend endpoints.
If backend fails health checks:

* Backend marked unhealthy
* Traffic stopped
* Requests routed to healthy region

---

## Q3: Can Kubernetes schedule pods across regions?

Answer:

No.
Kubernetes scheduler works within a single cluster.
Multi-region deployments require separate AKS clusters.
Global traffic distribution is handled by Azure Front Door.

---

## Q4: Difference between Pod Anti-Affinity and Topology Spread?

| Feature    | Anti-Affinity   | Topology Spread   |
| ---------- | --------------- | ----------------- |
| Purpose    | Avoid same node | Even distribution |
| Strictness | Strong          | Balanced          |
| Common Use | HA              | AZ balancing      |

---

## Q5: What happens if one AZ fails?

Answer:

* AKS reschedules workloads to healthy AZs
* App Gateway continues routing
* Front Door unaffected
* Autoscaler may create new nodes

---

# 39. Production Best Practices

## Recommended Architecture

* 3-zone AKS cluster
* Separate system/user node pools
* Azure CNI Overlay
* AGIC
* Azure Front Door Premium
* WAF enabled
* Private cluster
* Key Vault CSI driver
* Monitoring enabled
* GitOps with ArgoCD/Flux

---

# 40. End-to-End Deployment Steps

## Step 1

Create networking.

## Step 2

Create AKS cluster with AZs.

## Step 3

Create App Gateway.

## Step 4

Enable AGIC.

## Step 5

Deploy application.

## Step 6

Create ingress.

## Step 7

Configure Azure Front Door.

## Step 8

Add health probes.

## Step 9

Configure autoscaling.

## Step 10

Validate failover.

---

# 41. Validation Commands

## Check Node Zones

```bash
kubectl get nodes -L topology.kubernetes.io/zone
```

## Check Pod Distribution

```bash
kubectl get pods -o wide
```

## Describe Pod

```bash
kubectl describe pod <pod-name>
```

## Check Ingress

```bash
kubectl get ingress
```

---

# 42. Simulating Failures

## AZ Failure Test

Drain nodes:

```bash
kubectl drain <node-name> --ignore-daemonsets
```

## Region Failure Test

Disable backend in Front Door.

Observe traffic failover.

---

# 43. Important Azure Services Used

| Service       | Purpose            |
| ------------- | ------------------ |
| AKS           | Kubernetes cluster |
| App Gateway   | Layer 7 ingress    |
| AGIC          | Ingress automation |
| Front Door    | Global routing     |
| ACR           | Container registry |
| Key Vault     | Secrets            |
| Azure Monitor | Monitoring         |
| Log Analytics | Logs               |
| Azure DNS     | DNS                |
| NAT Gateway   | Outbound traffic   |

---

# 44. Real Interview Summary Answer

"We designed a highly available AKS architecture using multiple availability zones within each region and multiple Azure regions for disaster recovery. AKS node pools were distributed across Zone 1, Zone 2, and Zone 3 using VMSS. Applications used topology spread constraints and pod anti-affinity to distribute pods evenly across zones. Azure Application Gateway with AGIC handled ingress traffic, while Azure Front Door globally routed users to the nearest healthy region using health probes and priority or latency-based routing. HPA and Cluster Autoscaler ensured scalability, and Azure Monitor + Log Analytics provided observability."

---

# 45. Final Production Recommendations

## Must Have

* 3 AZ node pools
* Minimum 3 replicas
* HPA
* Cluster Autoscaler
* PodDisruptionBudget
* Readiness probes
* Topology spread constraints
* Azure Front Door
* WAF
* Monitoring
* Backup strategy

---

# 46. Common Mistakes

| Mistake             | Problem                  |
| ------------------- | ------------------------ |
| Single node pool    | SPOF                     |
| No readiness probes | Bad failover             |
| No topology spread  | Uneven pod placement     |
| No autoscaling      | Capacity issues          |
| Single region       | No DR                    |
| No PDB              | Downtime during upgrades |

---

# 47. Key Kubernetes Labels Used

| Label                       | Purpose             |
| --------------------------- | ------------------- |
| topology.kubernetes.io/zone | AZ awareness        |
| kubernetes.io/hostname      | Node placement      |
| agentpool                   | Node pool targeting |

---

# 48. Final Architecture Summary

```text
Users
  |
Azure Front Door
  |
--------------------------------
|                              |
Region 1                    Region 2
|                              |
Application Gateway       Application Gateway
|                              |
AGIC                       AGIC
|                              |
AKS Cluster                AKS Cluster
|                              |
AZ1 AZ2 AZ3               AZ1 AZ2 AZ3
```

---

# 49. Important kubectl Commands

```bash
kubectl get nodes
kubectl get pods -A
kubectl describe pod
kubectl top nodes
kubectl top pods
kubectl get ingress
kubectl get svc
kubectl get deployment
kubectl logs
kubectl exec -it
```

---

# 50. Important Terraform Commands

```bash
terraform init
terraform validate
terraform fmt
terraform plan
terraform apply
terraform destroy
terraform state list
terraform state show
terraform import
```

---

# Conclusion

This architecture provides:

* Zone-level high availability
* Region-level disaster recovery
* Automatic failover
* Autoscaling
* Secure ingress
* Global traffic management
* Production-grade reliability
* Strong interview-ready concepts

This is one of the most common enterprise AKS production deployment patterns used in real-world organizations.


# Additional Important Concept: System Node Pools vs User Node Pools Scheduling

This section should be added to the AKS Upgrade and High Availability guide.

---

# System Node Pools vs User Node Pools

AKS supports two types of node pools:

| Node Pool Type | Purpose |
|---|---|
| System Node Pool | Critical Kubernetes system workloads |
| User Node Pool | Application workloads |

---

# 1. System Node Pool

System node pools host critical Kubernetes components.

Examples:

- CoreDNS
- metrics-server
- kube-proxy
- CNI pods
- AGIC
- monitoring agents
- security agents

---

# Important Recommendation

Microsoft recommends:

```text
Always keep at least 3 system nodes
```

for production HA.

---

# System Node Pool Characteristics

| Feature | System Pool |
|---|---|
| Mode | System |
| Critical workloads | Yes |
| Minimum nodes | Recommended 3 |
| AZ deployment | Recommended |
| Autoscaling | Supported |
| Used for applications | Usually avoided |

---

# Example Terraform System Node Pool

```hcl
default_node_pool {
  name                 = "systempool"
  vm_size              = "Standard_D4s_v5"

  node_count           = 3

  auto_scaling_enabled = true
  min_count            = 3
  max_count            = 10

  mode                 = "System"

  zones = ["1", "2", "3"]

  vnet_subnet_id = azurerm_subnet.aks.id
}
```

---

# 2. User Node Pool

User node pools host:

- Business applications
- APIs
- Frontend apps
- Backend apps
- Batch jobs
- Stateful applications

---

# User Node Pool Characteristics

| Feature | User Pool |
|---|---|
| Mode | User |
| Application workloads | Yes |
| Separate scaling | Yes |
| Dedicated workloads | Yes |
| Taints commonly used | Yes |

---

# Example Terraform User Node Pool

```hcl
resource "azurerm_kubernetes_cluster_node_pool" "userpool" {
  name                  = "userpool"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.aks.id

  vm_size = "Standard_D4s_v5"

  mode = "User"

  enable_auto_scaling = true
  min_count           = 2
  max_count           = 20

  zones = ["1", "2", "3"]

  node_taints = [
    "workload=apps:NoSchedule"
  ]

  vnet_subnet_id = azurerm_subnet.aks.id
}
```

---

# Why Separate System and User Pools?

Benefits:

- Better isolation
- Safer upgrades
- Better scaling
- Prevent application starvation
- Protect critical Kubernetes services

---

# Real Production Architecture

```text
AKS Cluster
   |
--------------------------------
|                              |
System Node Pool         User Node Pool
|                              |
CoreDNS                  Applications
kube-proxy               APIs
Metrics Server           Frontend
CNI Pods                 Backend
AGIC                     Batch Jobs
```

---

# 3. Scheduling Applications on Specific Node Pools

Very important production concept.

Applications should usually run only on:

```text
User Node Pools
```

NOT on:
- system node pools

---

# Why?

Because application workloads can:

- consume resources
- affect CoreDNS
- affect monitoring
- affect cluster stability

---

# 4. Using Taints on System Pools

Common production pattern:

System node pool taint:

```text
CriticalAddonsOnly=true:NoSchedule
```

This prevents normal applications from scheduling there.

---

# 5. Using Node Selectors

Applications can target user pools.

Example:

```yaml
nodeSelector:
  agentpool: userpool
```

---

# 6. Using Node Affinity

More flexible than nodeSelector.

---

# Example Node Affinity

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: agentpool
          operator: In
          values:
          - userpool
```

---

# How It Works

Kubernetes scheduler checks:

```text
Node Label:
agentpool=userpool
```

Then schedules pod only on matching nodes.

---

# 7. System Workload Scheduling

AKS automatically schedules critical workloads on:

```text
System Node Pools
```

Examples:

- CoreDNS
- kube-proxy
- CNI
- metrics-server

---

# 8. User Workload Scheduling

Application workloads schedule on:

```text
User Node Pools
```

using:
- taints
- tolerations
- node selectors
- node affinity

---

# 9. Taints and Tolerations Example

## System Pool Taint

```text
CriticalAddonsOnly=true:NoSchedule
```

---

## Application Without Toleration

Application cannot schedule on system nodes.

---

## Application With Toleration

```yaml
tolerations:
- key: "CriticalAddonsOnly"
  operator: "Exists"
```

Now app CAN schedule there.

Usually avoided unless necessary.

---

# 10. Best Practice During Upgrades

Upgrade order:

```text
1. Control Plane
2. System Node Pools
3. User Node Pools
```

---

# Why Upgrade System Pools First?

Because:
- CoreDNS
- kube-proxy
- CNI

must remain compatible with cluster version.

---

# Important Production Recommendation

Never place:
- heavy applications
- batch jobs
- memory-intensive workloads

on:
- system node pools

---

# 11. Multi-Node Pool Production Pattern

```text
System Pool
  |
CoreDNS
Metrics
CNI
AGIC

User Pool 1
  |
Frontend Apps

User Pool 2
  |
Backend APIs

User Pool 3
  |
Batch Jobs
```

---

# 12. Spot Node Pools

Optional advanced pattern.

Example:

```text
Spot Pool
```

for:
- CI jobs
- batch workloads
- non-critical workloads

---

# 13. Important Interview Questions

## Q1: Why separate system and user node pools?

Answer:

To isolate critical Kubernetes services from application workloads and improve stability, scalability, and upgrade safety.

---

## Q2: Why should apps avoid system node pools?

Answer:

Applications may consume resources and impact critical components like CoreDNS, kube-proxy, and monitoring agents.

---

## Q3: How do applications schedule on specific node pools?

Answer:

Using:
- nodeSelector
- node affinity
- taints/tolerations

---

## Q4: What workloads run on system node pools?

Examples:
- CoreDNS
- kube-proxy
- metrics-server
- Azure CNI
- AGIC

---

# 14. Final Enterprise Scheduling Architecture

```text
Azure Front Door
       |
Application Gateway
       |
AKS Cluster
       |
------------------------------------------------
|                                              |
System Node Pool                         User Node Pools
|                                              |
CoreDNS                                   Frontend Apps
Metrics                                   Backend APIs
CNI                                       Batch Jobs
kube-proxy                                Stateful Apps
AGIC
```

---

# Final Important Understanding

Production AKS clusters should:

- Separate system and user workloads
- Use node affinity
- Use taints/tolerations
- Use topology spread constraints
- Use PDBs
- Use surge upgrades
- Use multi-AZ node pools

This provides:
- safer upgrades
- better isolation
- improved HA
- enterprise-grade Kubernetes operations
