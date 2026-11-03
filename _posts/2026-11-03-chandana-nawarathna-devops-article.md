---
layout: post
title: "Kubernetes Security Contexts and Pod Security Standards: A Practical Guide for DevOps Engineers"
date: 2026-11-03 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, security-contexts, pod-security-standards, devops, containers]
description: "A practical guide to configuring security contexts, supplemental groups, and Rancher's Pod Security Standards to harden Kubernetes workloads against common container escape vectors."
---

# Kubernetes Security Contexts and Pod Security Standards: A Practical Guide for DevOps Engineers

When you deploy containers into a Kubernetes cluster, the underlying host kernel runs with elevated privileges. Without explicit guards, a compromised container can escape its boundaries and affect other pods or the node itself. Kubernetes **Security Contexts** and **Pod Security Standards** (PSS) provide the primitives to reduce this risk — from dropping capabilities to enforcing mandatory runtime UIDs.

This article walks through the practical configuration of security contexts, explains the three Pod Security Standards tiers, and shows how to enforce them cluster-wide.

## Why Security Contexts Matter

A container runs as **root (UID 0)** by default inside the container image. If the image has a CVE that allows privilege escalation, the attacker gains root on the node. Security contexts let you:

- Run containers as a non-root user
- Drop Linux capabilities they don't need
- Enforce read-only root filesystems
- Restrict system calls via AppArmor/Seccomp
- Inject supplemental groups for controlled file access

## Security Context Knobs

### Per-Container vs Pod-Level

Security contexts can be set at the container or pod level. Pod-level settings apply as defaults for all containers in the pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    runAsNonRoot: true
  containers:
  - name: my-app
    image: nginx:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

| Field | Effect |
|---|---|
| `runAsUser` | UID the container runs as |
| `runAsNonRoot` | Kube rejects pod if it would run as root |
| `readOnlyRootFilesystem` | Mounts `/` as read-only (tmpfs can be added for writes) |
| `capabilities.drop` | Remove Linux capabilities (e.g., `ALL` removes everything) |
| `capabilities.add` | Add specific capabilities (use sparingly) |
| `privileged: false` | Disallows the privileged flag (default, but explicit is clearer) |

### Per-Container Security Context

```yaml
apiVersion: v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp: v1
        securityContext:
          runAsUser: 1000
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
```

## Pod Security Standards (PSS)

Kubernetes ships with three pre-defined PSS profiles that encode best-practice security configurations. They can be enforced via the **Dynamic Admission Control** plugin or external tools like **OPA Gatekeeper**.

| Tier | Enforced Constraints |
|---|---|
| **privileged** | No constraints (baseline for cluster-admin workloads) |
| **restricted** | Most restrictive: no privileged containers, no root, no hostNamespaces, no hostPIDs, no hostIPC, readOnlyRootFilesystem enforced, capabilities drop ALL |
| **baseline** | Moderate: prevents obvious misconfigurations but allows more flexibility than `restricted` |

### PSS `restricted` Effectively Means

- `runAsNonRoot: true` and `runAsUser: non-zero` for all containers
- `readOnlyRootFilesystem: true`
- `capabilities: { drop: [ALL] }` (or equivalent)
- No `hostPID`, `hostNetwork`, or `hostProcess`
- `allowPrivilegeEscalation: false`

### Enforcing PSS with Built-in AdmissionControl

```bash
# Enable the security context constraints admission plugin
kubectl patch clusterrole system:disruption-budgets -p '{"rules":[{"apiGroups": [""], "resources": ["pods"], "verbs": ["create","update"]}]}'

# Or via KubeAPI server flags (deploy-specific):
# --enable-admission-plugins=...,RestrictVolumeUseLimitRange,SecurityContextDeny
```

Note: The built-in PSS admission controller was removed in Kubernetes 1.29. Use OPA Gatekeeper or the `pod-security-standard` mutating/admission plugins instead.

## Practical Example: Hardening a Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hardened-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hardened-app
  template:
    metadata:
      labels:
        app: hardened-app
    spec:
      # Pod-level security context as default
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        readOnlyRootFilesystem: true
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
        securityContext:
          # Container overrides — further harden
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          # Drop specific capabilities that are rarely needed
          # capabilities:
          #   drop:
          #   - NET_ADMIN
          #   - NET_RAW
        env:
        - name: NODE_ENV
          value: "production"
        resources:
          requests:
            cpu: "200m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
      - name: sidecar
        image: docker.elastic.co/observability/otel-collector:0.85.0
        # sidecar inherits pod securityContext; can override if needed
```

### What This Accomplishes

- Even if the container is exploited, the process runs as UID 1000 (non-root)
- The filesystem is read-only — any write attempt lands on a tmpfs if the app needs it, or fails fast
- ALL Linux capabilities are dropped — no `cap_sys_admin`, `cap_setuid`, etc.
- `allowPrivilegeEscalation: false` prevents `setuid` binaries from elevating privileges

## Verifying the Security Posture

### Check the pod's effective security context

```bash
kubectl get pod <pod-name> -o jsonpath='{.spec.securityContext}'
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[0].securityContext}'
```

### Simulate privilege escalation attempt

```bash
# Run a privileged pod to confirm it's rejected
kubectl run privileged-test --image=nginx --overrides='{"spec":{"securityContext":{"privileged":true}}}'
# Expected: pod crashes with "cannot enable privileged mode: SecurityContextDeny" or similar
```

### Validate with `pss` label (if using Gatekeeper or similar)

```bash
kubectl label pod <pod-name> pod-security.kubernetes.io/restricted=enforce
```

## Common Gotchas

| Issue | Symptom | Fix |
|---|---|---|
| `runAsUser: 0` without `runAsNonRoot: true` | Pod runs as root despite intentions | Add `runAsNonRoot: true` at pod or container level |
| `readOnlyRootFilesystem: true` without tmpfs | App crashes on startup trying to write to `/` | Add a `emptyDir: {}` volume mounted at the path the app needs to write, with `mountPath` protected via `readOnly: false` in the volume mount definition |
| Forgetting `capabilities: { drop: [ALL] }` | Container still has `NET_RAW` or other capabilities | Explicitly drop all, then add back only what's needed |
| PSS `restricted` not enforced | Pods with relaxed security still schedule | Use OPA Gatekeeper `pod-security-policy` constraint template or your CNI's admission plugin |

## Complementary Defenses

Security contexts reduce the *blast radius* if a container is compromised, but they work best as part of a defense-in-depth strategy:

- **Network Policies** (covered earlier) restrict pod-to-pod communication
- **Image scanning** (Trivy) catches CVEs before images enter the cluster
- **OPA Gatekeeper** enforces constraints at admission time
- **Runtime security** (Falco, Aqua, Snyk) monitors for suspicious syscalls at runtime
- **Pod Security Standards** provide the policy framework; the implementation is via admission controllers

## Conclusion

Starting with per-pod security contexts (`runAsNonRoot`, `readOnlyRootFilesystem`, `capabilities.drop: [ALL]`) is the fastest win for hardening any Kubernetes cluster. From there, adopting Pod Security Standards and enforcing them via admission controllers creates a consistent, auditable security posture across namespaces.

Don't try to lock everything down in one go. Begin by adding `runAsNonRoot: true` and `readOnlyRootFilesystem: true` to your namespaces, then iterate toward the `restricted` profile as teams adjust their workloads. The upfront investment prevents the all-too-common container escape incidents that start with a single privileged container.

*This article draws on common Kubernetes security context patterns and Pod Security Standards guidance. Adjust UIDs, GIDs, and capability sets to match your organization's service account and runtime requirements.*

---