---
layout: post
title: "Kubernetes Resource Requests and Limits: A Practical Guide for DevOps Engineers"
date: 2026-10-13 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, resources, cpu, memory, devops, scheduling, hpa]
description: "How to properly set CPU and memory requests and limits in Kubernetes, avoid common pitfalls, and integrate with the Horizontal Pod Autoscaler for reliable application scaling."
---

## Overview

Kubernetes gives you fine-grained control over compute resources through **resource requests** and **limits**. Getting these right is one of the highest-impact optimizations a DevOps engineer can make: they affect pod scheduling, quality-of-service (QoS) classes, and how the Horizontal Pod Autoscaler (HPA) behaves. This article walks through the practical aspects of configuring requests and limits, with concrete examples and anti-patterns to avoid.

## Resource Requests vs Limits

| Concept | What it does | Effect |
|---------|--------------|--------|
| **requests** | The minimum amount of CPU/memory guaranteed to the container. | Used by the scheduler to decide where the pod fits. Also determines the node's share of CPU (if `limitRanger` is active). |
| **limits** | The maximum amount of CPU/memory the container is allowed to consume. | Enforced by the kernel; exceeding limits can cause the container to be terminated. |

A pod with no limits is classified as **Burstable**; a pod with both requests and limits equal is **Guaranteed**.

## Example: Well-Configured Deployment

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
        image: your-registry/my-app:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "200m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

- **Requests** of `200m` CPU and `128Mi` memory ensure the scheduler places the pod on a node with at least that much free capacity.
- **Limits** of `500m` CPU and `512Mi` memory prevent the container from consuming all node resources if it misbehaves.
- The ratio between requests and limits determines the QoS class, which in turn affects eviction order under pressure.

## Quality-of-Service Classes

| Requests | Limits | QoS Class | Eviction Priority |
|----------|--------|-----------|-------------------|
| Not set | Not set | Best-effort | Highest (evicted first) |
| Set | Not set | Burstable | Medium |
| Set | Equal to requests | Guaranteed | Lowest (evicted last) |

In practice, aim for **Guaranteed** when you need predictable performance (databases, latency-sensitive services), and **Burstable** for typical web workloads where occasional spikes are acceptable.

## Interacting with the Horizontal Pod Autoscaler

The HPA uses **CPU utilization** as its primary metric. If requests are set too low, the HPA will see high utilization and scale out aggressively — potentially over-provisioning. If requests are set too high, the HPA may never trigger because utilization stays low.

### Example: HPA based on custom CPU utilization

```bash
kubectl autoscale deployment my-app \
  --min-replicas 2 \
  --max-replicas 10 \
  --cpu-percent=70
```

The `--cpu-percent=70` target is calculated against the **requests** value. If a container requests `200m` CPU and is using `300m`, the HPA sees 150% utilization and may scale up. Adjust requests to align the target utilization with your scaling goals.

## Common Pitfalls

1. **Setting requests too high** — pods fail to schedule; HPA never scales; wasted node capacity.
2. **Setting limits too low** — containers get killed by the OOM Killer or CPU throttling during legitimate traffic spikes.
3. **Requests and limits both set to `"0"` or omitted** — leads to Best-effort QoS; unpredictable behavior under resource pressure.
4. **Forgetting to set both requests and limits** — if you set only limits without requests, the scheduler has no information for placement, and the pod gets Burstable QoS.
5. **Using identical requests/limits across all environments** — development, staging, and production typically have different traffic patterns and should have appropriately scaled values.

## namespace-Wide Safety with LimitRanges

Most clusters have a `LimitRange` resource in each namespace that enforces reasonable defaults. You can view it with:

```bash
kubectl describe limitrange <name> -n <namespace>
```

A typical LimitRange might enforce minimum/maximum values and set default requests/limits if they are omitted from a manifest. This helps catch missing resource declarations across the team.

## Verification Checklist

After deploying a new resource manifest, run:

```bash
# 1. Check the pod is scheduled on a node with enough free capacity
kubectl get pod my-app-XXXXX -o wide

# 2. Verify the QoS class
kubectl get pod my-app-XXXXX -o jsonpath='{.status.qosClass}'

# 3. Confirm the HPA is observing the expected metrics
kubectl get hpa my-app -o json

# 4. Simulate resource pressure and observe behavior
kubectl cgroupdebug -g memory # (requires privileged access)
```

## Conclusion

Resource requests and limits are the foundation of reliable Kubernetes deployments. They enable the scheduler to make informed placement decisions, define the QoS class that determines eviction priority, and provide the reference points the HPA uses for scaling decisions. Start by setting requests based on typical usage and limits with a safety margin of 1.5–2x. Iterate as you collect real metrics from your monitoring system, and enforce compliance with namespace LimitRanges across your teams.

*Further reading: Kubernetes Quality-of-Service Classes, Horizontal Pod Autoscaler guide, CPU/ memory concepts for Kubernetes, LimitRange documentation.*
