# Azure Policy for AKS – Design, Deployment, and Real-Time Implementation

## Overview

Azure Policy is a governance service in Azure that helps organizations enforce security, compliance, and operational standards across Azure resources. It evaluates resources during creation and updates to ensure they comply with organizational policies.

For Azure Kubernetes Service (AKS), Azure Policy is commonly used to enforce security baselines, networking standards, monitoring configurations, and compliance requirements.

---

# Real-Time Business Scenario

Suppose an organization has multiple development teams deploying AKS clusters. To maintain security and compliance, the platform team defines the following standards:

* AKS clusters must be deployed only in approved Azure regions.
* Only private AKS clusters are allowed.
* Azure Defender for Kubernetes must be enabled.
* Azure Monitor and Container Insights must be configured.
* Only approved VM sizes can be used for node pools.
* Required tags (Environment, Project, Owner, CostCenter) must be present.
* Diagnostic logs must be enabled.
* Privileged containers should not be allowed.
* HTTPS and managed identities must be enforced.

Instead of relying on developers to manually follow these standards, Azure Policy automatically validates every deployment.

---

# Azure Policy Design

Azure Policy can be assigned at different scopes.

```
Management Group
       │
Subscription
       │
Resource Group
       │
Individual Resource
```

### Typical Enterprise Design

* Management Group → Organization-wide policies
* Subscription → Environment-specific policies
* Resource Group → Application-specific policies
* Resource → Rarely used

In most enterprise environments, AKS policies are assigned at the Subscription level so that every AKS cluster created within the subscription follows the same governance rules.

---

# Step 1 – Define Governance Requirements

Before creating policies, identify the organization's requirements.

Example:

* Allow AKS only in Central India.
* Enforce private clusters.
* Enable Azure Monitor.
* Enable Microsoft Defender.
* Require mandatory resource tags.
* Restrict node pool VM sizes.
* Enforce supported Kubernetes versions.
* Block privileged containers.
* Enable diagnostic settings.

---

# Step 2 – Choose Policy Type

Azure provides two types of policies.

### Built-in Policies

Microsoft provides hundreds of built-in policies for Azure resources.

Examples:

* AKS clusters should be private.
* Azure Monitor should be enabled.
* Kubernetes version should be supported.
* Diagnostic settings should exist.
* Defender should be enabled.
* Only approved VM SKUs.

These policies are ready to use.

### Custom Policies

If organizational requirements are unique, custom JSON policies can be created.

Example:

* Allow deployment only in Central India.
* Restrict custom VM SKUs.
* Require company-specific tags.
* Restrict container images to an approved Azure Container Registry.

---

# Step 3 – Create Policy Definition

A policy definition consists of an IF condition and a THEN effect.

Example:

```
IF

Resource Type = AKS

AND

Location != Central India

THEN

Deny
```

If a developer attempts to deploy AKS in East US, Azure immediately blocks the deployment.

Error:

```
RequestDisallowedByPolicy
```

---

# Step 4 – Assign the Policy

Once the policy is created, assign it to the required scope.

Example:

```
Subscription

        │

AKS Security Policy
```

Now every AKS deployment in that subscription is automatically evaluated.

---

# Step 5 – Deploy Infrastructure

Developer executes:

```
terraform apply
```

Terraform sends the deployment request to Azure Resource Manager.

Azure Resource Manager invokes Azure Policy.

Policy evaluation begins.

```
Terraform

      │

terraform apply

      │

Azure Resource Manager

      │

Azure Policy Engine

      │

Policy Evaluation

      │

Compliant ?

 YES              NO

Deploy       Deny / Audit / Modify /
             DeployIfNotExists
```

---

# Policy Effects

## 1. Deny

Stops deployment immediately.

Examples:

* Wrong region
* Public AKS cluster
* Unsupported Kubernetes version
* Unapproved VM size

Example:

```
private_cluster_enabled = false

↓

Deployment Failed

↓

RequestDisallowedByPolicy
```

---

## 2. Audit

Allows deployment but marks the resource as non-compliant.

Example:

Azure Portal

```
Compliance

85%

Non-Compliant Resources

AKS-Cluster-02
```

Operations teams can later remediate these issues.

---

## 3. Modify

Automatically updates resource properties.

Example:

Developer forgets required tags.

```
Environment

Owner

Project
```

Azure Policy automatically adds them during deployment.

---

## 4. Append

Adds required properties before deployment.

Examples:

* Managed Identity
* Network Policy
* Security Configuration

---

## 5. DeployIfNotExists

Automatically deploys required configurations if missing.

Example:

Developer deploys AKS without monitoring.

Azure Policy automatically deploys:

* Azure Monitor Agent
* Container Insights
* Data Collection Rule
* Diagnostic Settings

No manual intervention is required.

---

## 6. AuditIfNotExists

Checks whether required resources exist.

Example:

Diagnostic Settings missing

↓

Resource is marked as Non-Compliant

---

# Example Scenarios

## Scenario 1 – Restrict Region

Requirement:

AKS must only be deployed in Central India.

Developer deploys AKS in East US.

Azure Policy evaluates the request.

Result:

Deployment denied.

---

## Scenario 2 – Private AKS Only

Requirement:

Only private AKS clusters are allowed.

Developer sets:

```
private_cluster_enabled = false
```

Azure Policy:

```
IF

Private Cluster = False

THEN

Deny
```

Deployment fails.

---

## Scenario 3 – Mandatory Tags

Developer creates an AKS cluster without tags.

Azure Policy automatically adds:

* Environment
* Project
* Owner
* CostCenter

using the Modify effect.

---

## Scenario 4 – Azure Monitor

Developer deploys AKS without enabling monitoring.

Azure Policy detects the missing configuration.

Effect:

DeployIfNotExists

Azure automatically installs:

* Azure Monitor Agent
* Container Insights
* Diagnostic Settings

---

## Scenario 5 – Defender

Developer forgets to enable Microsoft Defender.

Azure Policy automatically enables Defender using DeployIfNotExists.

---

# Enterprise Policy Architecture

```
Management Group

       │

Security Initiative

       │

├── Allowed Regions
├── Private AKS Only
├── Azure Monitor Enabled
├── Microsoft Defender Enabled
├── Mandatory Resource Tags
├── Approved VM Sizes
├── Diagnostic Settings
├── Supported Kubernetes Version
├── No Privileged Containers
├── HTTPS Only
├── Managed Identity Required

       │

Assigned To

Production Subscription

       │

AKS Resource Groups

       │

AKS Clusters
```

Instead of assigning multiple individual policies, organizations typically group related policies into an Initiative (Policy Set) and assign the initiative to the required scope.

---

# Terraform Example

Assign a policy at Resource Group level:

```hcl
resource "azurerm_resource_group_policy_assignment" "aks_security" {
  name                 = "aks-security-policy"
  resource_group_id    = azurerm_resource_group.aks.id
  policy_definition_id = data.azurerm_policy_definition.private_aks.id
}
```

Assign a policy at Subscription level:

```hcl
resource "azurerm_subscription_policy_assignment" "aks_security" {
  name                 = "aks-security"
  subscription_id      = data.azurerm_subscription.current.id
  policy_definition_id = data.azurerm_policy_definition.private_aks.id
}
```

---

# Advantages of Azure Policy

* Enforces organizational standards automatically.
* Prevents non-compliant resources from being deployed.
* Improves security posture.
* Ensures regulatory compliance.
* Reduces manual governance effort.
* Automatically remediates configuration drift.
* Provides centralized compliance reporting.
* Integrates seamlessly with Azure Resource Manager and Terraform.

---

# Interview Answer (2–3 Minutes)

"In our Azure environment, we used Azure Policy to enforce governance and security standards across AKS and other Azure resources. We assigned policies mainly at the subscription level so that every team deploying infrastructure followed the same organizational baseline. We used built-in policies wherever possible and grouped them into initiatives for easier management. For AKS, we enforced private clusters, approved regions, supported Kubernetes versions, mandatory resource tags, Azure Monitor, Microsoft Defender, and diagnostic settings. These policies were assigned through Terraform as part of our Infrastructure as Code deployment. During every deployment, Azure Resource Manager invokes the Azure Policy engine to evaluate the request. Depending on the configured effect, the deployment is either allowed, denied, audited, automatically modified, or remediated using DeployIfNotExists. This approach ensured that all AKS clusters complied with our organization's security and governance standards before they were deployed into production."
