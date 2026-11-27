---
layout: post
title: "Pod Disruption Budgets: Guaranteeing Availability for Kubernetes Workloads During Updates"
date: 2026-11-27 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, devops, availability, pdb, disruption-budget]
description: "A practical guide to defining and using Pod Disruption Budgets to protect workloads during voluntary evictions, node upgrades, and self-healing operations."
---

# Pod Disruption Budgets: Guaranteeing Availability for Kubernetes Workloads During Updates

When you roll out a new version of a Deployment or upgrade a node cluster, Kubernetes may need to terminate pods to make room for new ones or to drain nodes. Without a Pod Disruption Budget (PDB), a workload could be scaled down to zero instantly, causing an outage. PDBs express a minimum number or percentage of pods that must remain available at all times.

## Why PDBs Matter

- **Voluntary evictions**: `kubectl drain`, node upgrades, cluster autoscaler actions.
- **Self-healing**: When a pod crashes, the controller recreates it; a PDB ensures at least some pods stay up during the transition.
- **Predictable rollouts**: Guarantees a minimum subset of your service remains reachable during progressive updates.

## PDB Types

- **`minAvailable`**: Ensures at least the specified number or percentage of pods are available at all times. This is the most common choice for stateless services.
- **`maxUnavailable`**: Limits the number or percentage of pods that can be unavailable at once. Useful for stateful workloads where you want to cap the impact of a disruption.

## Example: Adding a PDB to a Deployment

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
  namespace: default
spec:
  minAvailable: 70%
  selector:
    matchLabels:
      app: my-app
```

Apply it:

```bash
kubectl apply -f my-app-pdb.yaml
```

## Verifying the PDB

```bash
# Show PDBs in the namespace
kubectl get poddisruptionbudgets -n default

# Describe a specific PDB
kubectl describe poddisruptionbudget my-app-pdb -n default
```

## Interacting with the PDB During Drain

```bash
# Drain a node while respecting the PDB
kubectl drain node-name \
  --ignore-daemonsets \
  --pod-selector="app=my-app" \
  --delete-emptydir-data
```

If the PDB would be violated, `kubectl drain` will wait (or fail, depending on `--force`) until enough pods are rescheduled elsewhere.

## Best Practices

1. **Set `minAvailable` to a percentage** that reflects your availability goals (e.g., 70% for a 3‑replica deployment gives you at least two pods running at all times).
2. **Avoid `maxUnavailable` for highly available services**; use `minAvailable` instead.
3. **Match the PDB selector to the Deployment's label template** so the PDB tracks the correct pods.
4. **Review PDBs when scaling workloads up or down**; adjust `minAvailable` if you change replica counts.
5. **Combine PDBs with Health Checks** (liveness/readiness) so the controller knows when a pod is actually ready to serve traffic.

## Conclusion

Pod Disruption Budgets are a simple, declarative way to add safety to voluntary pod evictions. By specifying a minimum number of available pods, you prevent accidental outages during node upgrades, cluster scaling, and rolling updates. Start with `minAvailable: 50%` (or a conservative percentage) for each of your Deployments, then tune based on your tolerance for reduced availability.

*Always test PDB behavior in a staging namespace before applying to production. The examples above use read‑only inspection and safe‑to‑apply patterns. Avoid `kubectl drain --force` unless you intend to bypass safety guarantees.*