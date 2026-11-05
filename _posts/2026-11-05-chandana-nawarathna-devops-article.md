---
layout: post
title: "GitOps with ArgoCD: A Practical Guide for DevOps Engineers"
date: 2026-11-05 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [argocd, gitops, kubernetes, ci-cd, devops]
description: "A hands-on guide to setting up GitOps workflows using ArgoCD, including app deployment, sync policies, and progressive delivery patterns."
---

# GitOps with ArgoCD: A Practical Guide for DevOps Engineers

GitOps is the de facto standard for managing Kubernetes clusters declaratively. By letting Git be the single source of truth for both application and infrastructure state, you gain auditability, automated reconciliation, and a clear separation between Git-driven changes and ad-hoc `kubectl` mutations. ArgoCD is the most widely adopted GitOps tool for Kubernetes, and this article walks through getting it running and using it effectively.

## Why GitOps Matters

In traditional Kubernetes workflows, engineers apply manifests with `kubectl apply -f` directly against a cluster. Over time this creates **configuration drift**: the live state diverges from the intended state stored in version control. Merges are missed, emergency patches land outside PR processes, and it becomes unclear why a pod is running a particular image version.

GitOps solves this by:

- **Declarative desired state** — defined entirely in Git YAML
- **Automated reconciliation** — the controller continuously compares live state against Git and self-heals
- **Audit trail** — every change is a commit with a reviewer, timestamp, and merge commit
- **Progressive delivery** — canary and blue-green deployments are encoded as Git changes, not imperative scripts

## ArgoCD Core Concepts

| Concept | Description |
|---|---|
| **Application** | The top-level resource that describes the desired state: source repo, path, sync policy, and destination cluster |
| **Source** | The repository (GitHub, GitLab, Bitbucket, or internal Git server) containing manifests |
| **Destination** | The Kubernetes cluster and namespace where the application is deployed |
| **Sync Policy** | Controls how and when ArgoCD reconciles the app (automated, manual, interval, prune, automated options) |
| **Health State** | Computed from Kubernetes resources (Available, Degraded, Progressing) and custom health checks |

## Installing ArgoCD

The quickest way to get ArgoCD running is via the ArgoCD Helm chart. Assuming you already have a Kubernetes cluster with `argocd` namespace available:

```bash
# Create the namespace if it doesn't exist
kubectl create namespace argocd

# Install the ArgoCD Helm chart
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd --namespace argocd
```

After installation, retrieve the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-password -o jsonpath="{.data.password}" | base64 -d
```

Access the UI by port-forwarding or using the `argocd` CLI:

```bash
# Port-forward to localhost:8080
kubectl -n argocd port-forward svc/argocd-server 8080:443

# Or login via CLI (once authenticated)
argocd login localhost:8080
argocd account update-password --account admin
```

## Defining Your First Application via the CLI

Rather than using the UI for the initial setup, you can define an Application manifest declaratively. This is the pattern you'll replicate in Git.

```bash
argocd app create my-web-app \
  --repo https://github.com/your-org/manifests.git \
  --path ./k8s/prod \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production \
  --sync-policy automated \
  --auto-create-namespace
```

- `--repo` points to the Git repository containing your K8s manifests
- `--path` is the folder relative to the repo root (supports glob patterns)
- `--dest-namespace` is the target namespace in the cluster
- `--sync-policy automated` makes ArgoCD auto-sync on `argocd app sync` or when the source changes
- `--auto-create-namespace` lets the App create the namespace if it doesn't exist

Check the app status:

```bash
argocd app get my-web-app -o yaml
argocd app status my-web-app
```

## Sync Policies in Detail

The `sync-policy` defines the reconciliation behavior. Here are the most common patterns:

### Automated Sync (hands-free)

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

- `prune: true` removes resources in the cluster that are no longer defined in Git
- `selfHeal: true` automatically reconciles when drift is detected (no manual `argocd app sync` needed)

### Manual Sync with Auto-Sync on Patch

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
    syncOptions:
    - CreateNamespace=true
```

- No automatic sync on every Git change; you run `argocd app sync my-web-app` manually
- Useful for environments where you want a review step before applying

### Interval-Based Sync

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
    syncOptions:
    - Replace=false
    - IgnoreDifferences=ResourceGroup.cattle.io/v1/Management
    cronSpec: "0 * * * *"  # runs every hour
```

- Use when you want to batch reconciliations or avoid rapid sync loops during active deployments.

## Progressive Delivery with ArgoCD

ArgoCD integrates with tools like Kustomize, Helm, and Terraform for progressive delivery patterns. A common pattern is **Canary** deployment using Argo Rollouts encoded in the App spec.

Example: Deploy a new version to 10% of traffic, then promote if healthy.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-service-canary
spec:
  source:
    repoURL: https://github.com/your-org/manifests.git
    path: ./k8s/prod
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
  parameters:
    - name: canaryWeight
      value: "20"
    - name: canaryStepByStep
      value: "true"
```

The above parameters can be consumed by your Helm charts or Kustomize overlays to adjust `replicaCount`, `strategy.type`, or `autoscaler` settings.

## Health and Progress Tracking

ArgoCD computes health based on resource conditions. You can view the health tree:

```bash
argocd app get my-web-app -o yaml
```

Key fields:

- **Healthy** — all K8s resources report `Ready=True`
- **Degraded** — some resources are in a non-Ready state
- **Progressing** — indicates the app is still rolling out (e.g., a Deployment with a pending `UpdatedReplicas`)

You can also add **health checks** that query custom endpoints:

```yaml
spec:
  healthCheck:
    type: TCP
    port: 8080
```

This makes ArgoCD mark the app as unhealthy if the container is running but the app inside is not serving.

## Common Pitfalls and Gotchas

| Issue | Symptom | Fix |
|---|---|---|
| **App stuck in `Synced` but `OutOfSync`** | Git has new commits, but ArgoCD hasn't reconciled | Run `argocd app sync <name>` or verify the `syncPolicy` `automated` settings |
| **Resource `prune` removes a workload you kept** | You scaled a Deployment manually, ArgoCD prunes it back | Use `argocd app set <name> --sync-options Replace=false` or avoid `prune: true` on mutable workloads |
| **Source repo credentials not working** | ArgoCD cannot clone the repo | Add SSH keys via `argocd repo add` or configure a `git` secret in the `argocd` namespace |
| **Health state never becomes `Healthy`** | A probe or selector is misconfigured | Check the `healthCheck` configuration or inspect resource conditions with `kubectl get pods -o yaml` |

## Conclusion

GitOps with ArgoCD replaces ad-hoc `kubectl` workflows with a repeatable, auditable process where Git is the single source of truth. Start by migrating one non-critical application: commit its manifests, add the ArgoCD Application resource, and let the controller handle reconciliation. From there, you can adopt automated sync policies, progressive delivery configurations, and health-check integrations to manage your entire fleet.

The fastest win is enabling `selfHeal: true` and `prune: true` on a test namespace app — you'll immediately see ArgoCD restore the desired state after accidental `kubectl` changes, and you'll gain confidence to delegate more workloads to the GitOps flow.

*This article is based on ArgoCD's official documentation and common GitOps patterns. Adjust repository paths, cluster endpoints, and parameter names to match your environment. If your organization uses a Git platform with SSO (GitHub Enterprise, GitLab SaaS, etc.), configure the OAuth callback in ArgoCD to enable single-sign-on for the UI.*