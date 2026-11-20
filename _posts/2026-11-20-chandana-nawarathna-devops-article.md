---
layout: post
title: "Progressive Delivery with Kubernetes: Canary and Blue-Green Deployments"
date: 2026-11-20 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, deployment, ci-cd, argo-rollouts, progressive-delivery, devops]
description: "Practical guide to implementing canary and blue-green deployments in Kubernetes using Argo Rollouts and native Kubernetes strategies, with safe rollback patterns."
---

# Progressive Delivery with Kubernetes: Canary and Blue-Green Deployments

Modern Kubernetes deployments need more than `kubectl set image`. Rolling updates can mask failures, and all-at-once updates risk downtime. Progressive delivery strategies — canary and blue-green — let you release changes gradually, validate them against real traffic, and roll back instantly if something goes wrong.

This article walks through both strategies with practical manifests, Argo Rollouts integration, and rollback patterns you can apply today.

## Why Progressive Delivery Matters

A traditional rolling update increases pods one at a time while decrementing the old ones. If the new version has a latency spike or error rate increase, you might not detect it until enough old pods are gone. Progressive delivery flips this:

- **Canary**: Keep a small percentage of traffic on the new version, monitor metrics, then promote or abort.
- **Blue-Green**: Swap an entire service endpoint to the new version atomically, with immediate rollback if needed.

Both approaches reduce risk and give you a safety net during releases.

## Canary Deployments with Argo Rollouts

Argo Rollouts is the de facto standard for progressive Kubernetes deployments. It introduces a `Rollout` resource that extends `Deployment` with advanced routing and analysis capabilities.

### Basic Canary Manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: app-rollout
spec:
  replicas: 5
  strategy:
    canary:
      steps:
        - setWeight: 20
        - setWeight: 40
        - setWeight: 60
        - setWeight: 100
      pause:
        seconds: 30
        message: "Waiting for canary stability before next weight increase"
  template:
    spec:
      containers:
        - name: app
          image: myregistry/app:latest
          ports:
            - containerPort: 8080
```

### Canary with Analysis Template

You can integrate Prometheus-based analysis so Argo Rollouts automatically checks success criteria before promoting weight:

```yaml
spec:
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause:
            seconds: 60
            pausePolicy: fixed
        - setWeight: 40
        - pause:
            seconds: 60
            pausePolicy: fixed
      analysis:
        templates:
          - name: canary-analysis
            metric:
              - matcher: source
                source:
                  name: prometheus
                  kind: Service
                  apiVersion: v1
                  spec:
                    selector:
                      matchLabels:
                        release: canary
                interval: "30s"
                threshold:
                  type: percent
                  metric: http_requests_total
                  value: 0.05
                  operator: less
```

**Key steps**: Each `setWeight` step increases traffic to the new version. Between steps, the pause either waits a fixed duration or continues automatically. If you add an `analysis` block, Argo will check the Prometheus metric before each step — if the error rate exceeds 5%, the rollout halts and can be auto-reverted.

### Promoting and Rolling Back

```bash
# Promote to 100% immediately
argo rollouts promote app-rollout

# Check current state
argo rollouts get rollout app-rollout

# Auto-revert on analysis failure (if configured)
# Argo Rollouts will automatically roll back to the previous stable version

# Manual rollback
argo rollouts abort rollout app-rollout --delete
```

## Blue-Green Deployments with Kubernetes Services

Blue-green deployments are ideal when you want to switch entire traffic flow at once, such as before a major version jump or when the new version has breaking changes.

### Manifest Pattern

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service
  labels:
    app: app
spec:
  selector:
    app: app
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: app-green
  labels:
    app: app
    tier: green
spec:
  selector:
    app: app
    version: green
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: app-blue
  labels:
    app: app
    version: blue
spec:
  selector:
    app: app
    version: blue
  ports:
    - port: 80
      targetPort: 8080
```

### Traffic Swap

- **Green** and **blue** `Deployment`s (or `ReplicaSet`s) run in parallel.
- The primary `app-service` selector targets the current version.
- To switch: update the `app-service` selector labels or use a DNS-level switch.

```bash
# Switch service to blue version
kubectl patch service app-service -p '{"spec":{"selector":{"app":"app","version":"blue"}}}'

# Switch back to green
kubectl patch service app-service -p '{"spec":{"selector":{"app":"app","version":"green"}}}'
```

### Rollback

Since both versions run simultaneously, rolling back is simply switching the selector back. If the blue version has a critical issue, immediate switch-back to green restores traffic with zero additional deployment time.

## Safe Rollback Patterns

Regardless of strategy, these practices ensure you can revert quickly:

1. **Keep old ReplicaSets/pods running** — Don't set `revisionHistoryLimit: 0` on your Deployment if you want instant rollback.
2. **Preserve database migrations** — Feature flags or separate migration steps; don't tie rollback to schema changes.
3. **Monitor during promotion** — Use `kubectl get pods`, Prometheus alerts, or Argo Rollouts analysis to detect issues before full promotion.
4. **Automate rollback criteria** — In Argo Rollouts, configure the analysis template to auto-abort on failure thresholds.

```bash
# Native Kubernetes rollback
kubectl rollout undo deployment/app

# Argo Rollouts abort
argo rollouts abort rollout app-rollout

# Switch blue-green back via selector patch
kubectl patch service app-service -p '{"spec":{"selector":{"app":"app","version":"green"}}}'
```

## Conclusion

Progressive delivery is not just a nice-to-have — it's a risk mitigation strategy that every Kubernetes team should have in their toolkit. Canary deployments via Argo Rollouts give you granular weight increases with optional automated analysis. Blue-green deployments offer instant traffic swaps and immediate rollback. Combine either with monitoring, feature flags, and preserved old replicas, and you have a release pipeline that protects both your users and your confidence.

*Always test progressive delivery patterns in a staging namespace first. The manifests above use safe, read-only or non-destructive flags; avoid `kubectl delete` or `kubectl drain` unless you intend to remove or cordon the resource.*

*This article draws on common Kubernetes progressive delivery patterns and Argo Rollouts documentation. Adjust weight steps, analysis thresholds, and service selector strategies to match your cluster topology and SLOs.*