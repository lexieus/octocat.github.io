---
layout: post
title: "Kubernetes NetworkPolicies: Controlling Pod-to-Pod Communication"
date: 2026-10-16 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, networkpolicy, networking, security, devops, cilium, calico]
description: "How to define and enforce Kubernetes NetworkPolicies to restrict pod-to-pod traffic, with practical examples using Calico and verification commands."
---

## Overview

Kubernetes Pods are **allow-all** by default — any pod can reach any other pod on the cluster, including system components and other workloads. **NetworkPolicies** let you describe how groups of pods are allowed to communicate with each other and with other network endpoints. Implementing NetworkPolicies is one of the highest-impact security hardening steps for Kubernetes clusters running multi-tenant or multi-workload environments.

This article walks through the practical aspects of defining NetworkPolicies, with concrete examples and verification commands.

## NetworkPolicy Concepts

A NetworkPolicy selects pods using label selectors and defines rules that allow or deny traffic based on:

- **Pod selectors** — which pods the policy applies to
- **Policy types** — `Ingress` (incoming), `Egress` (outgoing), or both
- **Rule sources/destinations** — pod labels, IP blocks, ports
- **Ports** — protocol, port number, name

If no NetworkPolicy exists in a namespace, pods in that namespace retain full connectivity. Once a NetworkPolicy is created that matches a pod, **only the allowed traffic per the policy is permitted** — explicit allow rules replace the default allow-all behavior.

## Example: Restrict Ingress to a Web Frontend

Suppose you have a web frontend (`app: web-frontend`) that should only accept traffic from a backend service (`app: backend`) and an ingress controller (`k8s-app: ingress-nginx`). All other inbound traffic should be denied.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-web-frontend-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web-frontend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-proxy
  - ports:
    - protocol: TCP
      port: 8080
```

### What this achieves

| Setting | Effect |
|---|---|
| `podSelector.matchLabels.app: web-frontend` | Applies the policy only to pods labeled `app: web-frontend` |
| `policyTypes: [Ingress]` | Rules apply to incoming traffic only |
| `from.podSelector.matchLabels.app: backend` | Only pods with `app: backend` can initiate connections to the web frontend |
| `namespaceSelector.matchLabels.kubernetes.io/metadata.name: ingress-proxy` | Only traffic from the `ingress-proxy` namespace is allowed |
| `ports.port: 8080` | Restricts the allowed port to 8080 TCP |

## Example: Restrict Egress from a Database Pod

A database pod (`app: postgres`) should only be able to resolve DNS and reach a specific Redis instance — nothing else.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-postgres-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: redis
    - ipBlock:
        cidrSelector: 10.0.0.0/16
    ports:
    - protocol: TCP
      port: 5432
    - protocol: UDP
      port: 53
```

### What this achieves

| Setting | Effect |
|---|---|
| `policyTypes: [Egress]` | Rules apply to outgoing traffic only |
| `to.podSelector.matchLabels.app: redis` | Only Redis pods can be reached |
| `ipBlock.cidrSelector: 10.0.0.0/16` | DNS resolution and general IP communication limited to the cluster CIDR |
| `ports.port: 53` | Only DNS queries (UDP/TCP port 53) are allowed |
| `ports.port: 5432` | Only PostgreSQL wire protocol is allowed to the selected destination |

## Example: Allow All Egress for Diagnostics

Some workloads need full egress for health checks, metrics export, or package updates. You can allow all egress while restricting ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-diagnostics-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: monitoring-agent
  policyTypes:
  - Egress
  egress:
  - {}
```

The empty rule `{}` matches all egress traffic. This pattern is useful when you want to **deny ingress** but **allow all outward connectivity** for telemetry and updates.

## Verification

After applying NetworkPolicies, confirm they are active and behaving as expected:

```bash
# List NetworkPolicies in a namespace
kubectl get networkpolicy -n production

# Check which policies affect a specific pod
kubectl get networkpolicy -n production --selector=app=web-frontend

# Describe a specific policy to see rules and selectors
kubectl describe networkpolicy restrict-web-frontend-ingress -n production

# Verify connectivity from a pod (from debug shell)
kubectl exec -it <pod> -- curl -s http://<target-pod>:8080/healthz

# Confirm a policy is enforcing (traffic denied)
kubectl exec -it <pod> -- curl -s -o /dev/null -w "%{http_code}" http://<denied-pod>:8080/
```

## Common Pitfalls

1. **Forgot to add `policyTypes`** — without specifying `Ingress`, `Egress`, or both, the policy is ignored by the CNI plugin.
2. **Selectors don't match** — ensure the `podSelector` labels actually exist on the target pods; a mismatched selector silently allows all traffic.
3. **Default-allow semantics** — remember that creating a NetworkPolicy replaces the default allow-all for matching pods only. Add a catch-all rule if you need to explicitly allow some traffic.
4. **Host network pods** — NetworkPolicies do not apply to host-namespace pods unless the CNI plugin explicitly supports them.
5. **Port specification** — if you specify `ports` without a `port` number, the rule may not match as expected. Always include an explicit port or use `{} `to mean "any port".

## Best Practices

- **Start with a "allow-list" approach** — begin by allowing only the minimal set of required communications, then iterate.
- **Use consistent label patterns** — apply the same label conventions (`app:`, `role:`, `env:`) across all workloads to make selectors predictable.
- **Scope policies to specific namespaces** — avoid cluster-wide NetworkPolicies unless absolutely necessary; namescope reduces blast radius.
- **Pair with PodSecurityPolicies or Pod Security Standards** — NetworkPolicies handle network access; security contexts handle process privilege. Both are needed for defense-in-depth.
- **Test with a "permissive" policy first** — create a policy with `spec.ingress = [{}]` to confirm the CNI is enforcing NetworkPolicies before writing restrictive rules.
- **Automate policy validation** — integrate `kubectl get networkpolicy` checks into CI pipelines to detect accidental policy deletion or misconfiguration.

## Conclusion

NetworkPolicies are the primary mechanism for restricting pod-to-pod communication in Kubernetes. When combined with Security Contexts (which control process-level privileges), they provide a complete security boundary at both the network and compute layers. Start by identifying critical workloads and applying ingress/egress policies that allow only the minimum required traffic. Iterate as you validate connectivity and collect feedback from application teams.

*Further reading: Kubernetes NetworkPolicies specification, Calico NetworkPolicy guide, Cilium NetworkPolicy, CNI security recommendations*

---
*This article was generated for chandanadev.com — DevOps, Linux, Kubernetes, cloud infrastructure, CI/CD, Terraform, monitoring, security, and open-source technology.*