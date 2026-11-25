---
layout: post
title: "Kubernetes Security Hardening: Practical Patterns for DevOps Engineers"
date: 2026-11-25 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, security, rbac, network-policies, pod-security, devops]
description: "Practical security hardening patterns for Kubernetes clusters: RBAC, Network Policies, Pod Security Standards, and secret management that DevOps engineers can apply today."
---

# Kubernetes Security Hardening: Practical Patterns for DevOps Engineers

A Kubernetes cluster that runs workloads is inherently shared infrastructure. Without deliberate hardening, each deployed application increases the attack surface. This article walks through concrete, non-disruptive security patterns that DevOps engineers can apply to existing clusters.

## 1. RBAC: Give Only What Is Needed

Role-Based Access Control is the first line of defence. The default `cluster-admin` cluster-role grants far too many permissions for day-to-day operations.

### Create a minimal cluster-role for common operations

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: dev-operator
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["get", "list", "create", "update"]
```

Apply it:

```bash
kubectl apply -f dev-operator-clusterrole.yaml
```

Create a `RoleBinding` or `ClusterRoleBinding` to assign it to a user or service account:

```bash
kubectl create rolebinding dev-operator-binding \
  --role-name=dev-operator \
  --user=alice@example.com \
  --namespace=default
```

**Key principle**: Start with read-only verbs (`get`, `list`, `watch`) and add `create`/`update`/`patch` only when the workflow requires it. Review bindings quarterly and remove unused permissions.

## 2. Network Policies: Control Pod-to-Pod Traffic

By default, all pods in a namespace can communicate with each other. Network Policies express which connections are allowed.

### Allow only ingress from a specific namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    ports:
    - protocol: TCP
      port: 8080
```

Apply it:

```bash
kubectl apply -n default -f allow-frontend-backend.yaml
```

### Default-deny policy for a namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: payment
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

This policy matches all pods (`podSelector: {}`) and declares that no ingress or egress is allowed by default. Add explicit policies on top to permit only the required traffic.

Apply it:

```bash
kubectl apply -n payment -f default-deny-all.yaml
```

**Key principle**: Deploy a default-deny policy in namespaces that host sensitive workloads (databases, internal APIs), then add targeted exceptions. Use `allow-external-ingress` or `allow-monitoring` as the only exceptions.

## 3. Pod Security Standards: Enforce Baseline Profiles

Pod Security Standards (PSS) replace the deprecated PodSecurityPolicy and provide three profiles: `privileged`, `baseline`, and `restricted`. Enforce the `restricted` profile for multi-tenant clusters.

### Admit pods that comply with the restricted profile

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: workloads
  annotations:
    kubernetes.io/pss: restricted
```

Alternatively, apply a cluster-wide constraint via the BuiltIn admission controller. Ensure the API server has the `--enable-admission-plugins=PodSecurity` flag.

### Common PSS violations and fixes

| Violation | Fix |
|---|---|
| `runAsNonRoot: false` | Add `runAsNonRoot: true` to the pod spec |
| `allowPrivilegeEscalation: true` | Set `allowPrivilegeEscalation: false` |
| `privileged: true` container spec | Remove the `privileged` flag or redesign the container |
| Missing `readOnlyRootFilesystem` | Add `readOnlyRootFilesystem: true` where the application allows it |

**Key principle**: Start with the `baseline` profile if `restricted` breaks existing workloads, then iterate toward `restricted`. Use the audit mode (`kubernetes.io/pss: audit`) first to see violations without rejecting pods.

## 4. Secret Management: Keep Secrets Out of Plain State

Secret values stored in `kubectl get secret -o yaml` or in Terraform state are base64-encoded but not encrypted at rest. Use dedicated secret management instead of Kubernetes native Secrets for sensitive data.

### Avoid putting secrets in Terraform state

Do not store passwords, TLS private keys, or cloud API secrets in Terraform state. Use one of the following patterns instead:

### Use External Secrets Operator to sync from Vault

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: vault-store
  target:
    name: db-credentials-secret
  data:
    - secretKey: password
      externalKey: db/password  # path in Vault
```

### Use sealed-secrets for version-controlled secrets

```bash
# Encrypt a secret so it can be checked into Git
kubeseal < secret.yaml > secret-sealed.yaml

# Decrypt at deployment time
kubectl apply -f secret-sealed.yaml
```

**Key principle**: Never bake plain-text secrets into container images or commit them to version control. If Kubernetes Secrets must be used, encrypt them at rest using the KMS provider, and rotate them regularly.

## 5. Cluster Hygiene: Regular Cleanup

Security is also about removing what shouldn't be there.

### List cluster-scoped resources that may be unused

```bash
kubectl get clusterroles --ignore-not-found=true
kubectl get clusterrolebindings --ignore-not-found=true
kubectl get csd --ignore-not-found=true  # Central Service Descriptor if applicable
```

### Remove terminated pods and unused image tags

```bash
# Show pods in Terminating state
kubectl get pods --all-namespaces -o json | jq '.items[] | select(.status.phase == "Pending")'

# Prune completed jobs older than 1 hour
kubectl delete jobs --force --grace-period=0 --all -n ci \
  --field-selector=.status.status=Completed \
  --field-selector='.metadata.creationTimestamp"<$(date -d "-1 hour" +%s)'
```

### Audit log retention

Ensure the API server config enables audit logging and sets a retention limit:

```bash
# Example: keep 30 days of audit logs, max 10MB per log file
--audit-log-max-bytes=10485760
--audit-log-max-retention-days=30
```

**Key principle**: Schedule a quarterly cluster clean-up: check for `ClusterRoles` without bindings, remove `Terminating` pods that stuck, prune old completed Jobs, and rotate secrets that are close to their expiration date.

## Conclusion

Kubernetes security is not a one-time checklist — it is a set of habits applied iteratively. Start with these practical patterns:

1. **RBAC** — least-privilege roles and bindings
2. **Network Policies** — default-deny in sensitive namespaces
3. **Pod Security Standards** — baseline or restricted profile
4. **Secret management** — external stores, not plain state
5. **Cluster hygiene** — quarterly reviews and pruning

Each pattern reduces the cluster's attack surface without requiring a full re-architecture. Apply one pattern, verify it works with your workloads, then move to the next. Over time these habits become part of your team's standard operating procedure.

*Always test security changes in a non-critical namespace first. The examples above use read-only or admit-only patterns. Adjust thresholds, labels, and profiles to match your organization's SLOs and compliance requirements.*