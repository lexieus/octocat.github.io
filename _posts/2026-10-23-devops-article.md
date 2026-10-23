---
layout: post
title: "Kubernetes Network Policies: Enforcing Zero-Trust Security Between Workloads"
date: 2026-10-23 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, network-policies, security, zero-trust, devops, cluster-security]
description: "Practical guide to implementing Kubernetes Network Policies to restrict pod-to-pod communication, with concrete examples and common pitfalls."
---

# Kubernetes Network Policies: Enforcing Zero-Trust Security Between Workloads

By default, Kubernetes allows all pods to communicate with every other pod in the cluster. This "allow-all" baseline is convenient for development but represents a significant security risk in production. Network Policies let you declare which pods can accept traffic from which other pods, namespaces, or IP ranges — effectively implementing zero-trust networking inside your cluster.

## Why Network Policies Matter

In a default Kubernetes cluster, any pod can:
- Initiate connections to any other pod
- Listen on services consumed by any other pod
- Access the cluster's internal DNS and metadata services
- Reach the node network stack and underlying infrastructure

A compromised pod can lateral movement, scan other workloads, or exfiltrate data. Network Policies provide a simple, expressive way to limit this blast radius by describing allowed traffic flows rather than trying to block individual threats.

## Prerequisites

Not all Kubernetes distributions enable NetworkPolicy support by default. Ensure your cluster's CNI plugin supports the NetworkPolicy API:

- **Calico**: Supported out of the box
- **Cilium**: Supported via `NetworkPolicy` CRDs
- **kube-router**: Supported
- **Weave Net**: Supported
- **AWS VPC CNI**: May require additional configuration

Verify support with:

```bash
kubectl get crd networkpolicies.crd.k8s.io
```

If the CRD exists and your CNI is compatible, you can create NetworkPolicy resources immediately.

## Example: Namespace-Isolated Communication

The most common pattern is to isolate namespaces from each other. Consider a cluster with `frontend`, `backend`, and `database` namespaces. Traffic should flow only as follows:

- `frontend` → `backend` (HTTP)
- `backend` → `database` (SQL)
- No direct `frontend` → `database` access

Create a `default-deny` policy first to block all inter-namespace traffic, then allow specific flows:

```yaml
# 1. Default deny — block all cross-namespace traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

This policy selects all pods (`podSelector: {}`) and declares that no ingress or egress is allowed by default. It acts as a catch-all until more specific policies are added.

Now allow the intended flow:

```yaml
# 2. Allow frontend → backend
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
  policyTypes:
  - Ingress
```

This policy applies to pods with `app: backend` and only allows ingress from pods with `app: frontend` on port 8080/TCP.

```yaml
# 3. Allow backend → database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432
  policyTypes:
  - Ingress
  - Egress
```

This restricts database access to only backend pods, and also allows the database pods to initiate egress as needed.

## Selector-Based Policies

NetworkPolicy `spec.podSelector` uses standard label selectors. You can target:

- Specific pods: `matchLabels: { app: database, version: v1 }`
- All pods in a namespace (empty selector): `podSelector: {}`
- Pods matching a condition: `matchLabels: { kubernetes.io/metadata.name: "..." }` (use with caution)

The `ingress` and `egress` lists are additive — if you specify one rule, only those sources/destinations are allowed. An empty `ingress` list blocks all incoming traffic.

## Allowing Traffic from Specific Services

You can reference other NetworkPolicies or endpoints using `namespaceSelector` and `podSelector`. For example, allowing traffic from a monitoring namespace:

```yaml
# 4. Allow monitoring namespace to scrape metrics
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-scrape
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: metrics-endpoint
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9090
  policyTypes:
  - Ingress
```

This allows any pod in the `monitoring` namespace to connect to port 9090 on metrics-endpoint pods.

## Egress Policies: Controlling Outbound Traffic

Ingress policies control inbound traffic, but egress policies control what your pods can send outward. This is useful for preventing data exfiltration or restricting access to external services.

```yaml
# 5. Allow only DNS and HTTP egress from backend pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: limit-backend-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  egress:
  - to:
    - ipBlock:
        cidr: 8.8.8.8/32   # DNS over TCP/UDP via allowed IPs
    - ipBlock:
        cidr: 1.1.1.1/32   # Alternative DNS
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0    # Allow all other egress (adjust as needed)
    ports:
    - protocol: TCP
      port: 443
  policyTypes:
  - Egress
```

In practice, you would typically whitelist only the specific external services your pods need (container registries, package repositories, API endpoints) and deny everything else.

## Common Pitfalls

1. **Policy not taking effect** — NetworkPolicy only affects pods in the same namespace as the policy, unless `namespaceSelector` is used. Ensure the policy's namespace matches the target pods' namespace.

2. **Default deny blocking node traffic** — If you deploy a `deny-all` policy without allowing DNS, nodes cannot resolve names, and pods cannot pull images. Always allow egress to `0.0.0.0/0` on port 53 (UDP/TCP) for DNS before tightening rules.

3. **Selecting the wrong pods** — `podSelector: {}` selects all pods in the policy's namespace. If you meant to target specific pods, add `matchLabels`.

4. **Forgetting policyTypes** — A NetworkPolicy without `policyTypes: [Ingress]` only affects egress, and vice versa. Specify both if you want bidirectional control.

5. **Assuming policies cascade** — NetworkPolicies are not cumulative in the way you might expect. A `deny-all` policy does not automatically allow your specific rules; you must explicitly allow the traffic you need.

## Verification Checklist

After applying NetworkPolicies, confirm they are working as intended:

```bash
# 1. Check that the policy is active
kubectl get networkpolicy <policy-name> -n <namespace> -o yaml

# 2. Verify the CNI has enrolled the policy
kubectl get networkpolicy <policy-name> -o jsonpath='{.metadata.uid}'

# 3. Test ingress from an allowed source
kubectl exec -it <frontend-pod> -- curl -s -o /dev/null -w "%{http_code}" http://<backend-service>:8080/healthz

# 4. Test that denied access is blocked
kubectl exec -it <frontend-pod> -- curl -s -o /dev/null -w "%{http_code}" http://<database-service>:5432/

# 5. Observe policy status in the CNI plugin logs (Calico, Cilium, etc.)
```

## Best Practices

- **Start with a `deny-all` egress policy** and whitelist only the external services your workloads need.
- **Use label selectors consistently** — tag pods with `app`, `tier`, `environment` to make policies reusable across namespaces.
- **One policy per direction per namespace** — multiple NetworkPolicies for the same `ingress`/`egress` direction in the same namespace are combined with OR logic.
- **Test with a non-critical workload first** — deploy the policies against a staging namespace before applying to production.
- **Complement with PodSecurityPolicies or OPA Gatekeeper** — NetworkPolicies handle network-layer isolation; enforce pod-level restrictions separately.
- **Document allowed flows** — maintain a diagram or spreadsheet of which namespace talks to which, on which ports, to aid future reviews.

## Conclusion

Kubernetes Network Policies are the most direct way to reduce the default blast radius in a cluster. By starting with a default-deny posture and explicitly allowing only the traffic your applications need, you move from an "allow-all" baseline to a zero-trust network architecture. The key is to iterate: deploy, verify, tighten, and repeat across namespaces and environments.

*This article draws on Kubernetes NetworkPolicy documentation and real-world cluster hardening patterns. For cluster-specific guidance, consult your CNI plugin's documentation and your organization's security team.*
