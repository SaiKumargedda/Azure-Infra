# Complete AKS Upgrade and OS Image Patching Guide

# 1. Introduction

This document explains complete AKS upgrade and patching concepts including:

- Kubernetes version upgrades
- AKS control plane upgrades
- Node pool upgrades
- OS image patching
- Surge upgrades
- PodDisruptionBudgets (PDB)
- Cordon and drain
- Node affinity
- Deprecated API checks
- Upgrade sequencing
- Zero downtime upgrades
- Rollback considerations
- Production best practices
- Interview concepts

This guide is useful for:

- AKS administration
- DevOps interviews
- Platform engineering
- Production Kubernetes operations
- SRE preparation

---

# 2. Types of AKS Upgrades

| Upgrade Type | Description |
|---|---|
| Control Plane Upgrade | Upgrades Kubernetes master/API server |
| Node Pool Upgrade | Upgrades worker nodes |
| OS Image Upgrade | Updates VM OS image/security patches |
| Addon Upgrade | CoreDNS, kube-proxy, CNI plugins |
| AKS Version Upgrade | Kubernetes minor/patch version |

---

# 3. Important Kubernetes Version Rules

## Kubernetes Minor Versions

Example:

```text
1.27 --> 1.28
```

This is a:

```text
Minor Version Upgrade
```

---

## Kubernetes Patch Versions

Example:

```text
1.28.3 --> 1.28.5
```

This is a:

```text
Patch Upgrade
```

---

# 4. Important AKS Upgrade Rule

AKS supports:

```text
One minor version upgrade at a time
```

Example:

Allowed:

```text
1.27 --> 1.28
```

Not allowed:

```text
1.27 --> 1.30
```

Need sequential upgrades.

---

# 5. Pre-Upgrade Checklist

Before upgrading ALWAYS check:

| Check | Why Important |
|---|---|
| Deprecated APIs | Prevent app failures |
| PDBs | Avoid downtime |
| Node capacity | Ensure surge nodes fit |
| Helm compatibility | Prevent chart failures |
| CRDs compatibility | API changes |
| Ingress compatibility | Networking issues |
| Backup strategy | Recovery safety |
| Monitoring | Observe upgrade |
| Resource quotas | Prevent scheduling failures |

---

# 6. Check Current AKS Version

```bash
az aks show \
  --resource-group prod-rg \
  --name prod-aks \
  --query kubernetesVersion
```

---

# 7. Check Available Upgrades

```bash
az aks get-upgrades \
  --resource-group prod-rg \
  --name prod-aks
```

---

# 8. Important Deprecated API Check

Very important before upgrades.

---

# Why?

Some APIs removed in newer Kubernetes versions.

Example:

```text
extensions/v1beta1
```

removed.

---

# 9. Check Deprecated APIs

## Using kubectl

```bash
kubectl get apiservices
```

---

## Using Pluto Tool

Very common production tool.

```bash
pluto detect-all-in-cluster
```

---

## Using Kubent

```bash
kubent
```

---

# 10. Example Deprecated API Problem

Old:

```yaml
apiVersion: extensions/v1beta1
```

New:

```yaml
apiVersion: networking.k8s.io/v1
```

---

# 11. Backup Before Upgrade

Recommended:

- Velero backup
- GitOps manifests
- Helm values backup
- etcd snapshots (self-managed clusters)

---

# 12. Understand Control Plane Upgrade

Control Plane includes:

- API server
- Scheduler
- Controller manager

Azure upgrades these managed components.

---

# 13. Upgrade Control Plane

```bash
az aks upgrade \
  --resource-group prod-rg \
  --name prod-aks \
  --kubernetes-version 1.28.5 \
  --control-plane-only
```

---

# 14. Why Upgrade Control Plane First?

Because:

```text
Control Plane Version >= Node Version
```

must be maintained.

---

# 15. Node Pool Upgrade

After control plane upgrade:

upgrade node pools.

---

# 16. Upgrade Node Pool

```bash
az aks nodepool upgrade \
  --resource-group prod-rg \
  --cluster-name prod-aks \
  --name systempool \
  --kubernetes-version 1.28.5
```

---

# 17. How AKS Node Upgrade Works Internally

AKS performs:

1. Add surge node
2. Cordon old node
3. Drain workloads
4. Recreate node
5. Schedule workloads
6. Remove old node

---

# 18. Important Upgrade Components

| Component | Purpose |
|---|---|
| Surge Node | Temporary extra node |
| Cordon | Prevent new pods |
| Drain | Evict pods |
| PDB | Protect availability |
| Scheduler | Reschedule workloads |

---

# 19. What is Cordon?

Cordon marks node as:

```text
Unschedulable
```

No new pods placed.

---

# Cordon Example

```bash
kubectl cordon aks-nodepool1-12345678-vmss000001
```

---

# 20. What is Drain?

Drain safely evicts pods from node.

---

# Drain Example

```bash
kubectl drain aks-nodepool1-12345678-vmss000001 \
  --ignore-daemonsets \
  --delete-emptydir-data
```

---

# 21. What is Surge Upgrade?

AKS temporarily creates extra nodes during upgrade.

Helps avoid downtime.

---

# Example

If cluster has:

```text
3 nodes
```

With:

```text
max surge = 1
```

AKS creates:

```text
4th temporary node
```

before removing old node.

---

# 22. Configure Max Surge

```bash
az aks nodepool update \
  --resource-group prod-rg \
  --cluster-name prod-aks \
  --name systempool \
  --max-surge 33%
```

---

# 23. Why Surge Important?

Without surge:

- workloads may become unavailable
- pods may fail scheduling

With surge:
- safer rolling upgrade
- higher availability

---

# 24. PodDisruptionBudget (PDB)

Very important production concept.

PDB protects application availability during upgrades.

---

# Example PDB

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget

metadata:
  name: app-pdb

spec:
  minAvailable: 2

  selector:
    matchLabels:
      app: myapp
```

---

# What PDB Does

If deployment has:

```text
3 pods
```

and:

```text
minAvailable = 2
```

Kubernetes allows only:
- 1 pod disruption at a time

---

# 25. Why PDB Important During Upgrade?

During node drain:

pods get evicted.

PDB ensures:
- enough pods stay running
- prevents downtime

---

# 26. Node Affinity During Upgrade

Node affinity controls:
- where pods schedule

---

# Example

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: workload
          operator: In
          values:
          - production
```

---

# Why Important During Upgrade?

If affinity rules are too strict:
- pods may fail scheduling on surge nodes

Very common production issue.

---

# 27. Taints and Tolerations

Important upgrade concept.

---

# Example

```yaml
tolerations:
- key: "critical"
  operator: "Exists"
  effect: "NoSchedule"
```

---

# Why Important?

During upgrades:
- workloads may move to new nodes
- tolerations must allow scheduling

---

# 28. DaemonSets During Upgrade

DaemonSets:
- run on every node

Examples:
- CNI
- monitoring agents
- security agents

Drain usually ignores DaemonSets.

---

# 29. Check Node Health Before Upgrade

```bash
kubectl get nodes
```

Ensure:
- Ready state
- No disk pressure
- No memory pressure

---

# 30. Check Pod Health

```bash
kubectl get pods -A
```

Check:
- CrashLoopBackOff
- Pending pods
- Failed scheduling

---

# 31. Check Resource Usage

```bash
kubectl top nodes
kubectl top pods -A
```

---

# Why Important?

Need enough capacity for:
- surge nodes
- pod rescheduling

---

# 32. Check Cluster Autoscaler

If enabled:

ensure:
- node limits sufficient
- autoscaler healthy

---

# 33. OS Image Upgrade

Different from Kubernetes version upgrade.

OS image patching updates:
- Ubuntu patches
- kernel fixes
- security patches
- container runtime patches

---

# 34. Check Node Image Version

```bash
kubectl get nodes -o wide
```

OR

```bash
az aks nodepool show \
  --resource-group prod-rg \
  --cluster-name prod-aks \
  --name systempool
```

---

# 35. Upgrade Node OS Image

```bash
az aks nodepool upgrade \
  --resource-group prod-rg \
  --cluster-name prod-aks \
  --name systempool \
  --node-image-only
```

---

# Important Understanding

This upgrades:
- VM OS image

WITHOUT:
- changing Kubernetes version

---

# 36. Auto OS Patching

AKS supports:

- Node image auto-upgrade
- Security patching

---

# Example

```bash
az aks auto-upgrade profile update \
  --resource-group prod-rg \
  --name prod-aks \
  --node-os-upgrade-channel NodeImage
```

---

# 37. Upgrade Channels

| Channel | Purpose |
|---|---|
| patch | Kubernetes patches |
| stable | Stable releases |
| rapid | Faster updates |
| node-image | OS image updates |

---

# 38. Zero Downtime Upgrade Strategy

Production best practice:

- Multi-node pools
- Multiple replicas
- PDBs
- Surge upgrades
- Readiness probes
- Liveness probes
- Topology spread constraints

---

# 39. Importance of Readiness Probes

Readiness probes prevent traffic to unhealthy pods.

During upgrade:
- new pod becomes ready
- only then traffic routed

---

# Example Readiness Probe

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
```

---

# 40. Topology Spread Constraints

Ensures pods spread across:
- nodes
- zones

---

# Example

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
```

---

# Why Important During Upgrade?

Prevents:
- all replicas on same node/zone

Improves availability during:
- node upgrades
- zone failures

---

# 41. Multi-AZ Upgrade Strategy

Production recommendation:

```text
AKS across 3 Availability Zones
```

During upgrade:
- workloads continue in healthy zones

---

# 42. Important AKS Addons

Check compatibility for:

- CoreDNS
- kube-proxy
- CSI drivers
- AGIC
- Azure CNI

---

# 43. Verify Upgrade After Completion

## Check Nodes

```bash
kubectl get nodes
```

---

## Check Pods

```bash
kubectl get pods -A
```

---

## Check Events

```bash
kubectl get events -A
```

---

## Check App Health

```bash
curl https://app.contoso.com/health
```

---

# 44. Common Upgrade Problems

| Problem | Cause |
|---|---|
| Pods Pending | Insufficient capacity |
| PDB blocks drain | Strict PDB |
| Failed scheduling | Node affinity/taints |
| App downtime | Single replica |
| Deprecated APIs | Old manifests |
| CrashLoop | App incompatibility |

---

# 45. Production Upgrade Best Practices

## Recommended

- Upgrade non-prod first
- Backup before upgrade
- Use surge upgrades
- Configure PDBs
- Use multiple replicas
- Use readiness probes
- Use topology spread constraints
- Monitor upgrade live
- Validate deprecated APIs
- Upgrade sequentially

---

# 46. Blue-Green Upgrade Strategy

Advanced enterprise pattern.

---

## Flow

```text
Old Cluster --> Active
New Cluster --> Upgrade/Test
```

Switch traffic after validation.

---

# Benefits

- Safer upgrades
- Easier rollback
- Minimal downtime

---

# 47. Canary Upgrade Strategy

Upgrade small subset first.

Example:

```text
10% traffic --> new version
```

Then increase gradually.

---

# 48. Rollback Considerations

Kubernetes version downgrade:
- limited support
- difficult

Better approach:
- backup
- blue-green
- GitOps rollback

---

# 49. Real Enterprise Upgrade Flow

```text
Check APIs
   |
Backup
   |
Upgrade Control Plane
   |
Upgrade System Node Pool
   |
Upgrade User Node Pools
   |
Validate Health
   |
Upgrade OS Images
   |
Post Validation
```

---

# 50. Complete Production Upgrade Architecture

```text
Azure Front Door
       |
Application Gateway
       |
AKS Cluster
       |
Multiple Node Pools
       |
Pods Spread Across AZs
       |
PDB + Surge + Readiness Probes
```

---

# 51. Important Interview Questions

## Q1: Why control plane upgraded first?

Answer:

Control plane version must always be greater than or equal to node versions.

---

## Q2: What is max surge?

Answer:

Temporary extra nodes added during upgrade to reduce downtime and maintain capacity.

---

## Q3: Why PDB important?

Answer:

PDB prevents too many pods from being disrupted simultaneously during node drain or upgrades.

---

## Q4: Difference between cordon and drain?

| Cordon | Drain |
|---|---|
| Prevents new pods | Evicts existing pods |
| Node becomes unschedulable | Pods moved away |

---

## Q5: Difference between Kubernetes upgrade and OS image patching?

| Kubernetes Upgrade | OS Image Upgrade |
|---|---|
| Kubernetes version changes | VM image patches |
| API/server changes | OS/kernel/security updates |

---

# 52. Final Interview Summary Answer

“In production we follow a staged AKS upgrade strategy. Before upgrading we validate deprecated APIs, check workload compatibility, verify PodDisruptionBudgets, and ensure enough cluster capacity for surge upgrades. We first upgrade the control plane followed by node pools sequentially. During node upgrades AKS creates surge nodes, cordons old nodes, drains workloads safely respecting PDBs, and reschedules pods using Kubernetes scheduler. We also use topology spread constraints, readiness probes, node affinity, and multiple replicas to ensure high availability during upgrades. Separately, we perform OS image patching to apply kernel and security updates without changing Kubernetes versions.”


---

# Conclusion

This upgrade strategy provides:

- High availability
- Safer rolling upgrades
- Minimal downtime
- Better workload resilience
- Enterprise-grade AKS operations
- Secure OS patching
- Controlled Kubernetes lifecycle management

These concepts are heavily used in real enterprise AKS environments and are very common in DevOps, SRE, platform engineering, and cloud architect interviews.
