---
layout: post
title: "Prometheus and Alertmanager: Practical Monitoring Patterns for Kubernetes Clusters"
date: 2026-10-29 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, prometheus, alertmanager, monitoring, observability, devops]
description: "A practical guide to setting up Prometheus and Alertmanager for Kubernetes monitoring, including service discovery, alert rules, templating, and real-world deployment patterns."
---

# Prometheus and Alertmanager: Practical Monitoring Patterns for Kubernetes Clusters

By default, Kubernetes applications run without built-in visibility into resource usage, request latency, or error rates. While `kubectl` and the Kubernetes dashboard provide operational metrics, production clusters require a dedicated monitoring stack. Prometheus + Alertmanager is the de facto standard for this purpose. This article walks through a production-ready setup, including service discovery, alert rules, notification templating, and common patterns for teams operating Kubernetes at scale.

## Why Prometheus + Alertmanager for Kubernetes?

Prometheus scrapes metrics from the Kubernetes API server, kubelet, and application endpoints via a pull-based model. Alertmanager handles routing, deduplication, and silencing of alerts. Together they provide:

- **Metric collection** for cluster and application health
- **Alert evaluation** against configured rules
- **Notification routing** to Slack, email, PagerDuty, or webhook endpoints
- **Alert lifecycle management** (inhibit, repeat, silence)

## 1. Deploying Prometheus with the Kubernetes Operator

The simplest way to get a functional Prometheus setup is using the Prometheus Operator, which manages CustomResource definitions for Prometheus, ServiceMonitor, and Alertmanager objects.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

After installation, the Prometheus UI is available at `http://<load-balancer-or-node-port>`.

## 2. Service Discovery: Finding Targets Automatically

Hardcoding scrape targets does not scale. Prometheus can discover targets automatically using Kubernetes labels and annotations.

### Native Kubernetes Service Discovery

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  labels:
    app: my-app
    prometheus: scrape
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: my-app
```

No additional configuration is needed — Prometheus discovers services with the `prometheus: scrape` label.

### Using `ServiceMonitor` CRDs

The Prometheus Operator extends Kubernetes with a `ServiceMonitor` resource that maps directly to scrape configurations.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  labels:
    prometheus: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: web
      interval: 15s
      scheme: http
```

Apply this into the `monitoring` namespace and Prometheus will automatically begin scraping `my-app:web` at the specified interval.

### Pod-Level Metrics via kubelet

Prometheus discovers kubelet metrics through the `kubernetes_sd_config` mechanism. When the Prometheus server runs inside the cluster, it can find kubelet endpoints via service discovery configured in the PrometheusSpec.

## 3. Writing Alert Rules

Alert rules define conditions that, when true, trigger notifications. Rules live in a `PrometheusRule` custom resource.

### Basic Alert: High Container Memory Usage

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: memory-usage-alerts
  labels:
    prometheus: kube-prometheus-stack
spec:
  groups:
  - name: cluster.rules
    interval: 30s
    rules:
    - alert: HighContainerMemoryUsage
      expr: (1 - (avg by (container) (rate(container_memory_working_set_bytes[5m])) * 100)) < 10
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "Container memory usage is above 90%"
        description: "Container {{ $labels.container }} on node {{ $labels.node }} has only {{ $value | humanize }} memory remaining. Consider scaling or optimizing resource requests."
```

**Key components:**

- `expr`: The Prometheus expression to evaluate. This rule triggers when available memory drops below 10%.
- `for`: How long the condition must persist before firing (avoids transient spikes).
- `labels.severity`: Categories alerts for routing.
- `annotations`: Human-readable messages displayed in Alertmanager and notification templates.

### Alert: Kubernetes Node Not Ready

```yaml
- alert: NodeNotReady
  expr: kube_node_status_condition{condition="Ready"} == 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Node {{ $labels.instance }} has been not ready for 5 minutes"
    description: "Node {{ $labels.instance }} reported condition Ready = false for more than 5 minutes. Check kubelet and network connectivity."
```

### Alert: High API Server Latency

```yaml
- alert: HighAPIServerLatency
  expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) > 0.5
  for: 1m
  labels:
    severity: warning
  annotations:
    summary: "95th-percentile API server latency is above 500ms"
    description: "The 95th-percentile request latency across the Kubernetes API is {{ $value | humanize }}. Investigate workloads or cluster capacity."
```

## 4. Alertmanager Configuration

Alertmanager receives evaluated alerts from Prometheus and routes them based on routing rules, receiver configurations, and inhibition rules.

### Basic Alertmanager CR

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Alertmanager
metadata:
  name: alertmanager
  labels:
    prometheus: kube-prometheus-stack
spec:
  configuration:
    global:
      resolve_timeout: 5m
    route:
      group_by: ['alertname', 'cluster']
      group_wait: 30s
      repeat_interval: 4h
      receiver: 'web.hook'
      routes:
      - match:
          severity: critical
        receiver: 'pagerduty'
      - match:
          severity: warning
        receiver: 'slack'
    inhibit_rules:
      - source_match:
          severity: critical
        target_match:
          severity: warning
        equal: ['alertname']
```

**Routing logic:**

- `group_by`: Groups alerts with the same labels together.
- `group_wait`: Delays the group notification to avoid alert flooding.
- `repeat_interval`: Re-sends a resolved alert after this interval if still active.
- `routes`: Directs alerts to different receivers based on labels (e.g., `severity: critical` → PagerDuty, `severity: warning` → Slack).
- `inhibit_rules`: Suppresses less-severe alerts when a more-severe alert is active (e.g., a `NodeNotReady` critical alert inhibits multiple `HighMemoryUsage` warnings on the same node).

### Receiver Configurations

#### Slack Receiver

```yaml
receivers:
  - name: 'slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXX'
        channel: '#k8s-alerts'
        send_resolved: true
        title: "{{ .CommonLabels.alertname }}: {{ .CommonAnnotations.summary }}"
        text: |
          *Severity:* {{ .Labels.severity }}
          *Description:* {{ .Annotations.description }}
```

#### PagerDuty Receiver

```yaml
receivers:
  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'PAGERDUTY_INTEGRATION_KEY'
        severity_mapping:
          critical: 'critical'
          warning: 'warning'
          info: 'info'
```

#### Webhook Receiver (custom integration)

```yaml
receivers:
  - name: 'web.hook'
    webhook_configs:
      - url: 'https://alerts.example.com/webhook'
        send_resolved: true
        http_config:
          follow_redirects: false
          tls_config:
            insecure_skip_verify: false
```

## 5. Templating and Notification Variables

Alertmanager supports a rich templating language based on Go's `text/template`. The `.` context contains:

- `.CommonLabels`: Labels from the original alert (e.g., `alertname`, `severity`, `instance`).
- `.CommonAnnotations`: Annotations from the original alert (e.g., `summary`, `description`).
- `.Alerts`: A list of individual alert objects, each with `.Labels`, `.Annotations`, `.StartsAt`, `.EndsAt`, and `.GeneratorURL`.

### Example: Multi-Line Slack Message

```yaml
route:
  receiver: 'slack'
  group_by: ['alertname', 'severity']
  group_wait: 10s
  group_interval: 10m
  repeat_interval: 1h

receivers:
  - name: 'slack'
    slack_configs:
      - channel: '#k8s-alerts'
        send_resolved: true
        title: "{{ .CommonLabels.alertname }} — {{ .CommonLabels.severity }}"
        text: |
          *Alert:* {{ .CommonAnnotations.summary }}
          *Details:* {{ .CommonAnnotations.description }}
          *Status:* {{ if eq .Status "firing" }}🔴 firing{{ else }}🟢 resolved{{ end }}
          *Duration:* {{ .StartsAt | date "2006-01-02 15:04:05" }}
          *More info:* {{ .GeneratorURL }}
```

## 6. Practical Deployment Pattern

### Step 1: Enable Monitoring on All Namespaces

By default, Prometheus only scrapes namespaces it is configured to watch. Add the `prometheus: scrape` label to namespaces that should be monitored, or use the `NamespaceMonitor` CRD to select all namespaces.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: NamespaceMonitor
metadata:
  name: monitor-all
  labels:
    prometheus: kube-prometheus-stack
spec:
  selector:
    matchNames:
    - production
    - staging
    - development
```

### Step 2: Add Resource Limits

Prevent Prometheus from consuming all cluster resources:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: monitoring-resources
  namespace: monitoring
spec:
  hard:
    cpu: "2"
    memory: "2Gi"
    pods: "5"
```

### Step 3: Verify Scrape Targets

```bash
# List targets in Prometheus UI at /targets
# Or query via Prometheus API
curl http://<prometheus-service>:9090/api/v1/targets
```

### Step 4: Test Alert Routing

Fire a test alert and verify routing:

```bash
# Create a test PrometheusRule that triggers immediately
cat > test-alert.yaml <<'EOF'
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: test-alert
  labels:
    prometheus: kube-prometheus-stack
spec:
  groups:
  - name: test.rules
    rules:
    - alert: TestAlert
      expr: 1 == 1
      for: 0s
      labels:
        severity: critical
      annotations:
        summary: "This is a test alert"
        description: "Verify Alertmanager routing is working"
EOF
kubectl apply -f test-alert.yaml
```

Check the Alertmanager UI or Slack channel for the delivered alert.

### Step 5: Clean Up Test Resources

```bash
kubectl delete prometheusrule test-alert
```

## Common Patterns and Gotchas

| Pattern | Recommendation |
|---------|----------------|
| **Start with audit mode** | Before enforcing blocking alerts, set `alertmanagers` `route` `receiver` to a test endpoint or use `alertmanager` `receivers` with `email` configs to validate alert shapes without notifying teams. |
| **Use namespace selectors strategically** | A `ServiceMonitor` without a `namespaceSelector` applies cluster-wide. Start with a narrow selector (e.g., `monitoring` namespace) and expand as needed. |
| **Parameterize alert thresholds** | Use Prometheus `flags` or `values` from a ConfigMap to make thresholds configurable without re-deploying rules. |
| **Combine with Node Exporter** | Ensure `kube-state-metrics` and `node-exporter` are deployed for full cluster visibility (CPU, memory, disk, network). |
| **Avoid alert fatigue** | Use `inhibit_rules` to suppress redundant alerts, `group_by` to cluster related alerts, and `repeat_interval` to limit notification frequency. |
| **Version alert rules** | Store `PrometheusRule` YAML in Git alongside application manifests. Treat rule changes with the same review process as deployment changes. |

## Conclusion

Prometheus + Alertmanager provides the foundational monitoring and alerting layer for Kubernetes clusters. By leveraging the Prometheus Operator's CRDs (`ServiceMonitor`, `PrometheusRule`, `Alertmanager`), teams can declaratively configure metric collection, alert evaluation, and notification routing without manually managing scrape configurations. Start with a basic setup — deploy the operator, add a few `ServiceMonitor` objects, write a couple of critical alert rules, and configure a Slack receiver — then iterate toward a full observability platform that includes Grafana dashboards, tracing, and log aggregation.

*This article is intended for operational reference. Always test alert rules in a non-critical namespace first and verify that legitimate workloads are not blocked before enforcing cluster-wide policies. Adjust thresholds and routing to match your organization's SLOs and incident response practices.*

---
*This article draws on common Prometheus and Alertmanager patterns and production monitoring experiences. Adjust scrape configurations, alert thresholds, and notification integrations to match your cluster topology, application ecosystems, and on-call rotation policies.*