---
layout: post
title: "Prometheus Alerting Rules: Writing Practical Alerts for Kubernetes"
date: 2026-11-28 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [prometheus, kubernetes, monitoring, alerting, devops, observability]
description: "A practical guide to writing effective Prometheus alerting rules for Kubernetes workloads, including rule structure, severity levels, common patterns, and gotchas to avoid."
---

# Prometheus Alerting Rules: Writing Practical Alerts for Kubernetes

Prometheus is the de facto monitoring system for Kubernetes, but writing alert rules that fire reliably — without noisy false positives or missed detections — is a skill that takes practice. This article walks through the structure of alerting rules, common patterns for Kubernetes workloads, and practical tips for keeping alerts actionable.

## Rule Structure

A Prometheus alerting rule is defined in a `Alertmanager` rule group and consists of a PromQL query, a condition, and a label set. The minimal rule looks like:

```yaml
groups:
- name: kubernetes.rules
  rules:
  - alert: HighCpuUsage
    expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High CPU usage on {{ $labels.instance }}"
      description: "CPU usage has been above 80% for 5 minutes. Current: {{ $value }}%"
```

Key components:

| Component | Purpose |
|---|---|
| `expr` | PromQL expression that evaluates to `1` (true) when the alert should fire |
| `for` | How long the expression must be true before the alert fires; prevents transient spikes from triggering |
| `labels` | Severity, team, or other custom labels for routing in Alertmanager |
| `annotations` | Human-readable messages; `$value` and `$labels` are templated at evaluation time |

## Common Kubernetes Alerting Patterns

### 1. Pod Status: Pending or CrashLoopBackOff

```yaml
- alert: PodCrashLoopBackOff
  expr: kube_pod_status_phase{phase="Pending"} > 0 or kube_pod_status_phase{phase="CrashLoopBackOff"} > 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Pod {{ $labels.pod }} in namespace {{ $labels.namespace }} is crashing"
    description: "The pod has been in CrashLoopBackOff for 2 minutes. Check container logs with: kubectl logs {{ $labels.pod }} -n {{ $labels.namespace }}"
```

### 2. Service Endpoint Health

```yaml
- alert: ServiceNoEndpoints
  expr: kube_endpoint_port_status{job="kubernetes",port="http"} == 0
  for: 1m
  labels:
    severity: warning
  annotations:
    summary: "Service {{ $labels.service }} has no healthy endpoints"
    description: "No endpoints are reporting healthy status for service {{ $labels.service }} in namespace {{ $labels.namespace }}"
```

### 3. High Memory Usage on Nodes

```yaml
- alert: NodeMemoryPressure
  expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Node {{ $labels.instance }} is using more than 85% memory"
    description: "Available memory is low. Free: {{ $labels.instance }}: {{ printf \"%.2f\" (100 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100) }}% used"
```

### 4. Replication Controller Desired vs Ready

```yaml
- alert: DeploymentReplicasMismatch
  expr: sum by (namespace, deployment) (kube_deployment_status_replicas) != sum by (namespace, deployment) (kube_deployment_status_replicas_ready)
  for: 3m
  labels:
    severity: warning
  annotations:
    summary: "Deployment replica mismatch in {{ $labels.namespace }}"
    description: "Desired: {{ $labels.deployment }}, Ready: {{ $labels.deployment }}. Check the deployment events with: kubectl -n {{ $labels.namespace }} describe deployment {{ $labels.deployment }}"
```

## Gotchas and Best Practices

### 1. Always Use a `for` Duration

Never fire an alert immediately when a condition becomes true. Transient spikes, short-lived pods, and auto-recovering scenarios should not generate notifications. Start with `for: 1m` or `for: 5m` and tune based on your workload's dynamics.

### 2. Prefer `rate()` and `irate()` Over Raw Counters

Raw counters like `kube_pod_container_status_restarts_total` will keep increasing and never reset, making them poor triggers. Use `rate()` for per-second averages or `irate()` for instant rates:

```yaml
# Good: restarts per minute
rate(kube_pod_container_status_restarts_total[1m]) > 5

# Avoid: absolute count with no time component
kube_pod_container_status_restarts_total > 10
```

### 3. Scope Queries to Relevant Labels Only

Avoid cluster-wide rules when you only need namespace-specific alerts. Use label selectors in your expressions:

```yaml
# Instead of: sum(rate(node_cpu_seconds_total[5m])) > 80
# Use: rate(node_cpu_seconds_total{mode!="idle"}[5m]) / rate(node_cpu_seconds_total[5m]) * 100 > 80
```

The first expression fires if *any* node is over 80% CPU, making it hard to identify the culprit. The second computes average CPU usage per node, letting you pinpoint which node is problematic.

### 4. Test Rules Before Deploying

Use `promtool` to validate alerting rules:

```bash
promtool check rules alert-rules.yaml
```

This checks for syntax errors, missing labels, and other common issues before the rules reach Alertmanager.

### 5. Group Alerts by Severity and Team

Organize rules into groups with meaningful labels so Alertmanager can route notifications correctly:

```yaml
groups:
- name: "team-infra.rules"
  rules:
  - alert: NodeDown
    expr: up{job="node-exporter"} == 0
    for: 2m
    labels:
      severity: critical
      team: infra
    annotations:
      summary: "Node {{ $labels.instance }} is down"
      description: "The node {{ $labels.instance }} has been unreachable for 2 minutes. Check infrastructure and connectivity."
```

## Conclusion

Writing Prometheus alert rules is an iterative process. Start with a few high-impact rules (node down, high CPU, pod crashing), test them, and expand coverage over time. The key principles are: always use a `for` duration, prefer rate-based expressions, scope labels narrowly, and validate with `promtool`.

*Always test new alert rules in a staging namespace or with a delayed `for` duration before promoting to production alerting. Misconfigured rules can flood notification channels and cause alert fatigue.*

---
*This article draws on common Prometheus and Kubernetes monitoring patterns. Adjust thresholds and expressions to match your cluster's SLOs, resource profiles, and notification preferences. For distribution-specific guidance, consult the Prometheus documentation and your cluster's monitoring setup.*