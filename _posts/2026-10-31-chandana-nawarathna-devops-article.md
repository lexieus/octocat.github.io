---
layout: post
title: "Progressive Delivery with ArgoCD: Canary and Blue-Green Deployments"
date: 2026-10-31 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [argocd, kubernetes, progressive-delivery, canary, blue-green, devops, ci-cd]
description: "Practical guide to implementing progressive deployment strategies using ArgoCD and Argo Rollouts, including canary and blue-green patterns with automated verification and rollback."
---

# Progressive Delivery with ArgoCD: Canary and Blue-Green Deployments

Traditional Kubernetes deployments often replace all pods at once, risking downtime and making it hard to isolate regressions. Progressive delivery patterns — canary and blue-green — let you roll out changes incrementally, verify behavior in production, and automatically roll back if something goes wrong. ArgoCD + Argo Rollouts provide the foundation for these patterns declaratively.

## Why Progressive Delivery Matters

A deployment that shifts 100% of traffic instantly can cause a widespread outage if a bug slips through. Progressive delivery limits the exposed surface area, collects real metrics from a subset of users, and gives you a safety net before full rollout. The two most common patterns are:

- **Canary**: New version runs alongside the stable version with a fraction of traffic. If metrics are healthy, traffic is gradually shifted.
- **Blue-Green**: Two parallel deployments (blue and green). Traffic switches en masse from one to the other, often after a health check confirms the target is ready.

Both patterns rely on three pillars: **traffic splitting**, **automated health checks**, and **automated rollback**.

## 1. Canary Deployments with Argo Rollouts

Argo Rollouts is a Kubernetes controller that replaces the standard `Deployment` resource and provides built-in canary support. The core idea is to create a new ReplicaSet with the updated pods while keeping the old ReplicaSet running, then shift traffic via a `Service`.

### Minimal Canary Manifest

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
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: myapp:v2
        ports:
        - containerPort: 8080
```

### How It Works

1. **Initial state**: 5 replicas of `myapp:v1` running, Rollout created with canary strategy.
2. **Step 1**: `setWeight: 20` — 1 new replica (`myapp:v2`) is brought up, 20% of Service traffic is routed to it via a `canary` sub-set.
3. **Step 2**: `setWeight: 40` — a second `myapp:v2` replica is added, traffic shifts to 40%.
4. **Remaining steps**: Traffic is incrementally increased until 100%, at which point the old ReplicaSet is terminated.

Each step pauses until the Rollout's health check passes (based on `progressDeadlineSeconds` and pod readiness). If a step times out or a pod fails, the Rollout can automatically roll back to the previous version.

### Adding Automation with Analysis

You can attach an `AnalysisTemplate` to validate metrics between steps, such as checking error rates or latency from a Prometheus datasource:

```yaml
spec:
  strategy:
    canary:
      steps:
      - setWeight: 20
        analysis:
          templates:
          - name: check-error-rate
            args:
              - name: error-rate-threshold
                value: "0.05"
            args:
            - name: window-period
                value: "5m"
```

## 2. Blue-Green Deployments

Blue-green deployments are useful when you want a faster full-traffic switch and have the capacity to run both versions simultaneously. The pattern uses two Services — one for the "blue" (current) version and one for the "green" (new) version — and an Ingress or LoadBalancer that switches its backend selection.

### Blueprint Manifest

```yaml
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
    - port: 8080
      targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: app-green
  labels:
    app: app
    version: green
spec:
  selector:
    app: app
    version: green
  ports:
    - port: 8080
      targetPort: 8080
---
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: app-rollout
spec:
  replicas: 10
  strategy:
    blueGreen:
      automatic: true
      directPodRequests: false
      preHealthCheck: true
      blueService: app-blue
      greenService: app-green
      autoPromoteTime: 300
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
        version: green
    spec:
      containers:
      - name: app
        image: myapp:v2
        ports:
        - containerPort: 8080
```

### Key Fields

- `automatic: true` — After the green pods pass the pre-health check, ArgoCD waits `autoPromoteTime` seconds before automatically promoting (switching the blue Service selector to green).
- `preHealthCheck: true` — A readiness gate is checked before the promotion window starts.
- `directPodRequests: false` — Traffic is routed through the Services, not directly to pods.

### Manual Promotion Workflow

If you prefer full control, set `automatic: false`. After the green deployment is healthy, an operator or GitOps push can annotate the Rollout to trigger promotion:

```bash
kubectl patch rollout/app-rollout -p '{"spec":{"strategy":{"blueGreen":{"automatic":true}}}}' --type=merge
```

Or via the ArgoCD UI, click **Promote** to shift traffic from blue to green.

## 3. Automated Rollback Patterns

Both canary and blue-green Rollouts support automatic rollback when analysis templates fail or health checks expire. The mechanism is consistent:

- If a `setWeight` step exceeds `progressDeadlineSeconds` without passing analysis, the Rollout is marked failed.
- Argo Rollouts records the event and reverts the ReplicaSet weight to the previous value.
- The old version continues running, minimizing the mean time to recovery (MTTR).

You can inspect rollout history and retry behavior via:

```bash
kubectl rollout history rollout/app-rollout
kubectl rollout status rollout/app-rollout --watch
```

## 4. GitOps Workflow with ArgoCD

In a GitOps model, the desired state of the Rollout is defined in a Git repository and synced by ArgoCD. A typical workflow:

1. **Feature branch**: Developer creates a branch, updates the Rollout manifest with the new image tag, and opens a PR.
2. **ArgoCD sync**: ArgoCD detects the change, renders the Rollout, and applies it to the cluster in a preview namespace (if using ArgoCD application sets with diff/experimental features).
3. **Promotion**: After code review and CI validation, the PR is merged. ArgoCD syncs the production application, triggering the canary or blue-green Rollout.
4. **Observability**: Engineers monitor the Rollout progress via `kubectl`, the ArgoCD UI, or Prometheus dashboards. If something goes wrong, a rollback can be initiated from the UI or via `kubectl rollout undo`.

### Example: Updating the Image in Git

```yaml
# rolls/app-rollout.yaml
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:v2.1  # bump version
```

Pushing this change to the `main` branch triggers ArgoCD, which rolls out the new version using the previously defined strategy.

## Conclusion

Progressive delivery transforms deployments from a risky "big bang" into a controlled, observable process. Canary deployments with Argo Rollouts provide fine-grained traffic shifting and automated analysis, while blue-green deployments offer a simpler full-switch pattern when capacity permits. Both integrate naturally into GitOps workflows with ArgoCD, letting you promote confidently and roll back instantly when needed.

Start with a canary strategy for a critical service, define an analysis template that checks your primary SLO (e.g., error rate < 1%), and iterate toward a full progressive delivery practice that matches your organization's risk tolerance and release cadence.

*This article is intended as a practical guide. Always test progressive strategies in a staging environment first and verify that your analysis templates and rollback conditions match your application's behavior under load. Adjust step weights, deadline settings, and promotion windows to match your SLOs and on-call practices.*