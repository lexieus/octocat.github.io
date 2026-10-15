---
layout: post
title: "Kubernetes Security Contexts: Running Pods with Least Privilege"
date: 2026-10-15 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, security, security-contexts, pods, privilege-escalation, devops]
description: "How to configure security contexts in Kubernetes to drop unnecessary capabilities, run as non-root users, and reduce the attack surface of your workloads."
---

## Overview

Kubernetes pods run with broad privileges by default. If a container is compromised, the attacker inherits the same access the pod had on the node — often root access, arbitrary system calls, and the ability to modify host-state. **Security contexts** let you reduce this risk by specifying what a container can and cannot do.

This article walks through the most effective security context settings, with YAML examples and verification commands.

## Security Context Concepts

A security context can be set at the pod level or the container level. It controls:

- **`runAsUser`** — the UID the container runs as
- **`runAsGroup`** — the primary group the container runs as
- **`runAsNonRoot`** — prevents the container from running as root
- **`capabilities`** — Linux capabilities to drop or add
- **`readOnlyRootFilesystem`** — mounts the root filesystem as read-only
- **`privileged`** — grants extended privileges (equivalent to `--privileged` in Docker)

By default, if none are specified, containers run as root with all capabilities — this is the highest-risk configuration.

## Example: Well-Hardened Deployment

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
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        runAsNonRoot: true
      containers:
      - name: app
        image: your-registry/my-app:latest
        ports:
        - containerPort: 8080
        securityContext:
          runAsUser: 1000
          capabilities:
            drop: ["ALL"]
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
        env:
        - name: CONFIG_DIR
          value: "/etc/app"
        volumeMounts:
        - name: config
          mountPath: "/etc/app"
      volumes:
      - name: config
        configMap:
          name: my-app-config
```

### What this achieves

| Setting | Effect |
|---------|--------|
| `runAsNonRoot: true` at pod level | The kube scheduler will reject the pod if it would otherwise run as UID 0 |
| `runAsUser: 1000` | Container starts as UID 1000 instead of root |
| `capabilities: drop: ["ALL"]` | All kernel capabilities are removed — the container cannot `mount`, `chmod`, `chroot`, etc. |
| `readOnlyRootFilesystem: true` | If malware tries to write to the container's own filesystem, it fails |
| `allowPrivilegeEscalation: false` | Even if `setcap` is used, the kernel blocks elevation |

## Verification

After deploying, confirm the security context is active:

```bash
# Check the pod is running as the expected UID
kubectl get pod my-app-XXXXX -o jsonpath='{.spec.securityContext.runAsUser}'

# Confirm the QoS class and user identity
kubectl get pod my-app-XXXXX -o jsonpath='{.status.qosClass}'

# List the effective capabilities (should be empty after drop ALL)
kubectl exec my-app-XXXXX -- cat /proc/self/status | grep Cap

# Verify the filesystem is read-only inside the container
kubectl exec my-app-XXXXX -- touch /etc/app/config 2>&1 && echo "writeable" || echo "read-only"
```

## Common Pitfalls

1. **`runAsNonRoot` without `runAsUser`** — the pod may still run as root if no UID is specified; always pair with an explicit UID.

2. **Dropping all capabilities then adding none back** — some workloads need specific capabilities (e.g., `NET_BIND_PORT` for ports < 1025). Start with `drop: ["ALL"]` and selectively add only what's needed.

3. **`readOnlyRootFilesystem` breaks volume mounts** — if a container needs to write to a `configMap` or `emptyDir` at runtime, the write will fail. Use `subPath` or adjust the mount path.

4. **Running as UID 0 inside the container despite `runAsNonRoot`** — some images have an entrypoint that `su` to root or use `socat` to expose the Docker socket. Verify the actual UID after the first startup.

## Best Practices

- **Start with `runAsNonRoot: true` and `runAsUser: 1000`** at the pod level, then add container-level settings as needed.
- **Drop ALL capabilities, then add selectively** — use `kubectl exec` to test what your application actually needs (e.g., ping, netbind, time).
- **Pair `readOnlyRootFilesystem` with proper volume handling** — config data should be pre-populated via ConfigMaps or Secrets; runtime writes should go to emptyDirs or persistent volumes explicitly allowed.
- **Test in staging first** — security context changes can break applications that assume root privileges (e.g., binding to privileged ports, writing to `/tmp`).
- **Use `allowPrivilegeEscalation: false`** as a belt-and-suspenders check — even if `setcap` is set somewhere in the image, this blocks the kernel from honoring it.

## Conclusion

Security contexts are one of the highest-impact, lowest-effort hardening measures in a Kubernetes workload. A default `runAsNonRoot: true` with `capabilities: drop: ["ALL"]` and `readOnlyRootFilesystem: true` closes the most common attack paths without requiring a new runtime or service mesh. Start with these three settings across all your deployments, then iteratively add or relax settings based on application feedback.

*Further reading: Kubernetes Security Contexts, CIS Kubernetes Benchmark, Capabilities reference, OCI runtime spec privileges*