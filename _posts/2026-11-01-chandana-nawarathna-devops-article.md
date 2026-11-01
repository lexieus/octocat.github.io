---
layout: post
title: "Kubernetes Network Policies: Controlling Pod-to-Pod Communication"
date: 2026-11-01 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, network-policies, security, observability, cilium, calico, devops]
description: "A practical guide to planning, implementing, and validating Kubernetes Network Policies to restrict arbitrary pod communication and reduce the cluster attack surface."
---

# Kubernetes Network Policies: Controlling Pod-to-Pod Communication

By default, Kubernetes allows all pods to communicate with every other pod in the cluster. This "allow-everything" baseline works for local development and simple proofs-of-concept, but it is unacceptable in production where isolation, compliance, and breach containment matter. Network Policies let you declare which pods can accept traffic from which other pods, namespaces, or network endpoints — effectively turning Kubernetes into a micro-segmented environment.

## Why Network Policies Matter

A compromised pod, a misconfigured service, or a container running with excessive privileges can lateral-move across the cluster if there are no network restrictions. Network policies answer three practical questions:

1. **Which pods can talk to my database?**
2. **Can frontend pods reach the internal metadata server?**
3. **Is a sidecar container able to reach the network debugger?**

Without policies, the answer to all three is "anyone." With policies, you define the contract and the Kubernetes network plugin enforces it.

## Planning the Policy Model

Before writing YAML, decide on the segmentation strategy. Common patterns include:

- **Namespace-based isolation**: Assign workloads to dedicated namespaces and apply policies per namespace. This is the simplest starting point.
- **Role-based segmentation**: Group pods by function (frontend, backend, database, infrastructure) and allow only the necessary intersections.
- **Label-driven policies**: Use consistent labels (`app.kubernetes.io/part`, `tier`, `environment`) so policies remain stable across rollouts.

A typical production layout might have four tiers:

| Namespace | Role | Typical Pods |
|-----------|------|--------------|
| `frontend` | entry | Ingress, React/Svelte apps |
| `backend` | service | Go/Java/Python services |
| `database` | state | PostgreSQL, MySQL, Redis |
| `monitoring` | infra | Prometheus, Grafana, Alertmanager |

## Building a Allowlist Policy

The most common first policy is to allow frontend pods to reach backend services only. This blocks direct database access from the internet or internal tooling.

Assume pods have the label `app.kubernetes.io/part: frontend` and backend services have `app.kubernetes.io/part: backend`.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  podSelector: {}        # applies to all pods in the namespace
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/part: frontend
      to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/part: backend
  egress: []             # no outbound restrictions yet
```

**What this does:**

- `ingress` restricts inbound traffic to pods matching `app.kubernetes.io/part: frontend`.
- `podSelector` under each rule selects the destination pods (backend services).
- `policyTypes: [Ingress]` means only ingress rules are evaluated; egress is left unrestricted for now.

Apply it:

```bash
kubectl apply -f allow-frontend-to-backend.yaml
```

## Blocking Unwanted Traffic

Just as important as allowing traffic is explicitly blocking it. A catch-all deny policy ensures that if a new pod is deployed without the correct labels, it cannot receive traffic from unexpected sources.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress-default-deny
  namespace: default
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress: []            # empty rules = deny all by default
```

Apply this in any namespace where you want a strict default-deny posture. Combine it with allowlist policies so that only explicitly permitted traffic flows.

## Using Cilium or Calico for Richer Policies

If your cluster uses a CNI plugin that extends the Kubernetes NetworkPolicy API (Cilium, Calico, Antrea), you get additional capabilities beyond the standard `networking.k8s.io/v1` subset.

### Cilium NetworkPolicy example: layer-4 + layer-7 rules

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-frontend-to-backend-https
  namespace: default
spec:
  selector:
    matchLabels:
      app.kubernetes.io/part: backend
  ingress:
    - fromEndpoints:
        - matchLabels:
            app.kubernetes.io/part: frontend
    - ports:
        - port: "443"
          protocol: TCP
  egress: []
```

This blocks any traffic that is not HTTPS (port 443/TCP) between the specified endpoints, even if a generic `allow-all` NetworkPolicy exists.

### Calico NetworkPolicy with IP blocks

Calico can reference IP CIDRs directly, which is useful for on-premises or hybrid setups where you want to allow traffic only from specific IP ranges.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-specific-ip-block
  namespace: default
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - ipBlock:
            cidr: 203.0.113.0/24
            except: ["203.0.113.100/32"]
      to:
        - podSelector: {}
```

## Debugging and Validation

A common pitfall is writing a policy that is too restrictive and then being unable to reach the service from the expected source. Use these techniques to validate:

### Check policy status

```bash
kubectl get networkpolicy allow-frontend-to-backend -o yaml
```

### Verify that the policy is active

```bash
kubectl describe networkpolicy allow-frontend-to-backend
```

Look for the `Selector` and `Policy types` sections to confirm the rule is parsed correctly.

### Test connectivity

```bash
# From a frontend pod shell
kubectl exec -it <frontend-pod> -- curl -s http://<backend-service>
```

### Use `tcpdump` or `cilium monitor`

```bash
# Cilium: see packets that are allowed/denied
cilium monitor --tcp -i 8080

# Calico: check policy stats
calctl policy stats
```

### Event-driven debugging

If a pod cannot receive traffic, check the underlying CNI plugin's events:

```bash
kubectl get events --field-selector involvedObject.name=<pod-name> --sort-by=.metadata.creationTimestamp
```

## Common Gotchas

| Issue | Symptom | Fix |
|-------|---------|-----|
| `policyTypes` missing | Policy has no effect (only `Ingress` or `Egress` is evaluated) | Always specify `policyTypes` explicitly |
| Too broad `podSelector` | Policy applies to all pods, unintentionally blocking or allowing traffic | Use precise `matchLabels` or `namespaceSelector` |
| Forgetting `except` on CIDR | IP range is too strict, legitimate traffic is dropped | Review CIDR blocks and use `except` for exceptions |
| Policy not updating after CNI restart | Changes seem ignored until the CNI agent re-loads | Restart the CNI daemon or the pod to pick up the new policy |

## Conclusion

Kubernetes Network Policies are the most direct way to reduce the cluster's attack surface and contain breaches. Start with a default-deny posture in each namespace, then add explicit allowlist policies for the traffic your workloads actually need. Use label-driven strategies so policies remain stable as pods scale up and down, and validate each policy with concrete connectivity tests before promoting it to production.

Combine Network Policies with OPA Gatekeeper (admission-time constraints) for a defense-in-depth posture: Gatekeeper prevents misconfigured pods from being admitted, and Network Policies restrict what those admitted pods can actually communicate with at runtime.

Start small — perhaps just a `deny-all-ingress-default-deny` in the `monitoring` namespace and an `allow-frontend-to-backend` policy — then iterate toward a full set of policies that match your threat model and compliance requirements.

*This article draws on common Kubernetes NetworkPolicy patterns and production cluster security experiences. Adjust label selectors, CIDR ranges, and policy ordering to match your cluster's CNI plugin, namespace structure, and security policies.*