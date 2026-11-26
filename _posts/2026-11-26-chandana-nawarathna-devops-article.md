---
layout: post
title: "Kubernetes Debugging: Systematic Troubleshooting Workflow for Production Issues"
date: 2026-11-26 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, debugging, troubleshooting, devops, observability]
description: "A practical, step-by-step debugging workflow for Kubernetes production issues — from pod failures to network problems and scheduled workload incidents."
---

# Kubernetes Debugging: Systematic Troubleshooting Workflow for Production Issues

When a Kubernetes workload misbehaves in production, the instinct is to start applying fixes. A structured workflow prevents wasted time, avoids destructive actions, and makes it easy to track what was tried and why.

This article presents a repeatable debugging pattern that works for pod failures, network issues, scheduled jobs, and node-level problems. All commands are read-only or safe-to-apply; none modify live resources without intent.

## 1. Establish the Baseline

Before diving into logs or APIs, confirm the symptom at the cluster level.

```bash
# List affected pods across all namespaces (replace POD_NAME_PATTERN as needed)
kubectl get pods --all-namespaces | grep POD_NAME_PATTERN

# Check the overall cluster health
kubectl get nodes -o wide

# Verify the component services are up
kubectl get componentstatuses 2>/dev/null || echo "ComponentStatuses not available"
```

**Why this matters**: If no pods are running anywhere, the issue may be cluster-wide (network, API server, kube-proxy). If only one namespace is affected, the scope is bounded.

## 2. Pod-Level Investigation

For a specific failing pod, follow this sequence:

### A. Pod phase and reason

```bash
kubectl get pod <pod-name> -n <namespace> -o json | jq '.status.phase, .status.reasons[]?'
```

Look for phases like `Pending`, `CrashLoopBackOff`, `ErrImagePull`, or `ImagePullBackOff`.

### B. Container status details

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Focus on:
- **Ready** — is the container passing liveness/readiness probes?
- **Restart Count** — repeated restarts indicate an application or resource issue.
- **Last State** — `Terminated` or `Waiting` gives the exit code and error message.

### C. Container logs

```bash
kubectl logs <pod-name> -n <namespace>
# If the container crashes quickly, fetch logs from the last attempt
kubectl logs <pod-name> -n <namespace> --previous
```

### D. Ephemeral container debugging (if the pod is unresponsive)

```bash
# List ephemeral containers already attached (if any)
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.ephemeralContainers}'

# Administer an ephemeral container (requires cluster-admin or appropriate RBAC)
kubectl debug -it <pod-name> -n <namespace> --image=busybox -- /bin/sh
```

**Safety note**: `kubectl debug` creates a temporary container inside the running pod. It does not modify the pod spec, and the temporary container is removed when the shell exits.

## 3. Resource Constraints

If the pod is `CrashLoopBackOff` or `Evicted`, check compute resources:

```bash
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 -i "requests\|limits\|memory\|cpu"
# Or via JSON output
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].resources}'
```

Also check the namespace-level **LimitRange** and **ResourceQuota**:

```bash
kubectl get limitrange -n <namespace>
kubectl get resourcequota -n <namespace>
```

A pod that exceeds its quota will remain in `Pending` state; a pod that exceeds its limit may be OOM-killed or restarted.

## 4. Network Diagnostics

For connectivity issues (service unreachable, ingress timeout):

### A. Service inspection

```bash
kubectl get service <service-name> -n <namespace> -o wide
# Verify the selector matches the app labels
kubectl get pods -l app=<app-label> -n <namespace>
```

### B. Endpoint status

```bash
kubectl get endpoints <service-name> -n <namespace>
```

If endpoints are empty, the pods selected by the service lack the required labels or are unhealthy.

### C. DNS resolution

```bash
# From inside a debug pod or nsenter
nslookup <service-name>.<namespace>.svc.cluster.local
# Or use dig if available
dig <service-name>.<namespace>.svc.cluster.local +short
```

### D. Direct pod-to-pod reachability (skip if you have NetworkPolicy restrictions)

```bash
# Execute inside a source pod and test the target
kubectl exec -it <source-pod> -n <namespace> -- curl -s <target-pod-ip>:8080 || echo "curl not available; try nc"
kubectl exec -it <source-pod> -n <namespace> -- nc -zv <target-pod-ip> 8080
```

### E. Packet capture (read-only, safe on production)

```bash
# Capture traffic to/from a pod for 30 seconds
tcpdump -i any -nn -c 200 -w /tmp/pod-traffic.pcap
# Note: requires CAP_NET_RAW or running as root; adapt for your cluster's security profile.
```

## 5. Scheduled Workload (Job / CronJob) Issues

For a Job that never completes or a CronJob with unexpected behavior:

```bash
# Job status
kubectl get jobs -n <namespace>
kubectl describe job <job-name> -n <namespace>

# Check pod timestamps and restart counts
kubectl get pod -l job-name=<job-name> -n <namespace>

# If the job is stuck in Pending, check quota and node capacity
kubectl describe node <node-name> | grep -i "pressure\|kapacity"
```

## 6. Node-Level Problems

When multiple pods on a single node are affected, or a pod is `Scheduled` but not running:

```bash
# Node conditions
kubectl get nodes -o jsonpath='{.items[0].status.conditions[?(@.type=="Ready")].status}'

# Node pressure events
kubectl describe node <node-name> | grep -i "pressure\|memory\|disk\|pid"

# Kernel logs (if accessible)
journalctl -u kubelet | tail -50
```

**Rollback pattern**: If a recent change (new Deployment, DaemonSet, or node drain) correlates with the issue, revert it:

```bash
kubectl rollout undo deployment/<deployment-name> -n <namespace>
# Or cancel a drain
kubectl uncordon <node-name>
```

## 7. Documentation and Post-Mortem

After the issue is resolved, record:

- **What failed** and at which layer (pod, node, network, control plane)
- **Which commands** identified the root cause
- **What fix** was applied and whether it was a permanent change or a temporary workaround
- **Prevention**: Should a LimitRange, quota, or monitoring alert be added to catch this class of issue earlier?

A brief entry in your team's runbook or issue tracker saves time the next time a similar symptom appears.

## Conclusion

Effective Kubernetes debugging is not about memorizing one-liners — it's about having a systematic workflow that narrows the search space quickly and reliably. Start at the cluster level, drill down to the pod, check resources and networking, and document the outcome.

The patterns above work across Kubernetes distributions (vanilla, GKE, EKS, AKS) and require only `kubectl` and standard Linux tools. Adapt the steps to your cluster's monitoring setup, and always test non-destructive diagnostics first.

*Always test diagnostic commands in a non-critical namespace or staging cluster first. The examples above use read-only or safe inspection patterns. Avoid `kubectl drain`, `kubectl delete`, or `kubectl scale` unless you intend to remove or reconfigure the resource.*

---
*This article draws on common Kubernetes debugging patterns and operational experience. Adjust commands and thresholds to match your cluster's topology, RBAC profile, and SLOs. For distribution-specific guidance, consult your Kubernetes provider's documentation.*