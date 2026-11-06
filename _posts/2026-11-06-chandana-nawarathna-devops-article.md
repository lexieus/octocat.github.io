---
layout: post
title: "OPA Gatekeeper: Enforcing Policy as Code in Kubernetes Clusters"
date: 2026-11-06 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, opa, gatekeeper, policy-as-code, security, devops]
description: "A practical guide to implementing OPA Gatekeeper policy-as-code in Kubernetes clusters, with ConstraintTemplates, real-world policy examples, and CI/CD integration patterns."
---
OPA Gatekeeper: Enforcing Policy as Code in Kubernetes Clusters

## Why Policy as Code Matters for Kubernetes

Kubernetes manifests are declarative and flexible, but that flexibility can lead to configuration drift, security gaps, and resource quotas being exceeded unintentionally. OPA (Open Policy Agent) Gatekeeper extends the Kubernetes admission control layer to enforce custom policies defined as Rego code. Instead of relying on manual reviews or documentation, every `kubectl apply` is validated against codified rules before it reaches the cluster.

This approach provides:
- **Automated compliance** — policies are checked on every apply, preventing non-compliant resources from being created.
- **Consistency** — the same rules apply across teams, namespaces, and clusters.
- **Auditability** — every violation is recorded, making it easy to track who tried what and when.
- **Shift-left security** — issues are caught early in the CI/CD pipeline, not at runtime.

## How Gatekeeper Works

Gatekeeper operates through two core Kubernetes custom resources:

1. **ConstraintTemplate** — defines a reusable schema and the Rego logic that evaluates it. It connects an external OpenAPI v3 schema to a Rego rule.
2. **Constraint** — instantiates a ConstraintTemplate by providing values for the schema's parameters. Multiple Constraints can reference the same Template with different values.

When a user creates or modifies a resource, the Kubernetes API server invokes the Gatekeeper controller, which evaluates the resource against all matching Constraints. If any Constraint rejects the resource, the admission is denied and an error is returned to the client.

## Practical Example: Enforcing CPU Request Limits

A common requirement for platform teams is to ensure that no pod runs without CPU requests, which can cause the Kubernetes scheduler to place pods unpredictably or trigger OOM kills when the container attempts to use unbounded CPU.

### Step 1: Create a ConstraintTemplate

The Template defines the shape of the constraint and the Rego rule that enforces it.

```rego
package k8s.admission.constraints

import data.kubernetes as k8s

# Schema defines the shape of the Constraint resource.
# This template ensures every pod has CPU requests defined.
schemas/
    enforce_cpu_requests {
        scope: "namespace"
        kinds: ["Pod"]
        predicate: {"match": {"kinds": [{"apiGroups": [""], "kinds": ["Pod"]}]}}
    }

# The Rego rule that actually validates the condition.
deny[msg] {
    not k8s.admission.review.kind.match.kinds[_] == "Pod"
    not any(count[k8s.admission.review.object.spec.containers[_].resources.requests.cpu] > 0)
    msg = sprintf("pod %s/%s must define CPU requests on all containers", [k8s.admission.review.namespace, k8s.admission.review.object.metadata.name])
}
```

### Step 2: Instantiate the Constraint

The Constraint binds the Template to specific values, such as the target namespace.

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: enforce_cpu_requests
metadata:
  name: cpu-requests-must-be-present
spec:
  match:
    kinds:
    - apiGroup: ""
      kind: Pod
  name: "require-cpu-requests"
  # Optional: enforce only in specific namespaces
  # namespaceSelector:
  #   matchLabels:
  #     team: platform
```

### Step 3: Test the Policy

Apply a pod without CPU requests — Gatekeeper will reject it:

```bash
kubectl apply -f pod-no-cpu.yaml
# Error: error when creating "pod-no-cpu.yaml": admission webhook "validation.gatekeeper.k8s.io" denied the request: ...
```

Apply a pod with CPU requests — it passes:

```bash
kubectl apply -f pod-with-cpu.yaml
# pod/created successfully
```

## Common Policy Patterns

### 1. Forbidden Container capabilities

Prevent containers from acquiring capabilities that could be used for privilege escalation.

```rego
deny[msg] {
    some cap := k8s.admission.review.object.spec.containers[_].securityContext.capabilities.add[_]
    cap == "NET_RAW" or cap == "NET_ADMIN"
    msg = sprintf("container %s added forbidden capability: %s", [review.object.metadata.name, cap])
}
```

### 2. Prohibit privileged containers

Ensure no pod can run with `privileged: true`, which effectively gives the container root access to the node.

```rego
deny[msg] {
    some container := k8s.admission.review.object.spec.containers[_]
    some privileged := container.securityContext.privileged
    privileged == true
    msg = sprintf("container %s runs with privileged mode", [container.name])
}
```

### 3. Enforce namespace labels for multi-tenancy

Require that every resource carries a `team` label, enabling namespace isolation and ownership tracking.

```rego
deny[msg] {
    not k8s.admission.review.object.metadata.labels["team"]
    msg = "every resource must have a 'team' label for multi-tenant isolation"
}
```

## Installing Gatekeeper

Gatekeeper is deployed as a Kubernetes controller and admission webhook. The quickest way to get started is via Helm:

```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo update
helm install gatekeeper gatekeeper/gatekeeper --namespace gatekeeper-system --create-namespace
```

After installation, verify the webhook is active:

```bash
kubectl get mutatingwebhookconfiguration | grep gatekeeper
kubectl get validatingwebhookconfiguration | grep gatekeeper
```

## Integrating with CI/CD

Policies should be version-controlled alongside manifests. A typical GitOps workflow:

1. **Policy as code** — Rego rules and ConstraintTemplates live in a `policy/` directory in the same repository as Kubernetes manifests.
2. **Pre-merge validation** — CI pipelines run `gatekeeper eval` or `opa test` against proposed changes to catch violations before they reach the cluster.
3. **Enforcement** — Merged changes trigger ArgoCD or direct `kubectl apply`, and Gatekeeper validates each resource at admission time.
4. **Violation reporting** — CI extracts violation details from webhook rejection responses and posts them as PR comments, making it easy for developers to fix their manifests.

## Conclusion

OPA Gatekeeper transforms Kubernetes security from a manual, post-hoc process into an automated, codified discipline. By expressing policies as Rego code stored in Git, platform teams can enforce compliance, resource standards, and security boundaries across every cluster in the organization. Start with one high-impact policy — such as requiring CPU requests or prohibiting privileged containers — and iterate toward a full policy set that matches your organization's threat model and operational requirements.

*This article draws on common OPA Gatekeeper patterns and production deployments. Adjust Rego rules, constraint scopes, and policy priorities to match your cluster's API group versions, namespace structure, and security requirements. Always test policies against a staging namespace before deploying to production, and iterate based on developer feedback and real-world usage patterns.*