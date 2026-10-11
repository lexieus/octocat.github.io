---
layout: post
title: "GitOps with ArgoCD: Progressive Delivery and Rollback Strategies"
date: 2026-10-11 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, gitops, argocd, devops, ci-cd, progressive-delivery, rollbacks]
description: "A practical guide to implementing progressive delivery with ArgoCD, including canary releases, automated rollbacks, and health-based promotion strategies."
---

## Introduction

GitOps has become the de facto standard for managing Kubernetes manifests. ArgoCD extends this by providing a dedicated continuous delivery tool for Kubernetes. One of its most powerful features is progressive delivery — the ability to shift traffic incrementally between application versions while monitoring health metrics.

This article walks through configuring progressive delivery with ArgoCD, using canary releases and automated rollback strategies that protect production workloads.

## Prerequisites

- A Kubernetes cluster with ArgoCD installed
- Application manifests in a Git repository
- `argocd` CLI configured to communicate with your ArgoCD server

## Setup: Application Sync and Health Checks

Before enabling progressive delivery, ensure your ArgoCD application is configured to sync manifests and report health status correctly.

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1beta1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/your-org/your-repo.git
    targetRevision: HEAD
    path: ./k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
  health:
    type: Preference
    state: Healthy
    threshold: 50
```

The `threshold: 50` setting means ArgoCD will consider the application healthy only when at least 50% of its pods are ready. This baseline is essential for progressive delivery to function.

## Progressive Delivery with ArgoCD Plugins

ArgoCD itself does not execute traffic-shifting logic — it delegates this to plugins such as `argocd-rollouts` or `external-secrets`. The most common pattern uses [Argo Rollouts](https://argo-workflows.github.io/argo-rollouts/) to manage the underlying canary setups.

### Installing Argo Rollouts

```bash
kubectl create namespace rollouts
kubectl apply -n rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

### Configuring a Canary Deployment

Define your Deployment with Argo Rollouts annotations, and ArgoCD will track the rollout status automatically.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
      annotations:
        # Argo Rollouts will observe and manage this Deployment
        argo.rollouts/enabled: "true"
        argo.rollouts/autoPromotionEnabled: "false"
    spec:
      containers:
      - name: app
        image: your-registry/my-app:{{image_tag}}
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

Setting `argo.rollouts/autoPromotionEnabled: "false"` disables automatic promotion, giving you manual control via ArgoCD sync status.

## Manual Promotion Workflow

With auto-promotion disabled, the workflow proceeds as follows:

1. **Canary analysis** — Argo Rollouts creates a canary replica and shifts a percentage of traffic (e.g., 10%) to the new version.
2. **Health validation** — Monitor metrics: error rate, latency, pod readiness.
3. **Manual promotion** — If health checks pass, trigger ArgoCD sync to promote the canary to the stable version.
4. **Automated rollback** — If any metric breaches the defined threshold, Argo Rollouts rolls back automatically, and ArgoCD reflects the rollout in its UI.

### Example: Promoting a Canary via ArgoCD

```bash
# Check the current rollout status
kubectl argo rollouts status deployment/my-app

# Promote the canary to stable
kubectl argo rollouts promote deployment/my-app
```

ArgoCD detects the promotion event and updates the application status accordingly. The UI will show the rollout as "Synced" once the new version is fully rolled out.

## Automated Rollback Strategies

Argo Rollouts can be configured to roll back when specific conditions are met, without manual intervention.

```yaml
annotations:
  argo.rollouts/linear:
    # Incrementally shift traffic one pod at a time
    # pause: '5m'  # Optional: pause at a specific step
  argo.rollouts/maxUnavailable: "1"
  argo.rollouts/maxSurge: "1"
```

If a readiness probe fails during a linear shift, Argo Rollouts rolls back to the previous version automatically. ArgoCD sync status will reflect the rollback, and the Git commit remains unchanged — the repository continues to point at the desired state.

## Visualizing Rollouts in the ArgoCD UI

Once the application is syncing, navigate to the ArgoCD UI. The **SYNC STATUS** section shows the current rollout state. Clicking the application reveals a timeline of canary sets, promotions, and any rollbacks that occurred. This visibility is invaluable for post-deployment analysis.

## Best Practices

| Practice | Recommendation |
|----------|----------------|
| **Health probes** | Define both `readinessProbe` and `livenessProbe` with appropriate timeouts |
| **Canary percentage** | Start small (5–10%), increase gradually if no issues are observed |
| **Promotion gate** | Use manual promotion until the team is comfortable with automated flows |
| **Monitoring integration** | Wire Argo Rollouts metrics into Prometheus/Grafana for alerting |
| **Rollback testing** | Practice rollbacks in a staging environment before enabling in production |

## Conclusion

Progressive delivery with ArgoCD and Argo Rollouts combines the safety of automated rollbacks with the confidence of incremental traffic shifts. By configuring health thresholds, disabling auto-promotion initially, and leveraging the ArgoCD UI for visibility, DevOps teams can reduce production incidents while accelerating deployment frequency.

Start with manual promotion workflows, add monitoring integrations, and iteratively enable automation as your team's comfort grows.