---
layout: post
title: "Kubernetes Resource Optimization: Requests, Limits, and Autoscaling for DevOps Engineers"
date: 2026-11-07 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, resource-management, hpa, vpa, cluster-autoscaler, cost-optimization, devops]
description: "Practical guide to Kubernetes resource optimization including requests/limits, horizontal and vertical autoscaling, and cluster management for cost efficiency."
---

# Kubernetes Resource Optimization: Requests, Limits, and Autoscaling for DevOps Engineers

Kubernetes gives you great flexibility, but without thoughtful resource configuration, clusters quickly become over-provisioned or unstable. This article walks through the core concepts and practical patterns for optimizing CPU and memory usage in Kubernetes workloads.

## Why Resource Optimization Matters

Every pod in Kubernetes consumes resources. If you specify too little, pods get evicted or scheduled poorly. If you specify too much, you pay for unused capacity. The sweet spot between under- and over-provisioning is where efficient clusters live.

Common pain points include:

- **No requests defined**: The scheduler cannot make informed placement decisions, and containers may experience starvation.
- **No limits defined**: A runaway container can consume all available node capacity, affecting other workloads.
- **Static over-provisioning**: Teams add safety margins heuristically, leading to cluster bloat over time.
- **Missing autoscaling**: Deployments scale based on operator intuition rather than actual utilization.

This article covers the three pillars of resource optimization: **definitions**, **autoscaling**, and **cluster management**.

## 1. CPU and Memory Requests and Limits

Every container should define both requests and limits. They serve different purposes and have different impacts on scheduling and runtime behavior.

### Requests

- **Scheduling**: The scheduler uses requests to decide which node a pod lands on. A pod with `requests: cpu: "500m"` will only be scheduled on nodes with at least 500m of available CPU.
- **QoS tier**: Guarantees the pod a place on the node. If the node becomes overcommitted, pods with **Burstable** or **BestEffort** QoS may be evicted first.

### Limits

- **Enforcement**: The kernel enforces limits. If a container exceeds its memory limit, the OOM Killer may terminate it. If it exceeds its CPU limit, the kernel throttles the container (CPU ready time increases).
- **Stability**: Prevents a single workload from consuming all node resources.

### Example: Well-Defined Resources

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        cpu: "200m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
```

**Rule of thumb**: Set requests to the expected average usage, and limits to the maximum burst the application can tolerate before degrading.

### Common Pitfalls

| Anti-Pattern | Better Approach |
|---|---|
| `resources: {}` (nothing specified) | Always specify requests and limits |
| `limits: cpu: "1"` without requests | Set requests first, then limits above |
| `requests: equal: limits` | Leave headroom; requests should be sustainable usage, not absolute maximum |
| Different units for cpu/memory across containers | Standardize on millicores and mebibytes for consistency |

## 2. Horizontal Pod Autoscaling (HPA)

The Horizontal Pod Autoscaler adjusts the number of pod replicas based on observed metrics. It is the most common autoscaling mechanism for stateless workloads.

### Basic HPA

```bash
# Create an HPA that scales based on CPU utilization
kubectl autoscale deployment app-deployment \
  --min-replicas=2 \
  --max-replicas=10 \
  --cpu-percent=70
```

### Custom Metrics

HPA can also scale based on custom metrics from metrics-server or Prometheus Adapter:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### HPA Considerations

- **Scale-down delay**: HPA waits 5 minutes after metrics improve before reducing replicas. Adjust with `--horizontal-pod-autoscaler-downscale-delay`.
- **Stubborn scaling**: If metrics are consistently at the boundary, HPA may oscillate. Consider adding a buffer or using a longer evaluation window.
- **External metrics**: Use the `metrics.k8s.io` API (heapster/metrics-server) for resource-based scaling. Custom metrics require the metrics adapter.

## 3. Vertical Pod Autoscaling (VPA)

While HPA changes the *number* of pods, VPA changes the *resource allocation* of individual pods. It recommends resource adjustments based on historical usage.

### VPA Modes

| Mode | Behavior |
|---|---|
| **Off** | VPA collects data but makes no changes |
| **Auto** | VPA updates pod resources automatically (replaces the pod to apply changes) |
| **Initial** | VPA sets initial resources at creation time only; manual updates after that |
| **Disabled** | No VPA management |

### Example: VPA in Auto Mode

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
spec:
  target:
    apiVersion: apps/v1
    kind: Deployment
    name: app-deployment
  updatePolicy:
    updateMode: Auto
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: "100m"
        memory: "64Mi"
      maxAllowed:
        cpu: "2"
        memory: "2Gi"
```

### VPA Workflow

1. VPA observes resource usage over time.
2. It recommends new resource values.
3. In `Auto` mode, it terminates and recreates the pod with the new allocation.
4. In `Initial` mode, it only sets resources on first creation.

**Note**: VPA is complementary to HPA, not a replacement. Use HPA for scale-out/in and VPA for per-pod optimization.

## 4. Cluster Autoscaler

The Cluster Autoscaler adjusts the number of nodes in your cluster based on pod scheduling needs. It works alongside HPA and VPA.

### When It Triggers Scale-Out

- A pod fails to schedule because no node has enough available resources.
- A pod is pending due to insufficient resources (even if some nodes exist but are too full).

### When It Triggers Scale-In

- All pods on a node are terminated and rescheduled elsewhere.
- The node has been underutilized for a configurable period (default 10 minutes).

### Basic Configuration

The Cluster Autoscaler is typically deployed as a deployment on each node group or managed through cloud provider integration.

**AWS EKS**: `aws autoscaling create-auto-scaling-group` with EKS node group tags, or use `eksctl create nodegroup`.

**GKE**: Enabled by default; configure via `gcloud container clusters update` with `--enable-autoscaling` and `--max-nodes`/`--min-nodes`.

**AKS**: Use `az aks update` with `--enable-cluster-autoscaler` and `--node-count`.

### Node Group Sizing

The Cluster Autoscaler is only as good as your node group sizing. Recommendations:

- Use **node groups with homogeneous instance types** so the autoscaler can replace nodes predictably.
- Set **minimum node count** to at least 2-3 for high availability.
- Reserve **burst capacity** or use spot instances for fault-tolerant, scalable workloads.
- Avoid over-large nodes: large nodes increase eviction risk and make fine-grained scaling harder.

## 5. Resource Quotas and Limit Ranges

For multi-tenant clusters or platform teams managing many namespaces, resource quotas and limit ranges enforce guardrails at the namespace level.

### Resource Quota

Limits total resource consumption per namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "2"
    requests.memory: "4Gi"
    limits.cpu: "5"
    limits.memory: "10Gi"
    pods: "50"
```

### Limit Range

Defines default and boundary values for pods and containers within a namespace:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: team-a
spec:
  limits:
  - type: Pod
    min:
      cpu: "100m"
      memory: "64Mi"
    max:
      cpu: "2"
      memory: "1Gi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "300m"
      memory: "256Mi"
  - type: Container
    min:
      cpu: "100m"
      memory: "64Mi"
    max:
      cpu: "1"
      memory: "512Mi"
    default:
      cpu: "300m"
      memory: "256Mi"
    defaultRequest:
      cpu: "200m"
      memory: "128Mi"
```

**Combined effect**: Quotas set the total budget; limit ranges ensure individual pods stay within reasonable boundaries.

## 6. Practical Optimization Workflow

### Step 1: Observe Before You Optimize

Never guess at resource values. Use these tools first:

```bash
# Install metrics-server if not present
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# View current resource usage per node and pod
kubectl top nodes
kubectl top pods -A
```

### Step 2: Set Requests for All Workloads

Start by adding requests to pods that lack them. Use `kubectl edit` or a tool like `kustomize` to batch-add:

```yaml
resources:
  requests:
    cpu: "300m"
    memory: "256Mi"
```

### Step 3: Define Reasonable Limits

Add limits to prevent resource exhaustion:

```yaml
resources:
  limits:
    cpu: "750m"
    memory: "1Gi"
```

### Step 4: Deploy HPA for Key Deployments

```bash
kubectl autoscale deployment web-frontend --min=2 --max=10 --cpu-percent=65
```

### Step 5: Evaluate VPA for Stateful or Long-Running Workloads

For databases, caches, or services with stable traffic patterns, VPA can reduce over-allocation.

### Step 6: Enable Cluster Autoscaler

Deploy and configure the Cluster Autoscaler for your cloud provider. Monitor node group fill rates and adjust min/max sizes.

### Step 7: Enforce Quotas and Limit Ranges

Apply resource quotas and limit ranges to namespaces to prevent runaway consumption and maintain predictable boundaries.

## Conclusion

Kubernetes resource optimization is not a one-time task—it is an iterative practice. Start with these steps:

1. **Add requests and limits** to every pod that is missing them.
2. **Deploy HPA** for workloads with variable traffic.
3. **Assess VPA** for long-running services with stable patterns.
4. **Configure Cluster Autoscaler** to match node capacity with pending pod demand.
5. **Enforce quotas and limit ranges** at the namespace level.

Monitor the results with `kubectl top` and adjust based on actual utilization patterns. Over time, these practices reduce cloud costs, improve cluster stability, and make incident response more predictable.

*Always test resource changes in a staging namespace first. Aggressive limits can cause unnecessary OOM terminations; overly permissive limits can lead to resource starvation. Align your settings with your application's SLOs and on-call escalation practices.*

*This article draws on common Kubernetes resource management patterns and production deployments. Adjust resource values, autoscaling thresholds, and quota settings to match your cluster's node types, workload profiles, and organizational policies.*