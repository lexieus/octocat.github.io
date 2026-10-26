---
layout: post
title: "Implementing GitOps with Argo CD: A Practical Guide for Kubernetes Deployment Workflows"
date: 2026-10-26 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, gitops, argocd, ci-cd, infrastructure-as-code, devops]
description: "A practical guide to setting up GitOps workflows using Argo CD, including application setup, sync policies, automated rollouts, and rollback strategies for production Kubernetes clusters."
---

# Implementing GitOps with Argo CD: A Practical Guide for Kubernetes Deployment Workflows

GitOps has become the de facto standard for managing Kubernetes workloads declaratively. By keeping the desired state of your infrastructure in a Git repository and using a dedicated controller to reconcile the cluster state, you gain version control, auditability, and automated rollouts. This article walks through a complete, production-ready GitOps setup using Argo CD.

## Why GitOps?

In traditional Kubernetes workflows, engineers apply manifests with `kubectl apply -f .` or scripted CI jobs. This approach works for small teams but creates friction at scale:

- **Drift**: Manual `kubectl` changes create a gap between the Git source of truth and the actual cluster state.
- **Audit gaps**: It is difficult to trace who changed what and when without a pull request–driven workflow.
- **Rollback friction**: Reversing a bad deployment often requires manual inspection and re-application of previous manifests.

GitOps solves these by making the Git repository the single source of truth. A control loop continuously compares the cluster state against the desired state in Git and auto-heals any drift.

## Prerequisites

- A running Kubernetes cluster (this guide uses a local `kind` cluster or a managed service).
- `kubectl` configured to talk to the cluster.
- Argo CD installed in the `argocd` namespace.

Install Argo CD with the Helm chart:

```bash
helm repo add argo-cd https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo-cd/argo-cd --namespace argocd --create-namespace
```

After installation, retrieve the initial password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Log in with the Argo CD UI port-forward:

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

## 1. Adding an Application to Argo CD

An **Application** in Argo CD represents a Kubernetes workload or set of workloads tracked in Git. The minimal application specification includes the source repository, the target cluster, and the path to the manifests.

Create a `demo-app` application via the Argo CD UI or by applying the following custom resource:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/chandanadev/example-app.git
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Key fields:

- `source.repoURL`: The Git repository containing your Kubernetes manifests.
- `source.path`: Relative to the repo root, the directory holding manifests (e.g., `k8s/`).
- `destination.server`: The Kubernetes API server endpoint.
- `destination.namespace`: The target namespace for deployment.
- `syncPolicy.automated.prune`: Ensures resources deleted from Git are also removed from the cluster.
- `syncPolicy.automated.selfHeal`: Automatically applies updates from Git to the cluster.

## 2. Directory Layout in Git

A typical Git repository for Argo CD follows this structure:

```
.
└── manifests/
    ├── namespace.yaml
    ├── deployment.yaml
    └── service.yaml
```

**namespace.yaml**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app-dev
```

**deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
  namespace: app-dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: example-app
  template:
    metadata:
      labels:
        app: example-app
    spec:
      containers:
        - name: app
          image: docker.io/chandanadev/example-app:latest
          ports:
            - containerPort: 8080
```

**service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-app
  namespace: app-dev
spec:
  selector:
    app: example-app
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
```

Commit these files to a branch (e.g., `main` or `dev`) and Argo CD will detect the new application immediately.

## 3. Sync Policies: Automated vs Manual

Argo CD offers three sync policy modes:

| Mode | Behaviour |
|------|-----------|
| **Automated** | The control loop continuously syncs the application to the cluster. `prune` and `selfHeal` are respected. |
| **Manual** | Changes must be triggered explicitly via the UI, CLI, or API. |
| **Semantic** | Sync only when the `targetRevision` changes (e.g., a new tag or branch). |

For production workloads, many teams use a **hybrid approach**: automated sync for infrastructure-level manifests (namespaces, CRDs) and manual sync for application deployments that require a review step before reaching the cluster.

Example: A semantic sync policy for a microservice that only updates on new container images:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
    allowEmpty: false
  syncOptions:
    - Validate=false
    - SkipDryRun=false
```

## 4. Rolling Updates and Canary Deployments

Argo CD inherits the update strategy from the Kubernetes Deployment. To control how pods are replaced, use the `rollingUpdate` field:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  replicas: 3
  ...
```

- `maxSurge`: Pods can be created above the desired replica count (default 25%).
- `maxUnavailable`: Unavailable pods during the update (default 25%).

For canary deployments, Argo CD supports integration with Argo Rollouts, which provides advanced routing and analysis:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: example-app-canary
spec:
  source:
    repoURL: https://github.com/chandanadev/example-app.git
    targetRevision: HEAD
    path: k8s/canary
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

The `k8s/canary` directory would contain Rollout and Service resources that route a percentage of traffic to the new version.

## 5. Rollback Strategies

When a deployment introduces a regression, Argo CD makes rollback straightforward. Because the Git repository is the source of truth, reverting to a previous commit and allowing the sync loop to reconcile restores the previous state.

**UI rollback**: In the Argo CD interface, navigate to the application, click the **ROLLBACK** button, and select the desired commit or revision.

**CLI rollback**:

```bash
argocd app set demo-app \
  --revision <COMMIT_HASH> \
  --sync-option prune=true
```

Or, edit the `targetRevision` in the Application manifest and let the sync loop apply the change.

**Important**: If `prune: true` is set in the sync policy, Argo CD will also remove resources that were added in the forward revision but do not exist in the reverted revision.

## 6. Connecting to Your CI Pipeline

GitOps does not eliminate the CI pipeline; it changes its role. Instead of deploying directly, CI jobs now focus on building and pushing container images, running tests, and publishing Helm charts or Kubernetes manifests. The actual deployment is handled by Argo CD.

A typical workflow:

1. Developer pushes code → CI trigger (GitHub Actions, GitLab CI, etc.).
2. CI builds the Docker image and pushes to a registry.
3. CI updates the `image:` tag in the manifest (or pushes a new Helm chart).
4. The Git repository change is detected by Argo CD.
5. The sync loop applies the new image tag to the cluster.

Example: A GitHub Actions step that updates the image tag:

```yaml
- name: Update image tag
  run: |
    git checkout main
    sed -i "s|image: .*|image: docker.io/chandanadev/example-app:${{ github.sha }}|g" manifests/deployment.yaml
    git commit -am "chore: bump image tag to ${{ github.sha }}"
    git push
```

The committed change is picked up by Argo CD, which syncs the updated image to the cluster.

## 7. Security and Access Control

Argo CD provides granular RBAC to control who can view, create, or modify applications.

- **Role-based access**: Create custom roles with specific permissions (e.g., `apps.view`, `apps.create`, `apps.sync`).
- **SSO integration**: Connect Argo CD to an identity provider (Keycloak, Azure AD, Google Workforce) using OIDC.
- **Cluster segmentation**: Use multiple Argo CD instances or logical divisions (projects) to isolate environments.

Example: A minimal role definition that allows viewing and sync-ing applications:

```yaml
apiVersion: v1
kind: Role
metadata:
  name: argocd-viewer
  namespace: argocd
rules:
- apiGroups: ["argoproj.io"]
  resources: ["applications"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["argoproj.io"]
  resources: ["applications"]
  resourceNames: ["*"]
  verbs: ["patch"]
```

## Conclusion

GitOps with Argo CD shifts the deployment workflow from "push to cluster" to "push to Git, reconcile automatically." The benefits are immediate: a single source of truth, audit‑ready change history, and automated rollbacks. By combining Argo CD's sync policies, rolling update strategies, and CI integration, DevOps teams can achieve reliable, repeatable Kubernetes deployments at scale.

Start with a single application, experiment with sync policies, and iterate toward a fully automated, production‑ready GitOps pipeline.

*This article draws on common Argo CD patterns and production GitOps experiences. Adjust repository structure, image tags, and RBAC policies to match your organization's security and operational requirements.*