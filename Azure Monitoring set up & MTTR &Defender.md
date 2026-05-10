# Complete Azure Monitoring, AKS Observability, Defender and Troubleshooting Guide

# 1. Introduction

This document explains complete enterprise Azure monitoring architecture including:

- AKS monitoring
- Container Insights
- Azure Monitor
- Log Analytics Workspace
- Application Insights
- Prometheus and Grafana
- Alerting architecture
- MTTR reduction strategies
- Defender for Cloud
- Defender for Containers
- ACR image vulnerability scanning
- Database latency troubleshooting
- Real-time production troubleshooting examples
- Terraform configurations
- Cost optimization for monitoring
- Security monitoring

This guide is useful for:

- DevOps interviews
- SRE interviews
- Azure operations
- Platform engineering
- Enterprise AKS monitoring

---

# 2. Enterprise Monitoring Architecture

```text
Applications
      |
AKS Cluster
      |
Azure Monitor Agent
      |
Container Insights
      |
Log Analytics Workspace
      |
Azure Monitor
      |
Alerts / Dashboards
      |
Teams / PagerDuty / Email
```

---

# 3. Monitoring Components

| Component | Purpose |
|---|---|
| Azure Monitor | Metrics and logs |
| Log Analytics | Central log storage |
| Container Insights | AKS monitoring |
| App Insights | Application telemetry |
| Prometheus | Kubernetes metrics |
| Grafana | Visualization |
| Defender | Security monitoring |

---

# 4. AKS Monitoring Setup

# Step 1 - Create Log Analytics Workspace

Terraform:

```hcl
resource "azurerm_log_analytics_workspace" "law" {
  name                = "prod-law"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  sku               = "PerGB2018"
  retention_in_days = 30
}
```

---

# Why LAW Required?

Centralized storage for:
- AKS logs
- metrics
- Azure diagnostics
- Container Insights

---

# 5. Enable Container Insights on AKS

Terraform:

```hcl
resource "azurerm_kubernetes_cluster" "aks" {

  name                = "prod-aks"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = "prodaks"

  default_node_pool {
    name       = "systempool"
    node_count = 3
    vm_size    = "Standard_D4s_v5"
  }

  oms_agent {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.law.id
  }

  identity {
    type = "SystemAssigned"
  }
}
```

---

# What Container Insights Collects

- Node metrics
- Pod metrics
- Container logs
- CPU/memory usage
- Restart counts
- Network metrics

---

# 6. Container Insights Architecture

```text
AKS Nodes
     |
Azure Monitor Agent
     |
Container Insights
     |
Log Analytics Workspace
```

---

# 7. Monitoring AKS Metrics

Examples:

| Metric | Usage |
|---|---|
| Node CPU | Capacity planning |
| Node Memory | OOM prevention |
| Pod Restarts | App health |
| Pending Pods | Scheduling issues |
| Disk Usage | Node pressure |

---

# 8. Important Kubernetes Monitoring Alerts

| Alert | Meaning |
|---|---|
| Node NotReady | Node issue |
| Pod CrashLoopBackOff | App crash |
| High CPU | Scaling issue |
| High Memory | Memory leak |
| Failed Scheduling | Capacity issue |

---

# 9. Azure Monitor Alerts

# Example CPU Alert

Terraform:

```hcl
resource "azurerm_monitor_metric_alert" "aks_cpu" {

  name                = "aks-high-cpu"
  resource_group_name = azurerm_resource_group.main.name
  scopes              = [azurerm_kubernetes_cluster.aks.id]

  criteria {
    metric_namespace = "Microsoft.ContainerService/managedClusters"
    metric_name      = "node_cpu_usage_percentage"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = 80
  }

  frequency   = "PT5M"
  window_size = "PT15M"

  severity = 2
}
```

---

# What Happens

If CPU > 80%:
- alert triggers
- notification sent

---

# 10. Log-Based Alerts

Example:
- Pod failures
- API exceptions
- HTTP 500 errors

---

# KQL Example

```kql
ContainerLog
| where LogEntry contains "ERROR"
```

---

# Terraform Log Alert

```hcl
resource "azurerm_monitor_scheduled_query_rules_alert" "pod_errors" {

  name                = "pod-errors"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  data_source_id = azurerm_log_analytics_workspace.law.id

  query = <<QUERY
ContainerLog
| where LogEntry contains "ERROR"
QUERY

  severity    = 2
  frequency   = 5
  time_window = 15
}
```

---

# 11. Alert Severity Levels

| Severity | Meaning |
|---|---|
| Sev0 | Full outage |
| Sev1 | Critical |
| Sev2 | Major degradation |
| Sev3 | Minor issue |

---

# 12. Alert Notification Channels

Alerts can trigger:
- Email
- Teams
- PagerDuty
- ServiceNow
- Webhooks

---

# Action Group Example

```hcl
resource "azurerm_monitor_action_group" "ops" {

  name                = "ops-alerts"
  resource_group_name = azurerm_resource_group.main.name
  short_name          = "ops"

  email_receiver {
    name          = "devops"
    email_address = "devops@contoso.com"
  }
}
```

---

# 13. MTTR Reduction Strategy

# Additional Important MTTR Interview Explanation

# How We Actually Achieve Low MTTR in Production

MTTR reduction is not achieved using only one tool.

It depends on:

- identifying different failure scenarios
- configuring proper monitoring
- creating targeted alerts
- defining remediation steps
- automating recovery wherever possible

---

# Important Production Understanding

Different failures require:
- different alerts
- different thresholds
- different remediation actions

because:
- not all incidents behave the same way

---

# Real Enterprise MTTR Strategy

```text
Failure Type
     |
Monitoring
     |
Alert Rule
     |
Notification
     |
Runbook / Automation
     |
Recovery
```

---

# 1. Infrastructure-Level MTTR

## Example Issues

- Node down
- VM CPU spike
- Disk full
- Memory pressure

---

# Monitoring Used

- Azure Monitor
- Container Insights
- Prometheus

---

# Example Alerts

## CPU Alert

```text
CPU > 85% for 10 mins
```

Action:
- autoscale nodes
- notify SRE team

---

## Node NotReady Alert

```text
Kubernetes Node = NotReady
```

Action:
- cordon/drain node
- autoscaler creates replacement

---

# MTTR Reduction Technique

- proactive detection
- auto-healing
- cluster autoscaler
- node auto repair

---

# 2. Application-Level MTTR

## Example Issues

- HTTP 500 spike
- pod crashes
- API latency increase

---

# Monitoring Used

- Application Insights
- Prometheus
- Grafana
- Log Analytics

---

# Example Alerts

## Error Rate Alert

```text
HTTP 500 > 5%
```

---

## Pod CrashLoop Alert

```text
Pod restart count > threshold
```

---

# Actions Taken

- restart deployment
- rollback release
- scale replicas
- investigate logs

---

# MTTR Reduction Technique

- health probes
- blue-green deployment
- canary rollout
- auto rollback

---

# 3. Database-Level MTTR

## Example Issues

- slow queries
- DB CPU high
- connection exhaustion
- deadlocks

---

# Monitoring Used

- Application Insights dependency tracking
- SQL Insights
- Query Store

---

# Example Alerts

## Query Latency Alert

```text
SQL dependency latency > 2 sec
```

---

## DB CPU Alert

```text
DB CPU > 90%
```

---

# Actions Taken

- identify slow query
- add indexes
- scale database
- enable caching

---

# MTTR Reduction Technique

- query optimization
- indexing
- read replicas
- Redis caching

---

# 4. Kubernetes-Level MTTR

## Example Issues

- pending pods
- failed scheduling
- image pull failures

---

# Example Alerts

## Pending Pods Alert

```text
Pending pods > threshold
```

---

## ImagePullBackOff Alert

```text
Container image pull failure
```

---

# Actions Taken

- increase node capacity
- fix ACR permissions
- verify node selectors/taints

---

# MTTR Reduction Technique

- autoscaling
- proper node pools
- ACR monitoring
- predeployment validation

---

# 5. Security-Level MTTR

## Example Issues

- CVE detected
- malware upload
- suspicious login

---

# Monitoring Used

- Microsoft Defender
- Defender for Containers
- Defender for SQL

---

# Example Alerts

## Critical CVE Alert

```text
Critical vulnerability detected in nginx image
```

---

# Actions Taken

- patch base image
- rebuild container
- redeploy workloads

---

# MTTR Reduction Technique

- image scanning
- automated patching
- CI/CD security gates

---

# 6. Networking-Level MTTR

## Example Issues

- ingress failure
- DNS resolution issue
- backend unhealthy

---

# Example Alerts

## Front Door Backend Unhealthy

```text
Backend health probe failed
```

---

## DNS Resolution Failure

```text
CoreDNS errors spike
```

---

# Actions Taken

- failover region
- restart CoreDNS
- fix ingress/backend

---

# MTTR Reduction Technique

- multi-region architecture
- health probes
- traffic failover
- redundant DNS

---

# Real Enterprise MTTR Approach

The key idea is:

```text
Faster Detection
      +
Faster Isolation
      +
Faster Remediation
      =
Lower MTTR
```

---

# How Monitoring Helps MTTR

| Monitoring Capability | MTTR Benefit |
|---|---|
| Alerts | Faster detection |
| Dashboards | Faster visibility |
| Distributed tracing | Faster RCA |
| Logs | Faster debugging |
| Health probes | Faster failover |
| Autoscaling | Automatic recovery |

---

# Important Interview-Level Understanding

Good MTTR is not only about:
- fixing issues quickly

It is mainly about:
- detecting issues early
- isolating root cause fast
- automating remediation

---

# Strong Interview Answer

“How we achieve low MTTR depends on the type of failure. We configure different monitoring and alerting strategies for infrastructure, applications, databases, Kubernetes, networking, and security layers. For example, we configure CPU, memory, and node health alerts for infrastructure; HTTP 500 and latency alerts for APIs; dependency latency and slow query alerts for databases; and CVE alerts using Defender for Containers. We integrate Azure Monitor, Application Insights, Log Analytics, Prometheus, Grafana, and Defender with automated remediation, autoscaling, health probes, and incident notifications to reduce detection and recovery time significantly.”
---

# 14. Enterprise Monitoring Flow

```text
Issue
   |
Azure Monitor Detects
   |
Alert Triggered
   |
Teams/PagerDuty
   |
Engineer Investigates
   |
Mitigation Applied
   |
Service Restored
```

---

# 15. Application Insights Setup

Used for:
- distributed tracing
- API telemetry
- dependency tracking
- DB latency tracking

---

# Terraform

```hcl
resource "azurerm_application_insights" "appi" {

  name                = "prod-appi"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  application_type = "web"
}
```

---

# 16. What App Insights Tracks

| Metric | Example |
|---|---|
| Response time | API latency |
| Failed requests | HTTP 500 |
| Dependencies | DB calls |
| Exceptions | Application errors |

---

# 17. Real-Time Database Latency Example

# Problem

Users report:

```text
API response taking 8 seconds
```

---

# Investigation Flow

```text
Client
   |
Frontend API
   |
Backend API
   |
Database
```

---

# Step 1 - Check App Insights

Observe:
- API latency spike

---

# Step 2 - Dependency Analysis

Application Insights shows:

```text
SQL dependency latency = 7 seconds
```

Clearly identifies DB bottleneck.

---

# Step 3 - Check SQL Metrics

Check:
- CPU
- DTU/vCore
- locks
- deadlocks
- slow queries

---

# Step 4 - Query Store Analysis

Find:

```sql
SELECT * FROM orders
WHERE customer_name LIKE '%john%'
```

Causing:
- table scan
- high latency

---

# Root Cause

Missing index.

---

# Fix

Create index:

```sql
CREATE INDEX idx_customer_name
ON orders(customer_name);
```

---

# Result

Latency reduced:

```text
7 seconds --> 200 ms
```

---

# 18. Common DB Latency Causes

| Cause | Example |
|---|---|
| Missing indexes | Slow queries |
| N+1 queries | ORM issue |
| Connection exhaustion | Too many connections |
| Cross-region DB | Network latency |
| Deadlocks | Transaction blocking |

---

# 19. DB Latency Prevention

## Recommended

- Proper indexing
- Read replicas
- Redis caching
- Query optimization
- Connection pooling
- Regional placement

---

# 20. Azure SQL Monitoring

Monitor:
- DTU/vCore
- storage
- deadlocks
- CPU
- query duration

---

# 21. ACR Monitoring and Security

Monitor:
- image push/pull
- failed pulls
- vulnerability scans

---

# 22. Enable Defender for Cloud

Terraform:

```hcl
resource "azurerm_security_center_subscription_pricing" "containers" {

  tier          = "Standard"
  resource_type = "Containers"
}
```

---

# Important Understanding

Defender plans must usually be:
- explicitly enabled

Not all advanced protections enabled automatically.

---

# 23. Enable Defender for Containers

Provides:
- AKS runtime protection
- Kubernetes recommendations
- image vulnerability scanning

---

# 24. ACR Vulnerability Scanning

Flow:

```text
Docker Image
      |
Push to ACR
      |
Defender Scan
      |
CVE Detection
```

---

# Example CVE

```text
CVE-2024-12345
```

Detected in:

```dockerfile
FROM ubuntu:18.04
```

---

# Defender Recommendation

```text
Upgrade base image
```

---

# Fixed Image

```dockerfile
FROM ubuntu:22.04
```

---

# 25. Enable Defender for ACR

Terraform:

```hcl
resource "azurerm_security_center_subscription_pricing" "acr" {

  tier          = "Standard"
  resource_type = "ContainerRegistry"
}
```

---

# 26. Enable Defender for SQL

Terraform:

```hcl
resource "azurerm_security_center_subscription_pricing" "sql" {

  tier          = "Standard"
  resource_type = "SqlServers"
}
```

---

# Defender for SQL Detects

- SQL injection
- suspicious logins
- anomalous behavior

---

# 27. Enable Defender for Storage

Terraform:

```hcl
resource "azurerm_security_center_subscription_pricing" "storage" {

  tier          = "Standard"
  resource_type = "StorageAccounts"
}
```

---

# Defender for Storage Detects

- malware uploads
- suspicious access
- data exfiltration

---

# 28. Enable Defender for Kubernetes

Terraform:

```hcl
resource "azurerm_security_center_subscription_pricing" "k8s" {

  tier          = "Standard"
  resource_type = "KubernetesService"
}
```

---

# Defender Kubernetes Features

- runtime threat detection
- RBAC recommendations
- pod security findings
- exposed dashboard detection

---

# 29. Defender Architecture

```text
AKS
  |
Defender Agent
  |
Security Events
  |
Defender for Cloud
  |
Security Recommendations
```

---

# 30. Azure Resource Monitoring

# Storage Account Monitoring

Monitor:
- ingress/egress
- transactions
- latency
- availability

---

# Database Monitoring

Monitor:
- query latency
- deadlocks
- CPU
- connection usage

---

# Front Door Monitoring

Monitor:
- backend health
- latency
- WAF events
- traffic patterns

---

# App Gateway Monitoring

Monitor:
- backend health
- failed requests
- response time

---

# 31. Prometheus + Grafana in AKS

Common enterprise pattern.

---

# Architecture

```text
AKS
   |
Prometheus
   |
Grafana
```

---

# Prometheus Collects

- Kubernetes metrics
- node metrics
- pod metrics
- application metrics

---

# Grafana Dashboards

Used for:
- real-time visualization
- SRE dashboards
- NOC monitoring

---

# 32. Cost Optimization for Monitoring

Very important.

---

# Common Monitoring Cost Drivers

| Service | Cost Driver |
|---|---|
| Log Analytics | Log ingestion |
| App Insights | Telemetry volume |
| Defender | Enabled plans |
| Prometheus | Storage |

---

# Cost Reduction Techniques

## Recommended

- Reduce log retention
- Filter noisy logs
- Use sampling
- Archive old logs
- Avoid debug logs in prod

---

# Example App Insights Sampling

```text
Only collect 20% traces
```

reduces cost.

---

# 33. Enterprise Incident Response Example

# Scenario

Users report:
- API timeout
- payment failure

---

# Investigation

## Azure Monitor

Alert triggered:
- High API latency

---

## App Insights

Dependency latency:
- SQL query slow

---

## SQL Metrics

CPU:
- 95%

---

## Query Store

Problem query identified.

---

## Mitigation

- Add index
- Scale DB
- Enable caching

---

# Result

MTTR reduced:
- from 2 hours
- to 15 minutes

---

# 34. Validation Commands

## Check AKS Metrics

```bash
kubectl top nodes
kubectl top pods
```

---

## Check Container Insights

```bash
kubectl get pods -n kube-system
```

---

## Check Defender Recommendations

Azure Portal:

```text
Defender for Cloud
```

---

## Check Logs

KQL:

```kql
ContainerLog
| limit 50
```

---

# 35. Important Interview Questions

## Q1: How do you reduce MTTR?

Answer:

Using proactive monitoring, centralized logging, automated alerting, dashboards, distributed tracing, runbooks, and rapid RCA workflows.

---

## Q2: How do you monitor AKS?

Answer:

Using Container Insights, Azure Monitor, Log Analytics, Prometheus, Grafana, Application Insights, and Defender for Containers.

---

## Q3: How do you troubleshoot DB latency?

Answer:

Using dependency tracking, App Insights, SQL Query Store, DB metrics, slow query analysis, indexing checks, and connection pool analysis.

---

## Q4: Is Defender automatically enabled?

Answer:

Basic recommendations may exist, but advanced Defender plans for Containers, SQL, Storage, and ACR typically require explicit enablement.

---

## Q5: How does ACR vulnerability scanning work?

Answer:

Defender scans images pushed to ACR, compares packages against CVE databases, and reports vulnerabilities with severity and remediation guidance.

---

# 36. Final Enterprise Monitoring Architecture

```text
Users
   |
Azure Front Door
   |
Application Gateway
   |
AKS Cluster
   |
Container Insights
   |
Azure Monitor
   |
Log Analytics
   |
App Insights
   |
Defender for Cloud
   |
Alerts + Dashboards
```

---

# 37. Final Interview Summary

“In production AKS environments we use Azure Monitor, Container Insights, Log Analytics, Application Insights, Prometheus, and Grafana for full-stack observability. We configure metric, log, availability, and security alerts integrated with Teams and PagerDuty to reduce MTTR. Application Insights dependency tracking helps identify database bottlenecks and slow queries in real time. We enable Microsoft Defender for Containers, ACR, SQL, Storage, and Kubernetes for vulnerability scanning, runtime protection, and CVE detection. Terraform is used to provision monitoring resources, alerts, Defender plans, and observability infrastructure consistently across environments.”

---

# Conclusion

This monitoring and security architecture provides:

- Centralized observability
- Faster troubleshooting
- Lower MTTR
- Real-time AKS monitoring
- Enterprise-grade alerting
- Secure container workloads
- CVE detection
- Database performance visibility
- Production-grade cloud operations

These concepts are heavily used in enterprise Azure environments and are very common DevOps, SRE, platform engineering, and cloud architect interview topics.
