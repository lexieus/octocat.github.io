---
layout: post
title: "Kubernetes Network Policies: Controlling Pod-to-Pod Communication"
date: 2026-11-04 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, network-policies, security, devops, containers]
description: "A practical guide to writing and applying NetworkPolicy objects that restrict pod communication, with concrete examples and common patterns for production clusters."
---

# Kubernetes Network Policies: Controlling Pod-to-Pod Communication

When you run workloads in a shared Kubernetes cluster, every pod can potentially communicate with every other pod and with network endpoints outside the cluster. This default-allow posture works fine for early prototypes, but in production you typically need granular traffic controls to limit the blast radius of misconfigured or compromised containers, meet compliance requirements, and enforce domain-driven service boundaries.

This article walks through the practical aspects of writing and applying `NetworkPolicy` objects, with patterns you can adopt immediately.

## Why Network Policies Matter

Kubernetes' built-in network plugins (Calico, Cilium, Romana, etc.) implement the NetworkPolicy API to enable fine-grained egress and ingress filtering. Without them, all pods share a flat network — a single compromised container can scan the cluster, exfiltrate data, or pivot to other workloads.

Network policies let you:

- Allow only the specific inbound traffic a pod needs
- Restrict outbound calls to external services (databases, APIs, registries)
- Enforce namespace isolation so teams can only reach their own workloads
- Segment workloads by function (frontend, backend, database, cache)

## Basic Building Blocks

A `NetworkPolicy` has four primary fields that control its behavior:

| Field | What it controls |
|---|---|
| `podSelector` | Which pods the policy applies to (based on labels) |
| `selector` | Which pods *outside* the policy's scope must match for the rule to apply |
| `ingress` | Inbound traffic rules (allowed sources and ports) |
| `egress` | Outbound traffic rules (allowed destinations and ports) |

If `podSelector` is omitted, the policy selects all pods in the namespace. If both `ingress` and `egress` are empty, the policy defaults to **deny all** traffic for the selected pods.

## Example 1: Allow Inbound Traffic from a Specific Service

Suppose you have a frontend deployment labeled `app: frontend` and a backend labeled `app: backend`. The backend should only accept traffic from the frontend — no other pods can reach it on port 8080.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

What this does:

- `podSelector` targets pods with `app: backend`
- `ingress.from` allows traffic only from pods with `app: frontend`
- The rule applies to TCP port 8080; any other port or source is blocked

## Example 2: Restrict Outbound Calls to a Database Service

A backend pod needs to connect only to a PostgreSQL service on port 5432. No other outbound destinations should be reachable.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-backend-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  egress:
  - to:
    - ipBlock:
        ip: 10.55.10.0/24
        except: [10.55.10.10/32]
    ports:
    - protocol: TCP
      port: 5432
```

Key details:

- `egress.to` limits the rule to a specific CIDR block (your database subnet)
- `ipBlock.except` excludes a single host (e.g., a monitoring endpoint) if needed
- The rule only allows TCP port 5432; all other outbound traffic is dropped

## Example 3:Namespace-Level Isolation

In multi-tenant clusters, you may want each namespace to only talk to its own pods and a curated set of shared services. This policy ensures pods in the `monitoring` namespace can only receive traffic from other `monitoring` pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: monitoring-isolation
  namespace: monitoring
spec:
  podSelector: {}
  ingress:
  - from:
    - podSelector:
        matchLabels:
          namespace: monitoring
  egress:
  - to:
    - podSelector:
        matchLabels:
          namespace: monitoring
```

Note: `podSelector: {}` selects all pods in the namespace. The `from.podSelector.matchLabels` restricts sources to pods that also have `namespace: monitoring` (this relies on a label you add to all monitoring pods).

## Example 4: Allow DNS and Outbound Internet for Containers

Most containers need DNS resolution and often need to pull images from registries. Here's a policy that permits the essential control-plane and infrastructure traffic while blocking everything else:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: essential-traffic-only
  namespace: default
spec:
  podSelector: {}
  ingress:
  - from:
    - ipBlock:
        ip: 0.0.0.0/0  # allow all inbound IPs (adjust as needed)
    ports:
    - protocol: TCP
      port: 53  # DNS
  egress:
  - to:
    - ipBlock:
        ip: 8.8.8.8/32  # allow Google DNS
      ports:
      - protocol: TCP
        port: 53
    - to:
        namespace: kube-system
      ports:
      - protocol: TCP
        port: 443  # allow Kubernetes API (if needed)
```

In practice, most clusters use a network plugin that already allows DNS and container runtime traffic. This pattern is useful when you want to explicitly document and constrain what's permitted.

## Debugging and Verification

After applying a NetworkPolicy, you can verify it with:

```bash
# List all NetworkPolicies in the namespace
kubectl get networkpolicies -n default

# Show the YAML of a specific policy
kubectl get networkpolicy allow-frontend-to-backend -o yaml

# Debug: try to reach a pod from another pod and check connectivity
# With Calico, you can use calicoctl:
calicoctl networkpolicy get allow-frontend-to-backend -o yaml

# Test connectivity from within a pod:
# From a shell inside a frontend pod:
curl http://backend-service:8080
# If the policy blocks it, the connection will timeout or be reset
```

Common pitfalls:

- **Forgot the `com.github.azure.cni` or `projectcalico.cnipolicies` CRD** — some network plugins require an extra step to enable the API
- **Selector mismatch** — if the `podSelector` or `selector` labels don't match any pods, the policy has no effect (it's not an error, just silent)
- **Adding a policy without a default-allow baseline** — suddenly all pod-to-pod traffic stops, which can be alarming until you realize you need a permissive policy for essential services
- **Relying on `except` to carve out exceptions** — `except` blocks are CIDR-based; if you need fine-grained exceptions, consider using multiple policies or a more expressive plugin like Cilium

## Common Patterns for Production Clusters

### 1. Allow Ingress from an External Load Balancer

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-lb-ingress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: app
  ingress:
  - from:
    - ipBlock:
        ip: 203.0.113.0/24  # your LB's public IP range
    ports:
    - protocol: TCP
      port: 443
```

### 2. Restrict Egress to a Specific Registry

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-registry-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: builder
  egress:
  - to:
    - ipBlock:
        ip: 52.1.234.0/24  # Docker Hub IP range (check current ranges)
      ports:
      - protocol: TCP
        port: 443
```

### 3. Allow Only Health Check Calls from Kube-Probe

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-health-checks
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          k8s-app: kube-proxy
    ports:
    - protocol: TCP
      port: 8080
```

Note: In newer Kubernetes versions, `namespaceSelector` is preferred over matching labels on pods in the `kube-system` namespace.

## Conclusion

Network policies are one of the most effective levers for reducing the attack surface of a Kubernetes cluster. Start with a permissive policy that allows your known traffic patterns, then iterate toward deny-by-default rules that only allow the specific inbound and outbound connections each workload needs.

The fastest win is adding a policy that restricts your database pods to only accept connections from your application tier. From there, you can segment frontend, backend, cache, and supporting services into isolated groups with explicit allow rules.

*This article draws on common NetworkPolicy patterns from the Kubernetes documentation and real-world cluster deployments. IP ranges and service names should be adjusted to match your environment. If your cluster uses a network plugin with extended features (e.g., Cilium's wireguard-based mesh or Calico's felix profiles), consult its specific documentation for any complementary configuration.*

---