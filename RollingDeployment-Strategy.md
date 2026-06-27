# README-09-RollingDeployment-Strategy.md

# Kubernetes Rolling Deployment Strategy

---

# 1. Overview

Rolling deployment is the most commonly used deployment strategy in Kubernetes and AKS.

It provides:

* Zero downtime deployments
* Controlled pod replacement
* Automatic rollback support
* Application availability during upgrades

Most enterprise applications use:

```text id="s8g4c2"
RollingUpdate
```

---

# 2. Why Rolling Deployment?

Without Rolling Update:

Current:

```text id="2w3u8a"
Pod1
Pod2
Pod3
Pod4
```

Deploy New Version:

```text id="w0m1af"
Delete All Pods

↓

Create New Pods
```

Problem:

```text id="75ly7o"
Downtime
```

---

With Rolling Update:

```text id="jtkvct"
Create New Pod

↓

Wait Until Healthy

↓

Delete Old Pod

↓

Repeat
```

No downtime.

---

# 3. Rolling Deployment Flow

Current:

```text id="jqj3kk"
v1

Pod1
Pod2
Pod3
Pod4
```

Deploy:

```text id="r3sq65"
v2
```

Process:

```text id="s5wsl0"
Create Pod5(v2)

↓

Ready?

↓

Delete Pod1(v1)

↓

Create Pod6(v2)

↓

Ready?

↓

Delete Pod2(v1)
```

Continues until all pods upgraded.

---

# 4. Deployment Strategy Location

Strategy is defined inside:

```text id="h5p3j0"
deployment.yaml
```

or Helm template:

```text id="z7czbh"
templates/deployment.yaml
```

Not inside Azure DevOps YAML.

Azure DevOps simply executes:

```bash id="u2m8rw"
helm upgrade
```

Kubernetes performs the rollout.

---

# 5. Deployment Manifest

```yaml id="1u8x5s"
apiVersion: apps/v1

kind: Deployment

metadata:

  name: banking-app

spec:

  replicas: 4

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxSurge: 25%

      maxUnavailable: 25%
```

---

# 6. maxSurge

Determines:

Extra pods allowed during upgrade.

Example:

Current:

```text id="hjlwm3"
4 Pods
```

maxSurge:

```yaml id="f13vxj"
maxSurge: 1
```

During deployment:

```text id="o8w9c1"
4 Existing

+

1 New

=

5 Pods
```

---

# 7. maxUnavailable

Determines:

Maximum unavailable pods.

Example:

```yaml id="4fnd8z"
maxUnavailable: 1
```

Means:

At least:

```text id="3yibp6"
3 Pods Available
```

during deployment.

---

# 8. Enterprise Recommendation

Production:

```yaml id="9fry7h"
strategy:

  type: RollingUpdate

  rollingUpdate:

    maxSurge: 25%

    maxUnavailable: 0
```

Reason:

Never reduce availability.

---

# 9. Readiness Probe

Determines:

Can pod receive traffic?

Example:

```yaml id="22lhf9"
readinessProbe:

  httpGet:

    path: /actuator/health

    port: 8080

  initialDelaySeconds: 30

  periodSeconds: 10
```

Traffic only reaches healthy pods.

---

# 10. Liveness Probe

Determines:

Is container alive?

Example:

```yaml id="h9g51j"
livenessProbe:

  httpGet:

    path: /actuator/health

    port: 8080

  initialDelaySeconds: 60
```

Failed container automatically restarted.

---

# 11. Startup Probe

Useful for Spring Boot applications.

Example:

```yaml id="ojt21n"
startupProbe:

  httpGet:

    path: /actuator/health

    port: 8080

  failureThreshold: 30
```

Prevents premature restarts.

---

# 12. Helm Template Example

values.yaml

```yaml id="8b9jv1"
strategy:

  maxSurge: 25%

  maxUnavailable: 0
```

deployment.yaml

```yaml id="9x6k2v"
strategy:

  type: RollingUpdate

  rollingUpdate:

    maxSurge: {{ .Values.strategy.maxSurge }}

    maxUnavailable: {{ .Values.strategy.maxUnavailable }}
```

---

# 13. What Happens If New Pod Fails?

Current:

```text id="f31w2s"
v1 Running
```

New Pod:

```text id="x9r7uv"
CrashLoopBackOff
```

Readiness probe fails.

Kubernetes stops rollout.

Existing pods continue serving traffic.

---

# 14. Check Rollout Status

```bash id="kmq7t5"
kubectl rollout status deployment banking-app
```

Output:

```text id="6z2xjv"
successfully rolled out
```

---

# 15. Rollout History

```bash id="lj4cn6"
kubectl rollout history deployment banking-app
```

Shows:

```text id="p5m8tb"
Revision 1

Revision 2

Revision 3
```

---

# 16. Rollback

```bash id="8b0g9x"
kubectl rollout undo deployment banking-app
```

Restores previous ReplicaSet.

---

Specific revision:

```bash id="7njcft"
kubectl rollout undo deployment banking-app --to-revision=2
```

---

# 17. Common Deployment Strategies

### Rolling Update

Most common.

Zero downtime.

---

### Recreate

Delete old pods first.

Causes downtime.

Rarely used.

---

### Blue-Green

Two environments:

```text id="2cnhmy"
Blue

Green
```

Switch traffic.

---

### Canary

Small percentage traffic.

Example:

```text id="ew9h7s"
10%

↓

25%

↓

50%

↓

100%
```

---

# 18. Common Failures

## Progress Deadline Exceeded

Cause:

Pod never became ready.

Check:

```bash id="c7bsmv"
kubectl describe pod
```

---

## CrashLoopBackOff

Check:

```bash id="bdzzj6"
kubectl logs pod-name
```

---

## ImagePullBackOff

Cause:

Wrong image tag.

Missing AcrPull role.

---

## Readiness Probe Failure

Application endpoint unhealthy.

---

# 19. Azure DevOps Integration

CD Pipeline:

```yaml id="v3fz6u"
- task: HelmDeploy@0

  inputs:

    command: upgrade
```

Helm executes:

```bash id="7k2wtr"
helm upgrade
```

Kubernetes Deployment object handles:

```yaml id="hmf35a"
strategy:

  type: RollingUpdate
```

Azure DevOps does not control rollout behavior.

Kubernetes does.

---

# 20. Improvements Implemented

✔ Rolling updates

✔ Readiness probes

✔ Startup probes

✔ Zero downtime deployments

✔ Controlled pod replacement

✔ Automatic rollback support

---

# 21. Interview Questions

### Where is deployment strategy defined?

Answer:

Inside Kubernetes Deployment manifest or Helm templates, not inside Azure DevOps pipeline YAML.

---

### Difference between maxSurge and maxUnavailable?

Answer:

maxSurge controls additional pods created during upgrade, while maxUnavailable controls how many existing pods can be unavailable.

---

### Why readiness probe?

Answer:

To ensure traffic reaches only healthy pods.

---

### Why startup probe?

Answer:

To avoid killing slow-starting applications.

---

### How did you rollback deployments?

Answer:

Using Helm rollback or kubectl rollout undo to restore previous revisions.

---

### Which strategy did you use?

Answer:

RollingUpdate with maxSurge 25% and maxUnavailable 0 to ensure zero downtime.

---

# Real Enterprise Flow

```text id="9wq4gk"
CD Pipeline

↓

Helm Upgrade

↓

Kubernetes Deployment

↓

Rolling Update

↓

Readiness Probe

↓

Healthy Pods

↓

Traffic Switch

↓

Zero Downtime
```
