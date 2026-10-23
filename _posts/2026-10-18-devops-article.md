---
layout: post
title: "Kubernetes Security Contexts: Running Pods with Least Privilege"
date: 2026-10-18 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, security, pods, security-contexts, hardening]
description: "How to use Kubernetes security contexts to run pods with least privilege, including runAsUser, runAsGroup, capabilities, and seccomp profiles with practical examples."
---

# Kubernetes Security Contexts: Running Pods with Least Privilege

When deploying containers at scale, assuming containers run with the same privileges as the node is a common misconfiguration. Kubernetes **security contexts** let you declare exactly what a container can and cannot do — reducing the attack surface if a container is compromised.

This article covers the practical use of security contexts with real-world manifests.

## What Is a Security Context?

A security context defines privilege and permission settings for a container or pod. It can configure:

- **User IDs** — `runAsUser`, `runAsGroup`, `fsGroup`
- **Capability management** — adding or dropping Linux capabilities
- **Seccomp profiles** — system call filtering
- **Privilege escalation** — preventing `setuid`, `setgid`, and kernel module loading

Security contexts apply at both the **container** and **pod** level. If set at the pod level, they apply to all containers in the pod.

## Container-Level Security Context

Here is a minimal example that runs a container as a non-root user and drops all capabilities:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-webapp
spec:
  containers:
  - name: app
    image: nginx:1.27
    securityContext:
      runAsUser: 1000
      runAsGroup: 3000
      capabilities:
        drop:
        - ALL
```

**What this does:**

| Field | Effect |
|---|---|
| `runAsUser: 1000` | Container starts as user ID 1000 |
| `runAsGroup: 3000` | Primary group inside the container is 3000 |
| `capabilities.drop: [ALL]` | All Linux capabilities are removed; none are added by default |

**Important:** The entrypoint or image may still attempt privilege escalation. Always pair `drop: ALL` with `runAsUser` for defense-in-depth.

## Dropping Specific Capabilities

Instead of dropping all capabilities, you can selectively drop only the dangerous ones while keeping others needed by your workload:

```yaml
securityContext:
  capabilities:
    drop:
    - NET_RAW
    - SYS_ADMIN
    add: []  # explicitly ensure no capabilities are added
```

**Common capabilities to drop:**

- `NET_RAW` — prevents crafting custom network packets
- `SYS_ADMIN` — a "catch-all" for many dangerous syscalls
- `SETUID`, `SETGID` — prevents privilege switching

**Keep capabilities your workload actually needs** (e.g., `NET_BIND_SERVICE` for binding to ports above 1024, or `TIME_SYNC` for container time synchronization).

## Preventing Privilege Escalation

Even when running as a non-root user, some processes may still attempt `setuid` or loading kernel modules. Add `allowPrivilegeEscalation: false` to prevent this:

```yaml
securityContext:
  runAsUser: 1000
  allowPrivilegeEscalation: false
```

This disables `su`, `sudo`, and any exec-based privilege elevation within the container.

## Seccomp Profiles

Seccomp (secure computing mode) filters which system calls a container can make. Kubernetes defaults to `RuntimeDefault` (the runtime's default profile), but you can specify `LocalhostProfile` to use a profile defined on the node, or `Unconfined` to disable the restriction (not recommended).

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

For a custom profile defined via `kubectl apply -f profile.json` on the node:

```yaml
securityContext:
  seccompProfile:
    type: LocalhostProfile
```

**Rule of thumb:** Use `RuntimeDefault` unless you have a locally compiled profile that explicitly allows your workload's syscalls.

## Pod-Level Security Context

Setting security context at the pod level applies it to all containers in the pod. This is useful for setting a shared `fsGroup` for volume access:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-volume
spec:
  securityContext:
    fsGroup: 4000
  containers:
  - name: app
    image: nginx:1.27
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}
```

In this example, any file created inside `/data` will be owned by group ID 4000, and containers can `chown` files to that group.

## Practical: Hardening a Deployment

Here is a complete manifest combining multiple security context settings:

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
      containers:
      - name: app
        image: myregistry.io/app:1.2.0
        securityContext:
          runAsUser: 1001
          runAsGroup: 3001
          capabilities:
            drop:
            - ALL
          allowPrivilegeEscalation: false
          seccompProfile:
            type: RuntimeDefault
        ports:
        - containerPort: 8080
        env:
        - name: NODE_ENV
          value: production
```

**Verification:** After deploying, confirm the container runs as the expected user:

```bash
kubectl exec hardened-app-7c6b8d9f8-2x5z6 -- id
# Expected output: uid=1001(app) gid=3001(app) groups=3001(app)
```

Also confirm capabilities are dropped:

```bash
kubectl exec hardened-app-7c6b8d9f8-2x5z6 -- cat /proc/self/status | grep Cap
# Expected: Cap: 0000000000000000 (no capabilities)
```

## When to Use Separate Container vs. Pod Context

| Situation | Use containerSecurityContext | Use podSecurityContext |
|---|---|---|
| Different users per container | ✅ | ❌ |
| Shared `fsGroup` for volume access | ❌ | ✅ |
| Disabling privilege escalation globally | ✅ | ✅ |
| System call filtering per container | ✅ | ❌ |

Set both at the appropriate level for your threat model.

## Common Pitfalls

1. **`runAsUser` has no effect if the image already has a USER instruction** — some images `USER`s into a non-root user during build. Check with `kubectl exec ... id`.
2. **Dropping `SYS_TIME` can break time synchronization** in clusters using `Chrony` or `systemd-timesyncd`.
3. **`allowPrivilegeEscalation: false` combined with `runAsUser`** is the safest default for untrusted workloads.
4. **fsGroup only affects emptyDir and persistentVolumeClaim volumes** — it does not change the user ownership of existing files outside the mounted path.
5. **Not all containers boot successfully as non-root** — some expect write access to `/tmp`, `/var/run`, or other system directories. Add `allowPrivilegeEscalation: false` gradually and test.

## Conclusion

Security contexts are a low-effort, high-impact hardening measure. Start with these three steps for every pod:

1. Set `runAsUser` and `runAsGroup` to non-root IDs.
2. Drop all unnecessary capabilities with `capabilities.drop: [ALL]`.
3. Set `allowPrivilegeEscalation: false`.

From there, add seccomp profiles and selective capability keeps as your threat model requires. Combined with network policies, pod security policies (if your cluster supports them), and regular vulnerability scanning, you significantly reduce the risk surface of your Kubernetes workloads.

*This article draws on Kubernetes security best practices and CIS Benchmark recommendations. For cluster-specific guidance, consult your organization's security team and the [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standard/).*
