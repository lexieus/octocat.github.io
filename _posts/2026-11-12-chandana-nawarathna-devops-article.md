---
layout: post
title: "Prometheus Monitoring: A Practical Setup Guide for Kubernetes Clusters"
date: 2026-11-12 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [prometheus, kubernetes, monitoring, alertmanager, devops, observability]
description: "A practical, step-by-step guide to deploying Prometheus and Alertmanager for Kubernetes monitoring, including service discovery, scraping configuration, and alert routing."
---

# Prometheus Monitoring: A Practical Setup Guide for Kubernetes Clusters

Monitoring is a core observability pillar, but getting a production-grade Prometheus deployment running in Kubernetes without unnecessary complexity is a common pain point. This article walks through a minimal-but-usable Prometheus setup using the [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator), covering service discovery, scraping targets, and alert routing with Alertmanager.

## Why Prometheus in Kubernetes?

Prometheus is the de facto monitoring system for Kubernetes workloads. Its strengths include:

- **Pull-based scraping** via HTTP, native service discovery against the Kubernetes API server.
- **Multi-dimensional data model** — metrics are identified by metric name and key/value label pairs.
- **Alertmanager integration** — deduplication, grouping, and routing of alerts to multiple receivers.
- **Kubernetes-native** — most cluster metrics are already exposed via cAdvisor, kube-state-metrics, and the Node Exporter.

## Deploying the Prometheus Operator

The easiest way to get a fully-configured Prometheus stack is the Prometheus Operator, which customizes Kubernetes resources to manage Prometheus, Alertmanager, and ServiceMonitors.

### Install via Helm

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus-prometheus-community prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

This chart installs Prometheus, Alertmanager, the Node Exporter, kube-state-metrics, and a Grafana instance in one step.

### Verify the installation

```bash
kubectl get pods -n monitoring
```

You should see pods like `prometheus-prometheus-node-exporter`, `prometheus-kube-state-metrics`, and `prometheus-alertmanager` running.

## Scraping User-Defined Workloads

By default, the Prometheus Operator scrapes system components (kubelet, API server, etc.). To scrape application metrics, you create a `ServiceMonitor` resource that tells Prometheus which endpoints to poll.

### Example: Exposing metrics from a Go service

Assuming your application already exposes a `/metrics` endpoint on port 9090:

1. **Annotate the Service** so the Operator can discover it:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  labels:
    app: my-app
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
spec:
  selector:
    app: my-app
  ports:
    - name: metrics
      port: 9090
      targetPort: 9090
      protocol: TCP
```

2. **Create a ServiceMonitor** (optional, if you want explicit control):

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics
      path: /metrics
      interval: 15s
  namespaceSelector:
    matchNames:
      - default
```

After applying the ServiceMonitor, Prometheus will begin scraping `my-app:9090/metrics` every 15 seconds. Verify in the Prometheus UI under *Targets*.

## Alertmanager: Routing and Deduplication

Scraping metrics is only half the story — you need Alertmanager to turn metric thresholds into actionable alerts.

### Basic Alertmanager configuration

Create a ConfigMap with alert routing rules:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml:
    global:
      resolve_timeout: 5m
    route:
      receiver: 'team-devops'
      group_by: ['alertname', ' severity']
      group_wait: 30s
      repeat_interval: 4h
      routes:
        - match:
            severity: critical
          receiver: 'team-oncall'
          repeat_interval: 1h
    receivers:
    - name: 'team-devops'
      emailConfigs:
        - to: 'devops@example.com'
    - name: 'team-oncall'
      emailConfigs:
        - to: 'oncall@example.com'
```

Apply it:

```bash
kubectl apply -f alertmanager-config.yaml
```

Update the Prometheus chart values to use the new ConfigMap, or mount it into the Alertmanager pod if deploying manually.

### Testing the alert pipeline

```bash
# Create a test alert via kubectl (requires Alertmanager installed)
kubectl -n monitoring patch alertmanager prometheus-alertmanager \
  -p '{"spec":{"route":{"receiver":"team-oncall"}}}'
```

Then trigger a sample alert from Prometheus using the CLI:

```bash
promtool alertmanager add-alert \
  --alertname='TestCritical' \
  --severity=critical \
  --alertmsg='This is a test alert'
```

## Visualizing with Grafana

The `kube-prometheus-stack` Helm chart ships with pre-built Grafana dashboards for Kubernetes resources, node resources, and kube-system workloads. After logging in (default admin / prom-operator), import additional dashboards from the [Grafana Hub](https://grafana.com/grafana/dashboards) for application-specific metrics (e.g., Go runtime metrics, PostgreSQL exporter).

## Best Practices

1. **Use the Prometheus Operator** rather than manually managing Prometheus instances — it handles resource creation, scaling, and configuration via custom resources.
2. **Limit scrape targets per Prometheus instance** — if you have hundreds of services, consider sharding across multiple Prometheus servers or using Thanos for long-term storage.
3. **Add `scrape_configs` sparingly** — prefer `ServiceMonitor` resources; they are namespaced and version-controlled.
4. **Set `metric_relabel_configs` carefully** — avoid dropping essential labels (like `namespace` or `pod`) that you may need for alert routing or dashboard filtering.
5. **Enable Alertmanager silence rules** via the UI or API for known maintenance windows, rather than disabling alerts globally.

## Conclusion

A working Prometheus + Alertmanager + Grafana stack is the foundation of Kubernetes observability. By leveraging the Prometheus Operator and ServiceMonitors, you can keep the configuration declarative, namespaced, and under version control. Start with the Helm chart, add ServiceMonitors for your workloads, and iterate alert rules as your incident response practice matures.

*Always test alert routing in a staging cluster before promoting to production. The commands shown are safe for observation and configuration; avoid changing production alert receivers without reviewing the impact on on-call schedules.*

---