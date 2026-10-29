---
layout: post
title: "OPA Gatekeeper: Enforcing Kubernetes Security Policies at Admission Time"
date: 2026-10-27 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, opa, gatekeeper, security, admission-controllers, devops]
description: "Practical guide to installing OPA Gatekeeper and enforcing security policies at Kubernetes admission time, with constraint templates and real-world patterns."
---
# OPA Gatekeeper: Enforcing Kubernetes Security Policies at Admission Time

By default, Kubernetes admits any pod or resource that passes the API server's basic validation. This "allow-everything" baseline works for getting started but creates security and compliance risk at scale. OPA Gatekeeper extends the Kubernetes API with constraint-based admission control, letting you declare and enforce policies at the moment resources are created or updated.

This article walks through installing Gatekeeper, writing constraint templates, and applying practical policies that prevent misconfigurations before they reach the cluster.

## Why Admission Policies Matter

A pod that runs with privileged containers, exposes unnecessary ports, or exceeds resource limits can affect cluster stability and security. While Kubernetes Security Contexts provide some control, they require per-pod configuration and do not compose across teams or namespaces. Gatekeeper moves policy into the infrastructure layer, where it is:

- **Declarative**: Policies live as Kubernetes resources (ConstraintTemplate + Constraint).
- **Reusable**: A single template can be bound to multiple namespaces or the entire cluster.
- **Auditable**: Admission decisions are logged and visible via `kubectl get constraints`.
- **Enforceable**: Policies block or mutate resources at creation/update time, preventing drift.

## Prerequisites

- A Kubernetes cluster (this guide uses a local `kind` cluster or a managed service).
- `kubectl` configured to talk to the cluster.
- `helm` installed for Gatekeeper deployment.

## 1. Installing Gatekeeper

Gatekeeper consists of two components: the `gatekeeper-controller-manager` that watches for Constraint and ConstraintTemplate resources, and the mutation/admission webhook it registers with the API server.

Install via Helm:

```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo update
helm install gatekeeper gatekeeper/gatekeeper --namespace gatekeeper --create-namespace
```

Verify installation:

```bash
kubectl get pods -n gatekeeper
```

Expected output (one or more controller pods running):

```
NAME READY STATUS RESTARTS AGE
gatekeeper-...   1/1 Running   0   10m
```

## 2. Writing a Constraint Template

A **ConstraintTemplate** extends the Kubernetes API with a new `kind`. It contains the Rego policy logic (OPA policy language) and binds it to a custom resource kind that users can reference.

The following template enforces that all containers specify a `readOnlyRootFilesystem: true`. Save this as `readonly-ftp.yaml`:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: containersreadonlyrootfilesystem
spec:
  crd:
    spec:
      names:
        kind: ContainerReadOnlyRootFilesystem
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8s.admission

        deny[msg] {
          input.review.object.kind == "Pod"
          not input.review.object.spec.containers[_].readOnlyRootFilesystem == true
          msg = sprintf("container %q must have readOnlyRootFilesystem set to true", [review.object.metadata.name])
        }
```

Apply the template:

```bash
kubectl apply -f readonly-ftp.yaml
```

## 3. Applying the Constraint

A **Constraint** is the instance of a ConstraintTemplate applied to a namespace or the cluster. It references the template by name and specifies which resources the policy applies to.

Apply a constraint that enforces `readOnlyRootFilesystem: true` across all namespaces:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: ContainerReadOnlyRootFilesystem
metadata:
  name: enforce-readonly-ftp
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaceSelector:
      matchLabels:
        enforce: "true"
  values:
    - readOnlyRootFilesystem: true
```

Apply:

```bash
kubectl apply -f readonly-constraint.yaml
```

What this does:

- The `match` section selects all `Pod` resources (except those in namespaces without the `enforce: "true"` label).
- The `values` section provides the enforced values from the template's `deny` rule.
- If a user creates a Pod without `readOnlyRootFilesystem: true` in a matching namespace, the admission webhook rejects the request.

Verify the constraint is registered:

```bash
kubectl get constraint
```

## 4. Practical Constraint Patterns

### 4.1 Resource Requests and Limits

Prevent pods from being scheduled without explicit CPU and memory requests, which can cause the Kubernetes scheduler to make suboptimal placement decisions.

ConstraintTemplate (`resource-requests.yaml`):

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: require-resource-requests
spec:
  crd:
    spec:
      names:
        kind: RequireResourceRequests
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8s.admission

        deny[msg] {
          input.review.object.kind == "Pod"
          count(input.review.object.spec.containers[_].resources.requests) == 0
          msg = sprintf("pod %q must define resource requests", [review.object.metadata.name])
        }
```

Constraint (`resource-requests-constraint.yaml`):

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: RequireResourceRequests
metadata:
  name: enforce-resource-requests
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  values: []
```

### 4.2 Container Image Policy

Restrict which container registries or image patterns are allowed. This is useful for enforcing internal registry usage.

ConstraintTemplate (`allowed-images.yaml`):

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: allowedcontainerimages
spec:
  crd:
    spec:
      names:
        kind: AllowedContainerImages
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8s.admission

        allowed := ["registry.example.com/*", "gcr.io/*"]

        deny[msg] {
          input.review.object.kind == "Pod"
          not any(input.review.object.spec.containers[_].image, allowed)
          msg = sprintf("image %q is not in the allowed list", [review.object.spec.containers[_].image])
        }
```

Constraint (`allowed-images-constraint.yaml`):

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: AllowedContainerImages
metadata:
  name: enforce-image-policy
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  values: []
```

### 4.3 Privileged Container Ban

Prevent containers from running with `privileged: true`, which grants all capabilities on the node.

ConstraintTemplate (`no-privileged.yaml`):

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: no privilegedpods
spec:
  crd:
    spec:
      names:
        kind: NoPrivilegedPods
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8s.admission

        deny[msg] {
          input.review.object.kind == "Pod"
          any(input.review.object.spec.containers[_].securityContext.privileged == true)
          msg = "privileged containers are not allowed"
        }
```

Constraint (`no-privileged-constraint.yaml`):

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: NoPrivilegedPods
metadata:
  name: ban-privileged
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  values: []
```

## 5. Verification

### 5.1 Test That a Policy Blocking Works

Create a Pod that violates a policy and confirm admission is denied:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-no-readonly
  labels:
    enforce: "true"
spec:
  containers:
  - name: test
    image: nginx
    # note: no readOnlyRootFilesystem set
EOF
```

Expected output:

```
Error from server: error when creating "STDIN": admission webhook "validation.gatekeeper.sh" denied the request: constraint "enforce-readonly-ftp": Pod must have readOnlyRootFilesystem set to true
```

### 5.2 Test That a Compliant Pod Admits

Create a Pod that satisfies the policy:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-with-readonly
  labels:
    enforce: "true"
spec:
  containers:
  - name: test
    image: nginx
    securityContext:
      readOnlyRootFilesystem: true
EOF
```

Expected output: Pod created successfully.

### 5.3 List Admission Decisions

Gatekeeper logs admission decisions in the controller logs. View them with:

```bash
kubectl logs -n gatekeeper -l app.kubernetes.io/name=gatekeeper
```

## 6. Tips and Gotchas

- **Start with `audit` mode**: Before enforcing a policy, set `spec.validationAction` to `audit` in the Constraint to log violations without blocking. Switch to `enforce` once the policy is validated.
- **Use namespace selectors strategically**: A Constraint without a `namespaceSelector` applies cluster-wide. Start with a narrow selector (e.g., a `dev` namespace) and expand.
- **Parameterize values**: Constraints accept `values` from the template. Use `values` to make policies configurable per environment without modifying the template.
- **Combine with NetworkPolicies**: Network Policies control runtime traffic flow; Gatekeeper controls admission-time configuration. Together they form a zero-trust posture: no unauthorized traffic enters, and no misconfigured pods are admitted.
- **Debug with `kubectl auth can-i`**: If a policy seems to not be firing, verify the service account running the workload has permission to create the resource in the target namespace.

## Conclusion

OPA Gatekeeper shifts security and configuration policy from "best effort per-pod" to "infrastructure-enforced declarative rules." By writing ConstraintTemplates once and binding them across namespaces or the entire cluster, DevOps teams can prevent entire classes of misconfigurations before they reach running workloads. Start with a single policy (e.g., `readOnlyRootFilesystem` or `resource requests`), verify it blocks violations and admits compliant resources, and iterate toward a full set of admission guards that match your organization's compliance and security requirements.

*This article is intended for operational reference. Always test policies in a non-critical namespace first and verify that legitimate workloads are not blocked before enforcing cluster-wide.*

*This article draws on common OPA Gatekeeper patterns and production admission control experiences. Adjust constraint templates and values to match your organization's security policies, container registries, and runtime requirements.*