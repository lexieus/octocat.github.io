---
layout: post
title: "Kubernetes Cluster Troubleshooting: A Systematic Debugging Approach for DevOps Engineers"
date: 2026-10-30 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, debugging, troubleshooting, devops, observability]
description: "A systematic approach to diagnosing and resolving common Kubernetes cluster issues, from node connectivity to application deployment failures."
---

# Kubernetes Cluster Troubleshooting: A Systematic Debugging Approach for DevOps Engineers

When a Kubernetes cluster misbehaves, the instinct is often to start applying fixes — restarting pods, rolling back deployments, or rebooting nodes. Without a structured method, this can waste time and introduce new issues. This article presents a systematic debugging framework that helps DevOps engineers methodically diagnose and resolve cluster problems, from node-level issues to application-level failures.

## Why a Structured Method Matters

Kubernetes issues rarely exist in isolation. A pod crash might stem from a node resource shortage, a network policy misconfiguration, an image pull failure, or an application bug. A step-by-step approach prevents wasted effort and ensures each hypothesis is validated before moving to the next.

## Phase 1: Verify the Basics

Before diving into complex diagnostics, confirm the foundational elements.

### Check Cluster Health

```bash
# Control-plane status
kubectl get nodes

# Ready status of all nodes
kubectl get nodes -o wide

# API server accessibility
kubectl cluster-info

# Namespaces and their status
kubectl get namespaces
```

### Confirm Workload State

```bash
# List running pods across namespaces
kubectl get pods -A

# Pods stuck in pending state
kubectl get pods -A --field-selector=status.phase=Pending

# Pods with restart counts
kubectl get pods -A -o=jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}: {.status.containerStatuses[0].restartCount}{"\n"}{end}'
```

## Phase 2: Node-Level Diagnostics

If pods are pending or nodes are unhealthy, investigate the control plane and worker nodes.

### Node Resource Availability

```bash
# Node capacity and usage
kubectl describe nodes <node-name> | grep -A 10 "Allocated resources"

# Disk pressure
kubectl describe nodes <node-name> | grep -i "disk pressure"

# Memory pressure
kubectl describe nodes <node-name> | grep -i "memory pressure"

# PID pressure
kubectl describe nodes <node-name> | grep -i "pid pressure"
```

### Kubelet Status

```bash
# Check kubelet service status on the node
systemctl status kubelet

# Kubelet logs (if accessible)
journalctl -u kubelet -f

# Node condition summary
kubectl get nodes -o jsonpath='{.items[*]}{.status.conditions[?(@.type=="Ready")].status}{"\n"}'
```

### CNI and Networking

```bash
# Check CNI pod status (e.g., Calico, Cilium, Flannel)
kubectl get pods -n kube-system -l k8s-app=calico-node

# Verify pod CIDR allocation
kubectl describe nodes <node-name> | grep -A 5 "CIDR"

# Test connectivity between nodes
kubectl exec -it <pod> -- ping <target-pod-ip>
```

## Phase 3: Pod-Level Investigation

When nodes are healthy but pods fail, examine the pod and container state.

### Pod Restart Policies and Reason

```bash
# Get detailed pod status
kubectl describe pod <pod-name> -n <namespace>

# Key fields to examine:
# - Reason: Why the pod is in Current status (e.g., CrashLoopBackOff, ErrImagePull, OOMKilled)
# - RestartCount: Number of container restarts
# - Message: Often contains the underlying error
```

### Container Status Details

```bash
# View container statuses within a pod
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{range .status.containerStatuses[*]}{.name}: {.ready} / {.state}{"\n"}{end}'

# Check last state (if terminated or waiting)
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{range .status.containerStatuses[*]}{.lastState}{"\n"}{end}'

# Dump container logs
kubectl logs <pod-name> -n <namespace>
```

### Common Pod Failure Patterns

| Symptom | Likely Cause | Quick Check |
|---------|-------------|-------------|
| `CrashLoopBackOff` | Application crash, missing config, signal handling | `kubectl logs`, check entrypoint |
| `ErrImagePull` / `ImagePullBackOff` | Registry auth, wrong image tag, DNS | `kubectl describe pod`, registry logs |
| `OOMKilled` | Memory limit too low, memory leak | `kubectl describe pod`, `kubectl get pod -o json` |
| `Pending` | Insufficient resources, affinity rules | `kubectl describe pod`, `kubectl get nodes` |
| `ContainerCreating` | Volume provisioning, PV binding, SC misconfig | `kubectl get pvc`, `kubectl describe storageclass` |

## Phase 4: Application-Level Debugging

When pods are running but the service misbehaves, investigate the application and its environment.

### Service and Endpoint Status

```bash
# Service discovery status
kubectl get services -A

# Endpoints associated with a service
kubectl get endpoints <service-name> -n <namespace>

# Verify selector matches labels
kubectl get pods -l <selector> -n <namespace>
```

### Ingress and DNS

```bash
# Ingress resource status
kubectl get ingress -A

# Ingress controller pods
kubectl get pods -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Test HTTP endpoint directly
curl -k https://<domain>/healthz

# Bypass ingress, test service directly
kubectl port-forward svc/<service> <local-port>:<target-port>
```

### ConfigMaps and Secrets

```bash
# Verify ConfigMap data is mounted correctly
kubectl get configmap <name> -n <namespace> -o yaml

# Check Secret keys and data
kubectl get secret <name> -n <namespace> -o yaml

# Verify mounted values in a running pod
kubectl exec <pod> -n <namespace> -- cat /etc/<mount-path>/<key>
```

### Resource Quotas and LimitRanges

```bash
# Check namespace resource quotas
kubectl get resourcequotas -n <namespace>

# View limit ranges enforced in a namespace
kubectl get limitrange -n <namespace>
```

## Phase 5: Cluster-Wide Issues

When multiple namespaces or the entire cluster is affected, look at cluster-level components.

### Control Plane Health

```bash
# API server response time
time kubectl get nodes

# etcd health (if self-hosted)
etcdctl endpoint health --endpoints=https://<etcd-endpoint>:2379

# Controller-manager and scheduler logs
kubectl logs -n kube-system deployment/controller-manager
kubectl logs -n kube-system deployment/scheduler
```

### Cluster Autoscaler

```bash
# Check autoscaler status
kubectl get pods -n kube-system -l app.kubernetes.io/name=cluster-autoscaler

# Review scaling events
kubectl describe hpa -A
```

### Audit Logs

```bash
# API server audit log (if enabled)
ls -l /var/log/audit/

# kube-apiserver logs
kubectl logs -n kube-system deployment/kube-apiserver
```

## Systematic Debugging Workflow

When paged at 02:00 for a production incident, follow this sequence:

1. **Confirm scope** — Is this one namespace, one node, or the entire cluster?
2. **Run the basics** — `kubectl get nodes`, `kubectl get pods -A`, `kubectl cluster-info`
3. **Identify the failure mode** — CrashLoopBackOff? Pending? OOMKilled? ImagePullBackOff?
4. **Validate the hypothesis** — Check resource limits, image tags, network policies, config volumes
5. **Apply the minimal fix** — Restart the specific pod, adjust a limit, update a ConfigMap
6. **Verify the resolution** — Confirm the pod transitions to `Running` and the service is healthy
7. **Document the root cause** — Update runbooks, add alerts, or modify manifests to prevent recurrence

## Tooling and Automation

- **kubectl troubleshoot**: A suite of diagnostic bundles (`kubectl troubleshoot bundle`) that collect logs, images, and cluster state.
- **k9s**: Terminal-based UI for real-time cluster monitoring and resource interaction.
- **Lens**: Desktop Kubernetes IDE with built-in debugging tools and YAML editor.
- **kube-score**: Linting tool for Kubernetes manifests (`kube-score check ./`).
- **kube-linter**: CI-integrated linter for manifests and Helm charts.

## Conclusion

A structured debugging approach turns opaque incidents into diagnosable problems. By progressing from cluster-level health checks through node diagnostics, pod investigation, and application-specific verification, DevOps engineers can efficiently isolate root causes and apply targeted fixes. The key is resisting the urge to "fix it immediately" and instead following the evidence wherever it leads. Document each incident, and over time the patterns will inform better defaults, alerts, and runbooks for your cluster.

*This article is intended as a practical debugging guide. Always test changes in a non-critical namespace or staging environment first. Adjust thresholds, resource limits, and network policies to match your cluster topology and workload characteristics.*