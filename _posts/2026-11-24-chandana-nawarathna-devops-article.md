---
layout: post
title: "Prometheus and Alertmanager: Practical Monitoring and Alerting for Kubernetes"
date: 2026-11-24 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, prometheus, alertmanager, monitoring, devops, observability]
description: "Practical guide to setting up Prometheus and Alertmanager for Kubernetes monitoring, including service discovery, recording rules, alert rules, and safe alert routing patterns."
---

# Prometheus and Alertmanager: Practical Monitoring and Alerting for Kubernetes

Modern Kubernetes clusters require observability to detect issues before they impact users. Prometheus is the de facto standard for cluster and workload metrics, and Alertmanager handles routing, silencing, and inhibition of alerts. This article walks through a production-ready Prometheus setup and practical alerting patterns you can adopt today.

## Why Prometheus for Kubernetes?

Prometheus scrapes metrics from exposed endpoints and stores them as time-series data. In Kubernetes, it can discover targets dynamically via labels, namespace annotations, and service discovery. Unlike hosted solutions, Prometheus runs inside your cluster, giving you full control over data retention, rule evaluation, and alert routing.

## Deploying Prometheus with the Operator

The easiest way to get a production-grade Prometheus is via the [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator). It manages CustomResourceDefinitions (CRDs) for `Prometheus`, `Alertmanager`, and `ServiceMonitor`.

### Install the Operator

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update prometheus-community
helm install prometheus-prometheus-community prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

This installs Prometheus, Alertmanager, the Node Exporter, and kube-state-metrics in the `monitoring` namespace.

### Verify the installation

```bash
kubectl get pods -n monitoring
```

You should see the prometheus, alertmanager, and related pods running.

## Configuring Service Discovery

By default, Prometheus does not know about Kubernetes services. The Prometheus Operator uses `ServiceMonitor` resources to discover and scrape metrics.

### Basic ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-monitor
  namespace: default
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: web
      path: /metrics
      interval: 15s
```

Apply it:

```bash
kubectl apply -f servicemonitor.yaml
```

The Prometheus server will automatically begin scraping any Service with the label `app: my-app` on port `web` at `/metrics`.

### Cluster-wide monitoring with Node Exporter

Node Exporter runs on each node and exposes hardware- and OS-level metrics (CPU, memory, disk, network). The Prometheus Operator deploys it via a DaemonSet.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  serviceMonitorSelector:
    matchLabels:
      addonmanager: node-exporter
  alertmanager:
    route:
      receiver: 'alertmanager.main'
    inhibit_rules:
      - source_matchers:
          - severity: 'critical'
        target_matchers:
          - severity: 'warning'
        equal: ['alertname', 'dev', 'instance']
```

## Recording Rules for Fast Queries

Recording rules pre-compute expensive PromQL queries and store the result as a new time-series. This reduces query latency and evaluation cost.

### Example: CPU availability per node

```rego
# In a RecordingRule CRD
groups:
- name: node-cpu-rules
  interval: 15s
  rules:
  - record: node:cpu:availability
    expr: 1 - (rate(node_cpu_seconds_total{mode="idle"}[1m]) * 100)
```

This records the CPU availability (used percentage) per node. You can then query `node:cpu:availability` instantly instead of running the full rate expression every time.

## Alert Rules and Alertmanager Routing

Alert rules define when to fire alerts based on evaluated PromQL expressions. Alertmanager receives these alerts and routes them to receivers (email, Slack, PagerDuty, etc.) based on labels and severity.

### Example: High pod restart rate

```yaml
groups:
- name: k8s-alerts
  rules:
  - alert: HighPodRestartRate
    expr: rate(kube_pod_container_status_restarts_total[5m]) > 0.5
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "Pod restart rate is high on {{ $labels.instance }}"
      description: |
        The container {{ $labels.container }} on node {{ $labels.instance }} has restarted
        {{ $value }} times per 5 minutes. Check for crash loops or OOM kills.
```

### Alertmanager configuration

```yaml
receivers:
  - name: 'alertmanager.main'
    email_configs:
      to: 'oncall@example.com'
      from: 'alertmanager@example.com'
    send_resolved: true

routes:
  - match:
      severity: critical
    receiver: 'alertmanager.main'
    routes:
      - match:
          alertname: HighPodRestartRate
        receiver: 'slack.warning'

inhibit_rules:
  - source_matchers:
      - severity: 'critical'
    target_matchers:
      - severity: 'warning'
    equal: ['alertname', 'dev']
```

Apply the configuration by mounting it as a ConfigMap or via the Alertmanager CRD.

## Safe Alert Routing Patterns

- **Avoid alert fatigue**: Only fire alerts for conditions that last `for:` a minimum duration (e.g., 2m, 5m). Transient spikes should not trigger pages.
- **Use severity labels**: Tag alerts with `severity: critical`, `warning`, `info` so Alertmanager can prioritize routing.
- **Inhibit duplicate alerts**: If a critical alert is already firing, inhibit lower-severity alerts with the same `alertname` to avoid noise.
- **Silence noisy alerts**: Use Alertmanager's UI or API to silence alerts during maintenance windows.
- **Send resolved notifications**: Set `send_resolved: true` in receivers so you know when a condition has cleared.

## Example: Full Monitoring Stack in Minikube

If you're testing locally, you can use the `kube-prometheus-stack` chart with Minikube:

```bash
minikube start --cpus 4 --memory 8192
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install my-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

Then access:

```bash
minikube service prometheus -n monitoring --url
# => http://192.168.49.2:30090
```

## Conclusion

Prometheus + Alertmanager gives you a complete monitoring and alerting loop for Kubernetes: scrape metrics → evaluate rules → fire alerts → route to the right people → clear when resolved. Start with a few high-impact alert rules (pod restart rate, node disk pressure, high CPU usage) and expand from there. Pair this with centralized logging (Fluent Bit → Loki) and you have a complete observability stack.

*Always test alert rules in a non-critical namespace first. The examples above use read-only PromQL expressions and safe annotation patterns. Adjust thresholds, `for:` durations, and receiver configurations to match your SLOs and on-call rotation.*