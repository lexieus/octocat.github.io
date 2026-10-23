---
layout: post
title: "Progressive Delivery with Argo Rollouts: Canary Deployments and Automatic Rollback"
date: 2026-10-24 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, argo-rollouts, ci-cd, canary, progressive-delivery, devops]
description: "Practical guide to implementing canary deployments using Argo Rollouts with automatic traffic shifting, health checks, and automated rollback strategies."
---

# Progressive Delivery with Argo Rollouts: Canary Deployments and Automatic Rollback

Traditional Kubernetes deployments replace pods atomically — if the new version has a bug, all traffic shifts instantly and users experience the failure. **Progressive delivery** with Argo Rollouts solves this by shifting traffic incrementally, validating each step before proceeding, and automatically rolling back on failure.

This article walks through implementing canary deployments using Argo Rollouts, with concrete manifests and rollback strategies.

## Why Progressive Delivery Matters

In a standard `kubectl apply` or Deployment, a new ReplicaSet becomes active immediately when ready. If a critical bug is deployed, every user is affected instantly. Progressive delivery reduces risk by:

- **Incremental traffic shifting** — route 10%, then 30%, then 100% to the new version
- **Automated health checks** — verify metrics (HTTP status, Prometheus queries) before each shift
- **Automated rollback** — revert to the previous version if health checks fail
- **A/B testing capability** — run multiple versions in production with targeted user segments

## Prerequisites

Argo Rollouts must be installed in your cluster. If you are using a managed platform that includes it (GKE EKS with add-ons, AKS), it may already be available.

```bash
# Check if Argo Rollouts is installed
kubectl get rollouts.argoproj.io

# Install if needed (GKE example)
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/argo-rollouts.yaml
```

## Basic Canary Rollout Configuration

The following Deployment and Rollout manifest implements a canary release with traffic shifting from 0% to 100% over five steps.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: myregistry.io/my-app:1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "200m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
---
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app-rollout
spec:
  replicaCount: 3
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 3m}
      - setWeight: 30
      - pause: {duration: 3m}
      - setWeight: 50
      - pause: {duration: 3m}
      - setWeight: 100
```

### How the canary strategy works

| Step | Action | Traffic after step |
|------|--------|-------------------|
| Initial | Deploy v2 alongside v1 | 0% v2, 100% v1 |
| 1 | Set weight to 10% | 10% v2, 90% v1 |
| 2 | Pause 3 minutes | Wait for manual or automated validation |
| 3 | Set weight to 30% | 30% v2, 70% v1 |
| 4 | Pause 3 minutes | Wait for validation |
| 5 | Set weight to 50% | 50% v2, 50% v1 |
| 6 | Pause 3 minutes | Wait for validation |
| 7 | Set weight to 100% | 100% v2, 0% v1 (full rollout) |

Each `pause` step gives you a window to verify the new version before traffic continues shifting. If you detect issues, you can interrupt the rollout and roll back.

## Health Checks and Automated Validation

The real power of Argo Rollouts comes from automated health checks. You can configure the rollout to wait for Prometheus metrics, HTTP responses, or other conditions before proceeding to the next step.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app-rollout
spec:
  replicaCount: 3
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause:
          duration: 2m
          setWeight: 30
          # Automated analysis using a hypothesis
          analysis:
            templates:
            - name: canary-analysis
              args:
              - name: canary-hypothesis
                value: "canary-success"
              - name: baseline-hypothesis
                value: "baseline-stable"
              steps:
              - setWeight: 30
              - wait:
                  duration: 1m
              - abort: "canary is unhealthy"
              - setWeight: 50
              - wait:
                  duration: 1m
              - setWeight: 100
```

### Using successfulRolloutsHistoryLimit and failedRolloutsHistoryLimit

```yaml
spec:
  rolloutStatusSettings:
    successfulRolloutsHistoryLimit: 5
    failedRolloutsHistoryLimit: 5
```

Argo Rollouts keeps a history of successful and failed rollouts. You can view the rollout history to understand what happened during a deployment.

```bash
# View rollout status and history
kubectl argo rollouts get rollout my-app-rollout

# View detailed rollout metrics
kubectl argo rollouts get rollout my-app-rollout -o yaml
```

## Automatic Rollback on Failure

Argo Rollouts can automatically roll back to the previous version if a step fails or a health check doesn't pass. There are two ways to trigger rollback:

### 1. Explicit abort/roll back

```yaml
strategy:
  canary:
    steps:
    - setWeight: 10
    - pause:
        # If health checks fail, abort and roll back
        abort: "response code >= 400 for canary pods"
        setWeight: 30
    - setWeight: 100
```

### 2. Automatic rollback via analysis templates

Argo Rollouts can automatically abort a rollout if the analysis template returns a failure status. This is the most common pattern.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app-rollout
spec:
  replicaCount: 3
  strategy:
    canary:
      steps:
      - setWeight: 10
      - analysis:
          # Run a Prometheus query; fail if error rate exceeds threshold
          templates:
          - name: canary-health
            args:
            - name: error-rate-threshold
              value: "0.05"  # 5% error rate max
            - name: query
              value: |
              sum(rate(http_requests_total{job="my-app",version="canary"}[5m])) /
              sum(rate(http_requests_total[5m]))
            - name: metric
              value: "error_rate"
            tolerance:
              - inequality: "LessThan"
                value: "{{steps.parameters.[0].value}}"
          - name: baseline-health
            args:
            - name: error-rate-threshold
              value: "0.02"
            - name: query
              value: |
              sum(rate(http_requests_total{job="my-app",version="stable"}[5m])) /
              sum(rate(http_requests_total[5m]))
            tolerance:
              - inequality: "LessThan"
                value: "{{steps.parameters.[0].value}}"
      - setWeight: 100
```

If the `canary-health` analysis template reports that the error rate exceeds 5%, the rollout automatically aborts and rolls back to the previous version.

## Canary Analysis Using Prometheus

You can integrate Argo Rollouts with Prometheus to validate the canary version before each traffic shift. The following example checks that the 5-minute average error rate for the canary is below 5% and that the success rate is above 99%.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app-rollout
spec:
  replicaCount: 3
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 2m}
      - analysis:
          templates:
          - name: canary-error-rate
            args:
            - name: max-error-rate
              value: "0.05"
            - name: prometheus-query
              value: |
                sum(rate(http_requests_total{job="my-app",version="canary"}[5m])) by (code_class)
                  / sum(rate(http_requests_total{job="my-app"}[5m])) by (code_class)
            - name: metric
              value: "error_rate"
            tolerance:
              - inequality: "LessThan"
                value: "{{steps.parameters.[0].value}}"
          - name: canary-success-rate
            args:
            - name: min-success-rate
              value: "0.99"
            - name: prometheus-query
              value: |
                sum(rate(http_requests_total{job="my-app",version="canary",code_class!~"5.."}[5m]))
                  / sum(rate(http_requests_total{job="my-app",version="canary"}[5m]))
            tolerance:
              - inequality: "GreaterThanOrEqual"
                value: "{{steps.parameters.[0].value}}"
      - setWeight: 30
      - pause: {duration: 2m}
      - analysis:
          templates:
          - name: canary-latency-p95
            args:
            - name: max-p95-latency
              value: "0.5"  # 500ms max
            - name: prometheus-query
              value: |
                histogram_quantile(0.95,
                  sum(rate(http_request_duration_seconds_bucket{job="my-app",version="canary"}[5m])) by (le))
            - name: metric
              value: "p95_latency"
            tolerance:
              - inequality: "LessThan"
                value: "{{steps.parameters.[0].value}}"
      - setWeight: 100
```

## Rollback Strategies

### Manual Rollback

If you detect issues during a canary deployment, you can pause the rollout and roll back manually:

```bash
# Pause the rollout at its current state
kubectl argo rollouts pause rollout/my-app-rollout

# Roll back to the previous version
kubectl argo rollouts undo rollout/my-app-rollout
```

### Automatic Rollback

As shown in the analysis templates above, Argo Rollouts can automatically detect failures and roll back without manual intervention. The rollback is itself a rollout that creates a new ReplicaSet with the previous version.

### Rolling Back via GitOps

If you are using GitOps (Argo CD), you can annotate the Rollout manifest to control rollout behavior:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/rollout.undoversion: "1"
```

This tells Argo CD to roll back to version 1 during the next sync.

## Canary Rollout Verification

After applying a rollout, verify the deployment status:

```bash
# Check current rollout status
kubectl argo rollouts get rollout my-app-rollout

# Expected output includes:
# - Current step
# - Traffic weight distribution
# - Number of healthy/unhealthy pods

# View rollout history
kubectl argo rollouts get history rollout/my-app-rollout

# View metrics for the current rollout
kubectl argo rollouts get metrics rollout/my-app-rollout
```

## Best Practices

1. **Start small** — begin with 10% traffic and short pause durations (1–2 minutes) for early validation
2. **Define clear health metrics** — error rate, success rate, latency percentiles; avoid relying on subjective "feelings"
3. **Set tolerance thresholds** — use analysis templates with explicit inequality checks rather than arbitrary pauses
4. **Keep a rollback window** — maintain `successfulRolloutsHistoryLimit` and `failedRolloutsHistoryLimit` to retain rollout history
5. **Integrate with GitOps** — declare rollout configurations in Git and let Argo CD manage the apply cycle; use `argocd app sync` to trigger rollouts
6. **Use namespace isolation** — deploy canary rollouts into dedicated namespaces for team-level experimentation before cluster-wide rollouts
7. **Complement with Cluster Autoscaler** — if canary usage spikes resource demand, the cluster autoscaler can add nodes automatically

## Conclusion

Progressive delivery with Argo Rollouts is the most effective way to reduce deployment risk in Kubernetes clusters. By shifting traffic incrementally, validating each step with automated health checks, and rolling back automatically on failure, you move from "deploy and hope" to "deploy with confidence." Start with simple weight-based canary steps and gradually incorporate Prometheus-based analysis templates as your team's maturity grows.

*This article is intended for operational reference. Always test canary strategies in a staging environment before applying to production workloads, and adjust pause durations and thresholds to match your application's traffic patterns and SLAs.*

---
*Further reading: Argo Rollouts documentation, CanaRY deployment patterns, Prometheus query examples for canary analysis, GitOps with Argo CD and Rollouts.*
