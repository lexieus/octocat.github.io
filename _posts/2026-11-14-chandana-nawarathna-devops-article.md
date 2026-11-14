---
layout: post
title: "Kubernetes Secrets Management: External-Secrets and Native Approaches"
date: 2026-11-14 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [kubernetes, secrets, external-secrets, vault, aws-secrets-manager, devops]
description: "A practical guide to managing Kubernetes secrets using the External-Secrets operator, covering integration with AWS Secrets Manager, HashiCorp Vault, and Azure Key Vault, plus native alternatives."
---

# Kubernetes Secrets Management: External-Secrets and Native Approaches

## Why Secret Management Matters in Kubernetes

Kubernetes secrets are base64-encoded by default and stored etcd. Without encryption-at-rest and proper access controls, anyone with cluster read access can decode sensitive values. Common pain points include:

- **Etcd-backed secrets are not encrypted** by default — anyone with `kubectl get secret` and base64 decode can read them.
- **Secret rotation is manual** — rotating database passwords, API keys, or TLS certificates often involves editing secrets and restarting workloads.
- **Secrets spread across namespaces** — teams may duplicate secrets or hardcode values in manifests.
- **External secret managers** (AWS Secrets Manager, HashiCorp Vault, Azure Key Vault) are not natively integrated — each requires custom tooling.

The External-Secrets operator bridges this gap by syncing secrets from external sources into Kubernetes, giving you the same workflow (`kubectl apply`) while keeping sensitive values in managed secret stores.

## The External-Secrets Operator

Install the operator via Helm:

```bash
helm repo add external-secrets https://external-secrets.github.io/kubernetes-external-secrets/
helm repo update
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets-system --create-namespace
```

Verify the installation:

```bash
kubectl get pods -n external-secrets-system
# external-secrets-xxx, external-secrets-controller-xxx should be running
```

## Integrating with AWS Secrets Manager

Assume you have a database password stored in AWS Secrets Manager: `prod/db-password`.

### 1. Create an `ExternalSecret` resource

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-password
  namespace: production
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: aws-secrets-manager
    key: db-password
  target:
    name: db-password-secret
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        property: password
```

Apply it:

```bash
kubectl apply -f externalsecret-db.yaml
```

### 2. Verify the secret is synced

```bash
kubectl get secret db-password-secret -n production -o yaml
# You should see the password value populated from AWS Secrets Manager
```

### 3. Use the secret in a deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: myapp:latest
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-password-secret
                  key: password
```

## Integrating with HashiCorp Vault

For organizations using Vault as their central secret manager, External-Secrets supports the Vault secret store.

### 1. Configure a `VaultSecretStore` (or `VaultGenericSecretStore`)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: VaultSecretStore
metadata:
  name: my-vault-store
  namespace: external-secrets-system
spec:
  vaultAddress: https://vault.example.com:8200
  auth:
    token: "${VAULT_TOKEN}" # or use Kubernetes auth mount
  secretPath: secret/data/my-app
  type: kv
```

### 2. Reference the secret store in an `ExternalSecret`

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-secret
  namespace: production
spec:
  secretStoreRef:
    kind: VaultSecretStore
    name: my-vault-store
  target:
    name: vault-secret
    creationPolicy: Owner
  data:
    - secretKey: api-key
      remoteRef:
        key: api-key
```

Apply both resources:

```bash
kubectl apply -f vault-secretstore.yaml
kubectl apply -f externalsecret-vault.yaml
```

### 3. Use the secret in your application

Same pattern as the AWS example — reference `vault-secret` via `secretKeyRef` in your pod specs.

## Native Kubernetes Secret Alternatives

If you prefer not to add an operator, two native approaches can provide better security than raw etcd secrets.

### Sealed Secrets (cert-manager based)

Sealed Secrets encrypt a secret using a key managed by cert-manager, producing a `SealedSecret` that can be safely committed to Git. The decryption key never leaves the cluster.

```bash
# Encrypt a local secret
kubeseal --fetch-keys > keys.yaml # first time only
echo '{"password":"my-secret"}' | kubeseal -o yaml > sealed-secret.yaml
```

Apply the sealed secret, which the cert-manager controller automatically decrypts into a regular Kubernetes secret.

### Kubernetes Secret Best Practices (without external tools)

- **Enable encryption at rest** for the `etcd` backend: `--experimental-encryption-provider-config` with a KMS provider (AWS KMS, GCP KMS, Azure Key Vault).
- **Limit secret exposure**: use `stringData` instead of `data` when possible (plaintext is base64-encoded on storage, but `stringData` is more developer-friendly).
- **Rotate secrets regularly** and automate rotation where possible (e.g., AWS Secrets Manager rotation Lambda functions).
- **Avoid putting secrets in pod specifications** — use volume mounts or environment variable references instead.

## Comparison: External-Secrets vs. Native Approaches

| Aspect | External-Secrets | Sealed Secrets / Native |
|---|---|---|
| **Secret source** | AWS Secrets Manager, Vault, Azure Key Vault, Google Secret Manager | Local generation, Git-encrypted |
| **Auto-sync** | Real-time sync from external source | Manual `kubectl apply` / GitOps |
| **Rotation** | Can trigger re-sync on secret version change | Manual or custom controller |
| **Git-friendliness** | SecretStore resources can be in Git; ExternalSecrets less commonly committed | SealedSecrets fully Git-compatible |
| **Operator overhead** | Requires External-Secrets operator installed | cert-manager for Sealed Secrets; native for plain secrets |

## Practical Workflow Recommendation

1. **For greenfield clusters**: Start with External-Secrets if you already use AWS/Vault/Azure. The operator handles sync, versioning, and re-fetching automatically.
2. **For existing GitOps pipelines**: Sealed Secrets integrate naturally — encrypt secrets in your repo, let cert-manager decrypt them into Kubernetes secrets at apply time.
3. **Hybrid approach**: Use External-Secrets for frequently rotated secrets (database passwords, API keys) and native encrypted secrets for static TLS certificates or cluster-specific credentials.

## Conclusion

Managing secrets in Kubernetes requires balancing convenience, security, and operational overhead. The External-Secrets operator reduces the friction of integrating with external secret managers while keeping your Kubernetes workflow unchanged. For teams already invested in GitOps, Sealed Secrets provide a zero-operator alternative that still improves upon plain etcd secrets.

Start with one integration — whether it's fetching a database password from AWS Secrets Manager or pulling an API key from Vault — and expand your secret management strategy as your team's needs evolve.

*Always restrict access to secret stores via IAM policies. Never commit raw secret values to Git. Test secret integration in a staging namespace before promoting to production.*