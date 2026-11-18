---
layout: post
title: "Kubernetes Troubleshooting: Systematic Debugging When Pods Fail"
date: 2026-11-18 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, troubleshooting, devops, observability, pod-debugging, clusters]
description: "A practical guide to systematically diagnose and resolve common Kubernetes pod failures, from image pull errors to network policy misconfigurations."
---

# Kubernetes Troubleshooting: Systematic Debugging When Pods Fail

By default, Kubernetes hides the complexity of your workloads behind desired-state declarations. When a pod crashes, the `Pending`, `CrashLoopBackOff`, or `ImagePullBackOff` status leaves many engineers guessing. This article walks through a systematic troubleshooting framework you can apply regardless of the error type, with real-world `kubectl` commands and minimal-risk examples.

## The Troubleshooting Framework

Before running commands, follow this sequence:

1. **Confirm the pod state** — `kubectl get pods -o wide`
2. **Check the pod events** — `kubectl describe pod <name>`
3. **Inspect the container logs** — `kubectl logs <pod>`
4. **Inspect container status** — `kubectl get pod <name> -o jsonpath='{.status.containerStatuses[*]}'`
5. **Validate the underlying node** — `kubectl get nodes -o wide`

## Common Failure Modes and Fixes

### 1. `ImagePullBackOff` — Registry or Credential Issue

A pod cannot pull the container image. Common causes:

- Missing or incorrect `imagePullPolicy`
- Invalid image name or tag
- Registry authentication failure
- Network unreachable to the registry

**Debugging steps:**

```bash
# 1. Check the pod events for the specific event
kubectl describe pod my-app-abc123

# 2. Verify the image name and tag are correct
kubectl get pod my-app-abc123 -o jsonpath='{.spec.containers[*].image}'

# 3. Test registry connectivity from the node (if you have shell access)
docker pull myregistry/my-app:tag

# 4. If using a private registry, verify the secret exists
kubectl get secret my-registry-secret -n production
```

**Minimal fix example — add or update the image pull secret:**

```bash
kubectl create secret docker-registry my-registry-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myuser@example.com
```

Then reference it in your Deployment:

```yaml
spec:
  containers:
  - name: app
    image: myregistry/my-app:latest
  imagePullSecrets:
  - name: my-registry-secret
```

### 2. `CrashLoopBackOff` — Application Crashes on Startup

The container starts and exits repeatedly. Causes include:

- Missing environment variables or configuration
- Application bugs (unhandled signals, failed DB connections)
- Resource limits too low (OOMKilled)
- Entrypoint/script errors

**Debugging steps:**

```bash
# 1. Get the container logs (use -f to follow)
kubectl logs my-app-abc123

# 2. If the container exits quickly, inspect the exit code and signal
kubectl get pod my-app-abc123 -o jsonpath='{.status.containerStatuses[0].state.terminated.reason}{ "\n" }{ .status.containerStatuses[0].state.terminated.exitCode}{ "\n"}'

# 3. Run the container interactively to reproduce the issue
kubectl run -it --rm debug-shell --image=nicolaka/netshoot --restart=Never -- /bin/bash
# Then: kubectl exec -it my-app-abc123 -- /bin/sh
```

**Minimal fix example — check liveness/readiness probes:**

If your app needs time to warm up, adjust the probe delays:

```yaml
spec:
  containers:
  - name: app
    image: myregistry/my-app:latest
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

### 3. `Pending` — Pod Cannot Be Scheduled

The pod remains in `Pending` status, typically because:

- Insufficient resources (CPU/Memory) on any node
- `nodeSelector` or `affinity` rules cannot be satisfied
- PersistentVolume claims cannot be bound
- Taints and tolerations mismatch

**Debugging steps:**

```bash
# 1. Check the reason and message
kubectl describe pod my-app-abc123 | grep -A 5 "Warning.*FailedScheduling"

# 2. List nodes with their allocatable resources
kubectl get nodes -o wide

# 3. Check resource requests/limits against node capacity
kubectl get pod my-app-abc123 -o jsonpath='{.spec.containers[*].resources.requests}'
```

**Minimal fix example — add resource requests:**

If no node has enough free memory/CPU, increase the node pool or reduce requests. As a quick check:

```yaml
spec:
  containers:
  - name: app
    image: myregistry/my-app:latest
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "500m"
```

### 4. `Evicted` — Disk Pressure or Manual Eviction

A node ran out of disk space and the kubelet evicted pods.

**Debugging steps:**

```bash
# 1. Check node disk usage
kubectl describe node <node-name> | grep -A 5 "DiskPressure"

# 2. Check pod volumes and claims
kubectl get pod my-app-abc123 -o jsonpath='{.spec.volumes[*].persistentVolumeClaim.claimName}'
```

**Minimal fix example — enforce quota or increase storage:**

Add a `storageClassName` to your PVC, or adjust `volumeClaimTemplate` requests in your StatefulSet.

## Systematic Debugging Checklist

When a pod fails, run through this sequence every time:

| Step | Command | What you learn |
|------|---------|----------------|
| 1 | `kubectl get pods -A` | Cluster-wide pod health |
| 2 | `kubectl describe pod <name>` | Events, restart count, IP, status |
| 3 | `kubectl logs <pod>` | Application stdout/stderr last lines |
| 4 | `kubectl logs <pod> --previous` | Logs from the previous container instance (useful after restart) |
| 5 | `kubectl get pod <pod> -o json` | Full pod spec and status JSON for deep inspection |
| 6 | `kubectl get nodes` | Node readiness, conditions, capacity |
| 7 | `kubectl get events --field-selector involvedObject.name=<pod>` | Node-level events tied to the pod |

## Defensive Practices to Reduce Future Debugging

| Practice | Why it helps |
|----------|--------------|
| **Health probes** (`livenessProbe`, `readinessProbe`) | Kubernetes can detect and restart unhealthy pods automatically |
| **Resource requests/limits** | Prevents `OOMKilled` and ensures fair scheduling |
| **ImagePullSecrets** in every pod that uses private registries | Avoids `ImagePullBackOff` at scale |
| **Node labels + affinity/anti-affinity** | Guarantees pods land on appropriate nodes (e.g., GPU, dedi, bare-metal) |
| **Limit pod disruption budgets** | Controls max unavailable during upgrades, avoids surprise evictions |
| **Centralized logging** (Fluent Bit, Loki, ELK) | `kubectl logs` works even after pod restart/replacement |
| **Cluster-level health checks** (Prometheus + node exporter) | Detects resource pressure, disk pressure, and node conditions before pods fail |

## Conclusion

Kubernetes troubleshooting becomes far less chaotic when you treat it as a repeatable framework rather than a guessing game. Start with the `describe` → `logs` → `events` sequence, validate the node conditions, and apply the minimal fix from the checklist above. Over time, add centralized logging and alerting on pod restart counts, OOM kills, and pending durations so you’re warned before users report issues.

*Always test changes in a staging or development namespace first. The commands above use safe, read-only or non-destructive flags; avoid `kubectl delete` or `kubectl drain` unless you intend to remove or cordon the resource.*

---
*This article is intended as a practical troubleshooting guide. Adjust thresholds, probe configurations, and resource values to match your application’s SLOs and cluster topology. If you encounter cluster-specific issues (e.g., CNI plugin behavior, kube-proxy timeouts), consult your CNI and control-plane documentation.*