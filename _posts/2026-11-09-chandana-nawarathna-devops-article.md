---
layout: post
title: "GitOps with ArgoCD: Progressive Deployment Strategies for Kubernetes"
date: 2026-11-09 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, gitops, argocd, ci-cd, progressive-deployment, canary, blue-green]
description: "A practical guide to implementing progressive deliveries — canary and blue-green — using ArgoCD in a Kubernetes GitOps workflow."
---

# GitOps with ArgoCD: Progressive Deployment Strategies for Kubernetes

In a Kubernetes GitOps workflow, every `git push` drives the desired state. But shipping code directly to production carries risk. Progressive deployments — canary and blue-green — let you validate new versions against a small subset of traffic before full rollout, reducing blast radius and enabling rapid rollback.

This article walks through implementing both patterns using ArgoCD, without modifying core application code.

## Why Progressive Deployments Matter

| Approach | Traffic Shift | Rollback Speed | Best For |
|---|---|---|---|
| **Canary** | % of pods gradually | Minutes via `kubectl rollout` | Services with stateful workloads, gradual validation |
| **Blue-Green** | 100% switch at once | Seconds via service selector swap | Stateless workloads, fast validation, zero-downtime releases |

ArgoCD supports both natively through its Application CRD and the Argo Rollouts controller.

## Prerequisites

- Kubernetes cluster with ArgoCD installed
- `argocd` CLI configured
- Application manifest in Git under `apps/` directory

## 1. Canary Deployments with Argo Rollouts

ArgoCD integrates with Argo Rollouts to manage canary analysis and promotion. The workflow is:

1. Deploy a `Rollout` manifest under the ArgoCD Application
2. ArgoCD syncs the Rollout, creating a Canary replica set alongside the Stable one
3. Traffic is split via a Service selector or Ingress
4. After analysis, either promote to 100% or rollback

### Minimal Rollout manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 3
  strategy:
    canary:
      steps:
      - setWeight: 20
        pause: {duration: 5m}
      - setWeight: 50
        pause: {duration: 5m}
      - setWeight: 100
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: myregistry/my-app:{{ imageTag }}
        ports:
        - containerPort: 8080
```

Apply this manifest and ArgoCD will track it. The Rollout controller creates a new ReplicaSet for each step, shifting traffic gradually.

### Canary analysis with external tools

Argo Rollouts can integrate with Prometheus, Grafana, or Datadog for automated analysis. Add an `analysis` section:

```yaml
strategy:
  canary:
    steps:
    - setWeight: 20
      pause: {duration: 5m}
    analysis:
      templates:
      - name: canary-passed
        args:
        - name: threshold
          value: "95"
        kind: template
        apiVersion: argoproj.io/v1alpha1
        text: |
          {{- $failures := divide (int .Metrics.Failures) (int .Metrics.Requests) 100 -}}
          {{ if gt $failures 10 }}FAIL{{ else }}PASS{{ end }}
      args:
      - name: requests
        source:
          metric:
            name: http_requests_total
            kind: Counter
      - name: failures
        source:
          metric:
            name: http_requests_total
            kind: Counter
            metrics:
            - name: http_requests_total
              query: http_requests_total{status=~"5.."}
```

If the failure rate stays below 10% across the analysis window, ArgoCD auto-promotes. Otherwise, it halts and rolls back.

## 2. Blue-Green Deployments via Service Arithmetic

Blue-green is simpler: two identical Deployments (`my-app-blue`, `my-app-green`) and a single Service that switches its `selector` at the moment of cutover.

### Manifest setup

```yaml
# Blue Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: blue
  template:
    metadata:
      labels:
        app: my-app
        version: blue
    spec:
      containers:
      - name: app
        image: myregistry/my-app:blue

# Green Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-green
spec:
  replicas: 0  # Start dormant
  selector:
    matchLabels:
      app: my-app
      version: green
  template:
    metadata:
      labels:
        app: my-app
        version: green
    spec:
      containers:
      - name: app
        image: myregistry/my-app:green

# Shared Service — selector switched at cutover
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
    version: blue  # initially points to blue
  ports:
  - port: 80  targetPort: 8080
```

### Cutover workflow

1. Deploy both to the cluster via ArgoCD
2. Set `my-app-green.replicas: 3` to spin up the new version
3. Update the Service selector to `version: green` — all traffic instantaneously shifts
4. If issues arise, switch back to `version: blue` immediately

This pattern requires no pod restarts and provides true zero-downtime when the new version is stateless.

## 3. ArgoCD Application Configuration

Instead of managing raw manifests, define an ArgoCD `Application` that references the Rollout or blue-green setup. Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/infra.git
    path: manifests/apps/my-app
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

Place the Rollout or Deployment+Service manifests under `manifests/apps/my-app/` in your Git repo. ArgoCD will sync them on every push, and you can trigger rollouts from the ArgoCD UI or CLI:

```bash
argocd app set my-app --parameter imageTag=v2.3.1
argocd app sync my-app
# Or trigger a canary step:
kubectl argo rollouts step set-weight my-app-rollout --weight=20
```

## 4. Practical Checklist

- [ ] Add `argocd-rollouts` if using canary deployments
- [ ] Define health checks and readiness/liveness probes in your Pod spec
- [] Configure `argocd` app auto-sync with `prune: true` for safe updates
- [ ] Test rollback scenarios in a staging namespace first
- [ ] Document the cutover procedure for blue-green deployments
- [ ] Use ArgoCD's `app set --parameter` for promotion from CI pipelines

## Conclusion

Progressive deployments are a safety net for Kubernetes releases. Canary deployments via Argo Rollouts give you granular traffic control and automated analysis; blue-green deployments offer instant cutover for stateless services. Either way, GitOps ensures the desired state is always codified in Git, and ArgoCD handles the safe sync from there.

Always test new deployment patterns in a dedicated namespace or staging environment before promoting to production. The best approach depends on your workload type, tolerance for latency during cutover, and whether you need automated analysis vs. manual sign-off.

*This article is intended as a practical guide. Adjust thresholds, weights, and patterns to match your SLOs, cluster topology, and on-call rotation. Always validate rollback paths before enforcing production policies.*