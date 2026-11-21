---
layout: post
title: "OPA Gatekeeper: Enforcing Kubernetes Security Policies"
date: 2026-11-21 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, opa, gatekeeper, security, policy-as-code, compliance]
description: "Practical guide to installing and using OPA Gatekeeper for Kubernetes security policy enforcement, with real-world constraint templates and admission control examples."
---

# OPA Gatekeeper: Enforcing Kubernetes Security Policies

Kubernetes clusters are highly flexible, but that flexibility can lead to insecure configurations creeping into production. Resource quotas limit consumption, but they don't validate *how* resources are shaped — you need policy-as-code for that. OPA Gatekeeper extends the Kubernetes admission control layer to enforce custom constraints at create or update time.

This guide walks through installing Gatekeeper, writing constraint templates, and applying practical policies you can adopt today.

## Why Gatekeeper?

Standard Kubernetes admission controllers (BuiltIn) cover a fixed set of checks. Gatekeeper lets you define:

- **Pod security standards** — mandatory labels, prohibited capabilities, allowed runtime contexts
- **Resource constraints** — enforce CPU/memory requests/limits across namespaces
- **Label and annotation requirements** — every workload must carry a cost-center or environment label
- **Namespace lifecycle rules** — prevent accidental deployment to production without review

Gatekeeper policies are compiled into OPA Rego expressions and validated by the AdmissionReview webhook. If a manifest violates a policy, the API server rejects it with a descriptive error.

## Installing Gatekeeper

The easiest way to get started is via the Gatekeeper Operator or the Helm chart:

```bash
# Using the Gatekeeper Helm chart (kubectl apply)
helm repo add gatekeeper https://gatekeeper.sh/charts
helm repo update gatekeeper
helm install gatekeeper gatekeeper/gatekeeper --namespace gatekeeper-system --create-namespace
```

Or with the operator:

```bash
kubectl apply -f https://github.com/open-policy-agent/gatekeeper/releases/latest/download/gatekeeper.yaml
```

Verify the installation:

```bash
kubectl get pods -n gatekeeper-system
```

You should see the controller-manager and audit pods running. Gatehook automatically registers as an admission webhook; no additional configuration is needed to activate it.

## Writing Your First Constraint Template

A **ConstraintTemplate** defines the Rego logic. It declares which API types it applies to and exposes parameters the Constraint will bind.

Here's a template that ensures every Pod carries a `environment` label:

```rego
package k8s.admission.label

import data.k8s.admission.label as api

default msg = "Pod must have an environment label"

withoperation = operation == "CREATE" or operation == "UPDATE"

deny[msg] withoperation {
    not api.has_label(operation, "environment")
}

has_label[has_label] operation {
    object := operation.object
    kind := object.kind
    # Only apply to Pods
    kind == "Pod" {
    metadata := object.metadata
    has_label := "environment" in metadata.labels
}
```

Save this as `label-template.yaml` and apply it:

```bash
kubectl apply -f label-template.yaml
```

## Binding the Template into a Policy

A **Constraint** binds a ConstraintTemplate to concrete values. This example enforces the label requirement across all namespaces:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: Constraint
metadata:
  name: require-environment-label
spec:
  match:
    # Apply to all Pods in any namespace
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    # Bind the "environment" parameter from the template
    labelSelector: matchLabels:
      environment: "{{ .environment }}"
  template:
    # Reference the package and rule from the ConstraintTemplate
    match:
      effective: true
    params:
      environment: "production"
```

Apply it:

```bash
kubectl apply -f label-constraint.yaml
```

Now any Pod YAML missing an `environment` label will be rejected:

```bash
kubectl apply -f pod-missing-label.yaml
# Error: Pod must have an environment label
```

## Practical Policies you should have

### 1. Prohibit privileged containers

```rego
package k8s.admission.privileged

deny[msg] {
    container := input.review.object.spec.containers[_]
    container.privileged == true
}

msg = "Privileged containers are not allowed"
```

Constraint:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: Privileged
metadata:
  name: disallow-privileged
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod", "Deployment"]
  template:
    match:
      effective: true
    params: {}
```

### 2. Enforce CPU requests equal to limits for critical workloads

```rego
package k8s.admission.cpurules

deny[msg] withoperation {
    container := input.review.object.spec.containers[_]
    not container.resources.requests.cpu
    not container.resources.limits.cpu
    container.resources.requests.cpu != container.resources.limits.cpu
}

msg = "CPU requests must equal limits for this container"
```

### 3. Force a namespace label on all resources

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: NamespaceLabel
metadata:
  name: require-namespace-label
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["*"]
  template:
    match:
      effective: true
    params:
      environment: "production"
```

## Debugging Admission Failures

When `kubectl apply` rejects a manifest, the error message is often terse. To see the full Rego evaluation:

```bash
kubectl get podgatekeepergatekeeper -n gatekeeper-system -o yaml
```

Inspect the `audit` webhook logs, or enable debug logging on the Gatekeeper controller. Most of the time, the rejection message ("must have label", "privileged containers not allowed") is sufficient.

You can also dry-run the admission check locally with `opa eval`:

```bash
opa eval --input '{"input": {"review": {"object": {"spec": {"containers": [{"name": "app", "image": "nginx", "resources": {"requests": {"cpu": "100m"}, "limits": {"cpu": "100m"}}}]}}}}' 'data.k8s.admission.cpurules' 2>&1
```

## Conclusion

Gatekeeper shifts security-left by encoding policies as code that lives alongside your manifests. Once templates are committed to version control, every `kubectl apply` becomes a gate — unsafe configurations are blocked before they reach the API server. Start with one or two high-impact policies (label requirements, privileged container bans) and expand from there.

*Always test policies in a non-critical namespace first. The Rego examples above use safe, read-only checks; avoid destructive operations like `kubectl delete` during evaluation.*

*This article draws on common Kubernetes admission control patterns and Gatekeeper documentation. Adjust template selectors, parameter bindings, and policy scopes to match your organization's SLOs and compliance requirements.*