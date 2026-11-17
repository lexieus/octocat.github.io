---
layout: post
title: "Prometheus Alerting: Practical Rules, Patterns, and Anti-Patterns"
date: 2026-11-17 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [prometheus, alerting, monitoring, devops, observability]
description: "A practical guide to writing effective Prometheus alerting rules, common patterns that work, and anti-patterns that cause alert fatigue in production environments."
---

# Prometheus Alerting: Practical Rules, Patterns, and Anti-Patterns

Prometheus is the de facto monitoring system for Kubernetes and cloud-native infrastructure. But writing alerting rules that are actionable, reliable, and don't cause alert fatigue is surprisingly difficult. This article covers practical rules you can implement today, patterns that solve real problems, and anti-patterns to avoid.

## Why Prometheus Alerting is Hard

Prometheus alerts are evaluated every `evaluation_interval` (default 1 minute). Unlike traditional monitoring systems that alert on static thresholds, Prometheus encourages expression-based alerting using [PromQL](https://prometheus.io/docs/prometheus/latest/querying/). The flexibility is powerful, but without discipline, it leads to:

- **Spamming** — alerts firing repeatedly without new information
- **Silencing** — teams mute alerts because they're never useful
- **Missing** — critical issues fly under the radar because rules are too complex

The goal is alerts that answer three questions: *What is wrong? Why does it matter? What should I do about it?*

## Quick Start: Your First Alerting Rule

Here's a minimal, sensible alert for high CPU usage:

```yaml
# alerts.yml
groups:
  - name: linux-node
    rules:
      - alert: NodeHighCPU
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Node {{ $labels.instance }} CPU usage high"
          description: |
            CPU usage is above 80% (current: {{ $value }}%).
            Check for long-running processes or consider scaling.
          runbook: "https://runbooks.example.com/node-high-cpu"
```

Key ingredients:

- `for: 5m` — avoids transient spikes, requires the condition to persist
- `labels.severity` — enables silencing, routing, and escalation
- `annotations.summary` / `annotations.description` — human-readable context
- `annotations.runbook` — link to troubleshooting guide

## Patterns That Work

### 1. Use `for` to Debounce Transient Spikes

Never alert on the raw instant vector. Always add a `for` duration to filter out glitches.

```yaml
alert: HighMemoryUsage
expr: node_memory_Active_Mem > 0.9 * node_memory_MemTotal
for: 2m
```

A 2-minute window ensures a brief memory surge doesn't trigger an alert, but sustained pressure still catches it.

### 2. Favor Rate/Count Over Raw Counters

When monitoring request rates or error counts, use `rate()` or `instant_rate()` to normalize per-second values.

```yaml
alert: RequestErrorRateHigh
expr: rate(http_requests_total{code=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
for: 3m
```

This alerts when **5% of requests** in the last 5 minutes are error responses, regardless of total traffic volume.

### 3. Anchor Alerts to Specific Instances or Pods

Without label scoping, a single alert can fire for every instance in your fleet — flooding your team.

```yaml
alert: PodRestartingFrequently
expr: (sum by (pod) (kube_pod_status_phase{phase="Running"} == 0) / sum by (pod) (kube_pod_restart_count[5m])) > 3
for: 5m
labels:
  severity: critical
annotations:
  summary: "Pod {{ $labels.pod }} restarting frequently on {{ $labels.instance }}"
```

The `by (pod)` ensures one alert per pod, not one per node × pod combination.

### 4. Group Related Alerts with `groups`

Keep your alerting rules organized by classifying them into groups. This lets you silence entire categories at once.

```yaml
groups:
  - name: k8s-cluster
    rules:
      - alert: NodeNotReady
        expr: node_status{phase!="Ready"} == 1
      - alert: KubeletHeartbeatMissing
        expr: absent(kubelet_seconds_since_last_heartbeat{> 300})
```

You can silence `k8s-cluster` as a group when performing maintenance.

## Anti-Patterns to Avoid

### ❌ Alerting on Raw Instant Values Without `for`

```yaml
# BAD: Fires on every evaluation if CPU > 80%, even if it spikes for 1 second
expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100) > 80
```

Always pair with `for: <duration>` to require persistence.

### ❌ Too Many Severity Levels

Limiting to 3 severity levels keeps silencing and escalation manageable:

| Level | When to Use |
|---|---|
| `critical` | Immediate action required — service outage |
| `warning` | Attention needed soon — degradation or boundary approaching |
| `info` | Observational — metrics of interest, no action required |

### ❌ Generic Annotations Without Context

```yaml
# BAD: "summary": "High CPU", "description": "CPU is high"
```

Instead, include actionable detail:

```yaml
# GOOD: "summary": "Node {{ $labels.instance }} CPU usage high"
"description": "CPU usage is {{ $value }}%. Check for long-running processes or consider scaling."
```

### ❌ Alerting Without a Runbook Link

Every alert should reference a runbook or troubleshooting guide. If you don't have one, the alert shouldn't exist yet.

```yaml
annotations:
  runbook: "https://internal.example.com/runbooks/node-cpu-high"
```

## Silencing Strategies

Even with good rules, you'll need to mute alerts during deployments or maintenance. Use label-based silencing:

| Silencing Scope | When to Use |
|---|---|
| **By severity** | Mute `info` during chaotic deployments |
| **By job/instance** | Mute specific nodes being updated |
| **By alert group** | Mute entire categories (e.g., `k8s-cluster`) |
| **Time-windowed** | Automatically expire after 30m / 2h |

Never use silencing as a crutch for bad rules. If an alert is noisy, fix the rule, not the silence.

## Example: Complete Alerting Suite for a Kubernetes Node

Here's a compact, production-ready set of rules you can drop into your `alerts.yml`:

```yaml
groups:
  - name: k8s-nodes
    rules:
      - alert: NodeHighCPU
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Node {{ $labels.instance }} CPU usage high"
          description: "CPU usage above 80% (current: {{ $value }}%). Check for resource leaks or consider scaling."
          runbook: "https://internal.example.com/runbooks/node-cpu"

      - alert: NodeHighMemoryUsage
        expr: (node_memory_MemTotal - node_memory_MemAvailable) / node_memory_MemTotal > 0.9
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "Node {{ $labels.instance }} memory usage high"
          description: "Memory usage above 90% (current: {{ $value }}%). Investigate OOM risks."
          runbook: "https://internal.example.com/runbooks/node-memory"

      - alert: NodeNotReady
        expr: node_status{phase!="Ready"} == 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Node {{ $labels.instance }} not ready"
          description: "Node has been in NotReady state for more than 1 minute."
          runbook: "https://internal.example.com/runbooks/node-not-ready"

      - alert: KubeletHeartbeatMissing
        expr: absent(kubelet_seconds_since_last_heartbeat{> 180})
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Kubelet heartbeat missing on {{ $labels.instance }}"
          description: "Kubelet hasn't sent a heartbeat in over 3 minutes."
          runbook: "https://internal.example.com/runbooks/kubelet-heartbeat"
```

## Conclusion

Prometheus alerting isn't about covering every metric — it's about surfacing the right information at the right time. Start with a few well-scoped rules, add `for` durations, label everything with severity, and link to runbooks. Iterate based on what your team actually needs to know, and prune alerts that generate more noise than signal.

The most effective alerting strategy is one that your team trusts enough to act on, and ignores when it's not relevant. That trust comes from discipline: consistent labels, annotated context, and regular review.

*Always test alerting rules in a staging environment first. Use `promtool alert check` to validate rules before deploying to production. Adjust thresholds and `for` durations to match your workload patterns and SLA requirements.*