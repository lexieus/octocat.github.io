---
layout: post
title: "Kubernetes Resource Quotas and Limit Ranges"
date: 2026-10-09T00:00:00+0530
author: Chandana Nawarathna
categories: [kubernetes, devops, infrastructure]
tags: [kubernetes, resource-quotas, limit-ranges, compute-resources, cluster-administration]
description: "A practical guide to setting resource quotas and limit ranges in Kubernetes to prevent namespace exhaustion and ensure fair resource allocation across teams."
---

# Kubernetes Resource Quotas and Limit Ranges

When multiple teams or workloads share a Kubernetes cluster, unchecked resource consumption can quickly lead to namespace exhaustion, causing critical pods to fail and disrupting services. Kubernetes provides two complementary mechanisms to prevent this: **ResourceQuotas** and **LimitRanges**.

This guide walks through both concepts with practical examples you can apply to your clusters.

## ResourceQuotas

A `ResourceQuota` defines total resource constraints for a namespace. It limits the aggregate amount of compute resources (CPU and memory) that can be created across all objects in that namespace.

### Example: CPU and Memory Quota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: development
spec:
  hard:
    cpu: "4"
    memory: "8Gi"
    pods: "10"
    persistentvolumeclaims: "5"
    requests.cpu: "2"
    limits.cpu: "4"
    requests.memory: "4Gi"
    limits.memory: "8Gi"
```

In this example:

- `cpu` and `memory` set absolute maxima for the namespace
- `pods` limits the total number of pods
- `persistentvolumeclaims` limits PVC creation
- `requests.cpu` / `limits.cpu` separate resource requests (what the scheduler sees) from limits (the maximum a container can use)
- Same pattern applies to memory

### Viewing Quota Usage

```bash
# Check quota status for a namespace
kubectl get resourcequota compute-quota -n development

# Output example:
NAME           USED   CPU   MEMORY   AGE
compute-quota   150m   2Gi   3h
```

### Common Quota Patterns

| Pattern | Use Case |
|---------|----------|
| Team isolation | Each team gets dedicated CPU/memory budgets |
| Environment separation | Dev, staging, and prod have distinct limits |
| Burst protection | Prevent spike-based resource exhaustion |

## LimitRanges

While **ResourceQuotas** cap total cluster usage, **LimitRanges** define per-pod and per-container resource constraints. They ensure every container requests and sets appropriate limits, even if no quota is explicitly specified.

### Example: Default and Min/Max Limits

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: development
spec:
  limits:
  - type: Pod
    max:
      cpu: "2"
      memory: "512Mi"
    min:
      cpu: "100m"
      memory: "64Mi"
  - type: Container
    default:
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:
      cpu: "300m"
      memory: "128Mi"
    max:
      cpu: "1"
      memory: "512Mi"
    min:
      cpu: "100m"
      memory: "64Mi"
    resources:
      requests:
        cpu: "300m"
        memory: "128Mi"
      limits:
        cpu: "1"
        memory: "512Mi"
```

This LimitRange:

- Sets **per-pod** maxima: no pod can exceed 2 CPU or 512 MiB memory
- Sets **per-pod** minima: every pod must reserve at least 100 mCPU and 64 MiB memory
- Sets **per-container** defaults: new containers without explicit specs get 500 mCPU / 256 MiB
- Sets **per-container** minimum requests and limits: containers must request at least 100 mCPU / 64 MiB and cap at 1 CPU / 512 MiB

### Enforcing Limits

When you create a pod in a namespace with this LimitRange:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: nginx:latest
    resources:
      requests:
        cpu: "300m"
        memory: "256Mi"
      limits:
        cpu: "700m"
        memory: "512Mi"
```

Kubernetes will validate against the LimitRange. If you omit `resources`, the default values (500 mCPU / 256 MiB) are automatically injected. If you specify only requests, limits are derived or clamped per the range definitions.

## Practical Deployment Pattern

Here's a complete workflow for a development namespace:

1. **Create a LimitRange** first — ensures all workloads have sane defaults:

```bash
kubectl apply -f limitrange.yaml
```

2. **Create a ResourceQuota** to cap total namespace usage:

```bash
kubectl apply -f resourcequota.yaml
```

3. **Deploy workloads** knowing they'll be validated against both:

```bash
kubectl apply -f deployment.yaml
```

### Verification

Check that your pod respects the limits:

```bash
# Describe the pod to see allocated resources
kubectl describe pod my-app -n development

# Check quota consumption
kubectl get resourcequota compute-quota -n development -o json
```

### Best Practices

| Practice | Recommendation |
|----------|----------------|
| **Start with defaults** | Use LimitRanges even before enabling quotas |
| **Right-size requests** | Set requests based on actual workload baseline, not peak |
| **Separate environments** | Use distinct namespaces with independent quotas |
| **Review regularly** | Adjust quotas as team patterns evolve |
| **Document limits** | Include quota info in onboarding docs for new teams |

## Conclusion

ResourceQuotas and LimitRanges are essential Kubernetes primitives for operating multi-tenant clusters responsibly. ResourceQuotas prevent namespace-level resource bankruptcy, while LimitRanges ensure every pod has meaningful resource boundaries. Together they form the foundation of predictable, reliable cluster operations.

Start with a LimitRange across all your namespaces, then add ResourceQuotas tailored to team or environment needs. The upfront investment pays off by preventing the all-too-common "node pressure" incidents that disrupt services.

---

*This article draws on common Kubernetes administration patterns. For cluster-specific guidance, consult your Kubernetes distribution's official documentation.*
