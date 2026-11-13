---
layout: post
title: "CI/CD Pipeline with GitHub Actions: Automated Kubernetes Deployment Patterns"
date: 2026-11-13 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [github-actions, kubernetes, ci-cd, devops, docker]
description: "A practical guide to building a GitHub Actions CI/CD pipeline for Kubernetes, covering image build, manifest promotion, progressive deployment patterns, and best practices."
---
# CI/CD Pipeline with GitHub Actions: Automated Kubernetes Deployment Patterns

By default, teams start with manual `kubectl apply` flows, but as clusters grow and deployment frequency increases, this becomes a bottleneck and a risk. A proper GitHub Actions CI/CD pipeline enforces pull-request-based changes, runs validation automatically, and delivers updates to Kubernetes with confidence. This article walks through a practical, production-ready GitHub Actions workflow for Kubernetes deployments, covering build, test, and deploy stages with zero-downtime rollout patterns.

## Why GitHub Actions + Kubernetes?

Common pain points with manual deployments include:

- **No audit trail** — it is unclear who pushed which change and when.
- **Configuration drift** — `kubectl apply` run ad-hoc diverges from the desired state in Git.
- **Secrets management** — credentials hardcoded in scripts or shell history.
- **No gatekeeping** — changes merge directly to production without validation.

A GitHub Actions pipeline solves all of these by making the pipeline the single source of truth for every deployment.

## Prerequisites

- A Kubernetes cluster with `kubectl` configured
- A GitHub repository with the application source code
- OIDC federation between GitHub and your cloud provider (or a Kubernetes service account with appropriate permissions)
- The `k8s-deploy` GitHub Action or `actions/setup-kubectl`

## Workflow Overview

The pipeline has four stages:

1. **Lint and build** — validate manifests, compile code, run unit tests
2. **Push image** — build the container image and push to a registry
3. **Apply manifests** — update image tags in Kubernetes resources
4. **Verify and promote** — run smoke tests, wait for health checks

### Full Pipeline (.github/workflows/kubernetes-deploy.yaml)

```yaml
name: Kubernetes CI/CD

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  IMAGE_REGISTRY: docker.io/youruser
  IMAGE_NAME: my-app
  KUBE_NAMESPACE: production
  KUBE_CONTEXT: production-cluster

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push Docker image
        id: push-image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.IMAGE_REGISTRY }}/${{ env.IMAGE_NAME }}:latest

      - name: Run unit tests
        run: |
          make test
          # or: go test ./...
          # or: npm test

  deploy:
    needs: build-and-test
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v4
        with:
          version: "1.30.0"
          # Authenticate via OIDC; requires OIDC federation with your IDP
          # Alternatively, use kubeconfig secret:
          # kubeconfig: ${{ secrets.KUBE_CONFIG }}

      - name: Configure kubecontext
        run: |
          kubectl config use-context ${{ env.KUBE_CONTEXT }}

      - name: Set image tag in manifests
        run: |
          # Replace the image tag in all YAML manifests under k8s/
          find k8s -name "*.yaml" -exec sed -i "s|${IMAGE_NAME}:[a-z0-9]*|${IMAGE_NAME}:${{ github.sha }}|g" {} +

      - name: Apply manifests
        run: |
          kubectl apply -k k8s/  # kustomize overlay, or: kubectl apply -f k8s/

      - name: Wait for rollout
        run: |
          kubectl rollout status deployment/${{ env.IMAGE_NAME }} -n ${{ env.KUBE_NAMESPACE }} --timeout=300s

      - name: Smoke test (curl-based)
        run: |
          # Example: wait for the service to be ready
          SERVICE_IP=$(kubectl get svc/${{ env.IMAGE_NAME }} -n ${{ env.KUBE_NAMESPACE }} -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
          for i in $(seq 1 30); do
            if curl -sf "http://${SERVICE_IP}:8080/health" > /dev/null 2>&1; then
              echo "Health check passed"
              exit 0
            fi
            sleep 5
          done
          echo "Health check failed after 150s"
          exit 1
```

## Progressive Deployment Patterns

Once the basic pipeline is running, you can integrate progressive delivery strategies to reduce risk further.

### Canary Deployments with Argo Rollouts

If you are already using ArgoCD (see the [GitOps with ArgoCD article](/2026/11/09/gitops-argocd-progressive-deploy)), you can extend the pipeline to trigger canary steps:

```bash
# From the pipeline, after the new image is pushed:
kubectl argo rollouts step set-weight my-app-rollout --weight=20
# ... wait for analysis ...
kubectl argo rollouts step set-weight my-app-rollout --weight=100
```

### Blue-Green Deployments

The pipeline can also switch a Service selector to shift traffic instantly:

```yaml
# In the pipeline, after the new deployment is ready:
kubectl patch deployment my-app-green -n production -p '{"spec":{"replicas":3}}'
kubectl patch service my-app -n production -p '{"spec":{"selector":{"version":"green"}}}'
```

If issues appear, switch the selector back:

```bash
kubectl patch service my-app -n production -p '{"spec":{"selector":{"version":"blue"}}}'
```

## Best Practices

| Practice | Reason |
|---|---|
| **Pin image tags** | Use `${{ github.sha }}` or a semver tag never `latest` in production |
| **Lint manifests before apply** | `kubeval` or `kubectl convert` catches API mismatches early |
| **Store secrets in cloud secret managers** | Inject via Kubernetes `secret` or GitHub Encrypted Secrets, never hardcode |
| **Run plan before apply** | `terraform plan` style preview, or `kubeval validate` |
| **Enforce branch protection** | Require the CI pipeline to pass before merging to `main` |
| **Automate rollback** | If the rollout fails or smoke tests fail, the pipeline should `kubectl rollout undo` |

## Conclusion

A GitHub Actions CI/CD pipeline for Kubernetes transforms deployments from ad-hoc `kubectl` commands into a disciplined, auditable process. By containerizing the application, automating validation, and integrating with progressive delivery tools like Argo Rollouts, teams can ship faster with confidence. Start with the basic build-apply-verify workflow above, then iterate canary or blue-green strategies as your comfort level grows.

*Always test new pipeline configurations in a staging namespace first. The example commands are safe for observation and apply; avoid destructive operations (such as deleting namespaces or forcing rollbacks) without understanding the impact on in-flight workloads. Align your pipeline with your organization's SLOs, on-call runbooks, and change-management procedures.*

*This article is intended as a practical guide. Adjust the workflow to match your container registry, Kubernetes distribution, and operational practices. Image references, secret names, and namespace values should be customized for your environment.*