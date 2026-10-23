---
layout: post
title: "Kubernetes Cluster Health Monitoring and Automated Remediation"
date: 2026-10-21 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, monitoring, automation, devops, health-checks]
description: "Practical scripts and patterns for monitoring Kubernetes cluster health and automatically remediating common issues."
---

# Kubernetes Cluster Health Monitoring and Automated Remediation

Operating Kubernetes clusters in production requires more than deploying workloads — you need continuous visibility into cluster health and the ability to automatically respond to common problems before they impact users. This article presents practical scripts and patterns for monitoring cluster health and executing safe, automatic remediation.

## Why Health Monitoring Matters

Kubernetes clusters consist of many interacting components — kubelet, container runtime, API server, etcd, and workloads. Issues such as node pressure, crashing pods, etcd latency, and API server throttling can degrade performance or cause outages. Proactive monitoring and automated remediation reduce mean time to recovery (MTTR) and free operators from repetitive manual fixes.

## 1. Node Resource Pressure Detection

Node-level resource pressure is one of the most common causes of pod scheduling failures and evictions. The following Bash script checks for the four pressure conditions defined by Kubernetes and reports their status.

```bash
#!/usr/bin/env bash
set -euo pipefail

PRESSURE_TYPES=("Memory" "DiskPressure" "NetworkUnavailable" "PIDPressure")

for pt in "${PRESSURE_TYPES[@]}"; do
  # kubectl alpha node alpha reports pressure; kubectl describe node also shows it
  pressure=$(kubectl describe node "$(hostname)" 2>/dev/null | grep -E "Conditions:|${pt}" || echo "unknown")
  echo "$pt: $pressure"
done
```

**Running the script**

```bash
$ chmod +kube-health-check.sh && ./kube-health-check.sh
Memory: MemoryPressure false
DiskPressure: false
NetworkUnavailable: false
PIDPressure: false
```

If `MemoryPressure` or `DiskPressure` is `true`, the script can trigger further investigation or automatic cleanup (see the remediation section below).

## 2. Pod Crashing Loop Detection

Pods that repeatedly crash can indicate application errors, misconfigured resources, or infrastructure limits. This script watches a namespace for pods in `CrashLoopBackOff` and prints their details.

```bash
#!/usr/bin/env bash
set -euo pipefail

NAMESPACE="${1:-default}"

echo "Watching for CrashLoopBackOff pods in namespace: $NAMESPACE"
kubectl get pods -n "$NAMESPACE" -o json | \
  jq -r '.items[] | select(.status.reason == "CrashLoopBackOff") |
  "\(.metadata.name): \(.status.message // "no message") / \(.status.startTime // "N/A")"' 2>/dev/null || \
  kubectl get pods -n "$NAMESPACE" -o wide | grep -i "crash"
```

**Example output**

```
Watching for CrashLoopBackOff pods in namespace: production
web-app-7x8b2: Image pull backoff / 2026-10-20T14:23:00Z
worker-2mq44: Insufficient cpu / 2026-10-20T14:25:00Z
```

Pods with `CrashLoopBackOff` should be investigated by checking container logs, resource requests/limits, and image availability.

## 3. Automated Node Pressure Remediation

When a node experiences `MemoryPressure`, the kernel may evade pods, but proactive cleanup can prevent reaching that state. The following script safely removes terminated pods and prunes unused containers to free memory.

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "Cleaning up terminated pods and unused containers on $(hostname)"
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.status.phase == "Succeeded" or .status.phase == "Failed") |
  .metadata.name' | \
  xargs -r -I {} kubectl delete pod {} --ignore-not-found=true 2>/dev/null

# Prune exited containers via crictl (requires container runtime access)
if command -v crictl >/dev/null 2>&1; then
  crictl rmp --filters '{"status":[" exited"]}' 2>/dev/null || true
fi

echo "Remediation complete."
```

**Important:** This script assumes the runner has appropriate RBAC and container runtime access. Always test in a staging environment before running in production.

## 4. etcd Latency and Health Checks

etcd latency spikes can cause API server delays and pod scheduling issues. A simple health check using `etcdctl`:

```bash
#!/usr/bin/env bash
set -euo pipefail

ETCD_ENDPOINT="${ETCD_ENDPOINT:-http://127.0.0.1:2379}"

echo "Checking etcd health..."
etcdctl endpoint health --endpoints="$ETCD_ENDPOINT"

echo "Checking etcd statistics..."
etcdctl endpoint stats --endpoints="$ETCD_ENDPOINT" \
  | grep -E "rtt|op" \
  || echo "etcdctl stats summary unavailable, check connectivity"
```

**Typical healthy output**

```
$ ./etcd-health.sh
 endpoints: http://127.0.0.1:2379, status: healthy
 rtt: min=0.344ms max=1.289ms avg=0.692ms
```

If `rtt` values climb above 5ms consistently, investigate network latency, disk I/O, or etcd member health.

## 5. Summary: Integrating into Observability Pipelines

These scripts are designed to be called from monitoring systems such as Prometheus alertmanager receivers, cron-based health checks, or CI/CD pipelines. Consider the following patterns:

- **Prometheus alert**: Trigger the remediation script via a custom alertmanager receiver.
- **CronJob**: Run the node pressure check every 5 minutes and page on detection.
- **GitOps**: Store scripts in a dedicated `cluster-monitoring` directory and apply them as part of cluster bootstrap.

No single script replaces a full observability stack, but these patterns provide immediate, actionable feedback for common Kubernetes operational scenarios.

## References

- Kubernetes Node Reference: <https://kubernetes.io/docs/reference/configuration-api/node/>
- etcdctl documentation: <https://github.com/etcd-io/etcd/tree/main/etcdctl>
- Prometheus client_go example: <https://github.com/prometheus/client_golang>

---
*This article is intended for educational and operational reference. Always test scripts in a non-production environment and verify RBAC permissions before running remediation commands in clusters running workloads.*
