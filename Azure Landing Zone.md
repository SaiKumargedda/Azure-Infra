# Azure Landing Zone - Complete README

## Introduction

An **Azure Landing Zone (ALZ)** is a secure, scalable, and governed
Azure environment designed according to Microsoft's Cloud Adoption
Framework (CAF). It provides the foundational infrastructure on which
Azure workloads such as AKS, Virtual Machines, App Services, Databases,
and Storage Accounts are deployed.

------------------------------------------------------------------------

## Objectives

-   Standardize Azure deployments
-   Enforce governance and compliance
-   Improve security
-   Centralize networking
-   Enable monitoring and logging
-   Control cloud costs
-   Support Infrastructure as Code (IaC)

------------------------------------------------------------------------

## High-Level Architecture

``` text
Azure Tenant
│
├── Management Groups
│   ├── Platform
│   ├── Landing Zones
│   ├── Sandbox
│   └── Decommissioned
│
├── Platform Subscriptions
│   ├── Networking
│   ├── Identity
│   └── Management
│
└── Workload Subscriptions
    ├── Development
    ├── Testing
    └── Production
```

------------------------------------------------------------------------

## Core Components

### 1. Management Groups

-   Organize subscriptions
-   Apply Azure Policy
-   Apply RBAC
-   Delegate administration

### 2. Subscriptions

-   Billing boundary
-   Resource isolation
-   Environment separation (Dev/Test/Prod)

### 3. Identity

-   Microsoft Entra ID
-   RBAC
-   Managed Identities
-   Service Principals
-   PIM
-   MFA

### 4. Networking

Hub-and-Spoke topology:

``` text
Hub VNet
│
├── Azure Firewall
├── VPN Gateway
├── ExpressRoute
├── Bastion
└── Private DNS
      │
--------------------------
│          │            │
Dev      Test        Prod
AKS       VMs       Databases
```

### 5. Governance

Typical Azure Policies: - Allowed regions - Allowed VM SKUs - Mandatory
tags - HTTPS only - Deny public IPs - Require diagnostic logs

### 6. Security

-   Defender for Cloud
-   Azure Firewall
-   Key Vault
-   Private Endpoints
-   NSGs
-   DDoS Protection
-   Microsoft Sentinel

### 7. Monitoring

-   Azure Monitor
-   Log Analytics Workspace
-   Application Insights
-   Container Insights
-   Alerts

### 8. Cost Management

-   Budgets
-   Cost Alerts
-   Chargeback
-   Showback
-   Resource tagging

------------------------------------------------------------------------

## Terraform Folder Structure

``` text
terraform/
├── modules/
│   ├── networking/
│   ├── policy/
│   ├── keyvault/
│   ├── monitoring/
│   ├── defender/
│   └── management-groups/
├── environments/
│   ├── dev/
│   ├── test/
│   └── prod/
├── backend.tf
├── providers.tf
├── variables.tf
└── outputs.tf
```

------------------------------------------------------------------------

## Azure DevOps Deployment Flow

``` text
Developer
    │
Azure Repos
    │
Pull Request
    │
Branch Policies
    │
Azure Pipelines
    │
Terraform Validate
    │
Terraform Plan
    │
Approval
    │
Terraform Apply
    │
Landing Zone
    │
AKS / Applications
```

------------------------------------------------------------------------

## Benefits

-   Standardized infrastructure
-   Secure-by-default deployments
-   Centralized governance
-   Simplified compliance
-   Better scalability
-   Lower operational overhead
-   Consistent networking
-   Centralized monitoring

------------------------------------------------------------------------

## Interview Questions

### What is an Azure Landing Zone?

A secure and governed Azure platform foundation that provides
networking, identity, governance, monitoring, and security before
workloads are deployed.

### Why is it used?

To standardize cloud deployments, improve security, enforce governance,
and simplify operations.

### How is it deployed?

Typically using Terraform or Bicep integrated with Azure DevOps or
GitHub Actions.

### Can AKS be deployed without a Landing Zone?

Yes, but enterprise environments recommend deploying AKS inside a
Landing Zone so it inherits centralized networking, security,
monitoring, and governance.

------------------------------------------------------------------------

## Real-Time Enterprise Flow

1.  Build the Landing Zone using Terraform.
2.  Configure Management Groups and subscriptions.
3.  Deploy Hub networking and shared services.
4.  Configure Azure Policies and RBAC.
5.  Enable Azure Monitor and Defender.
6.  Create AKS clusters in workload subscriptions.
7.  Deploy applications through Azure DevOps CI/CD.
