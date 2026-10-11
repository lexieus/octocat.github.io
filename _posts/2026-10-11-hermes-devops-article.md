---
layout: post
title: "Automating Kubernetes Operations with Hermes Agent: Practical Commands and Workflows"
date: 2026-10-11 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps, Kubernetes]
tags: [hermes-agent, kubernetes, devops, automation, cli]
description: "A practical guide to using Hermes Agent for common Kubernetes cluster management tasks, including namespace operations, resource inspection, and routine administrative workflows."
---

# Automating Kubernetes Operations with Hermes Agent: Practical Commands and Workflows

When managing Kubernetes clusters, routine administrative tasks can consume significant time. Hermes Agent provides a direct interface for executing Kubernetes operations programmatically, enabling automation of repetitive workflows and integration into existing DevOps pipelines.

This article covers practical, safe Hermes Agent commands for common Kubernetes cluster management tasks. All examples are non-destructive and designed for read-only inspection or controlled operations.

## Prerequisites

Hermes Agent must be configured with access to your Kubernetes cluster. Verify connectivity:

```bash
hermes kubernetes status
```

If using a custom kubeconfig, specify it via the `KUBECONFIG` environment variable or Hermes' configuration profile system.

## Inspecting Cluster Resources

### List all namespaces

```bash
hermes kubernetes get namespaces
```

Output includes NAME, STATUS, AGE, and default resource quotas/limit ranges applied.

### Describe a specific namespace

```bash
hermes kubernetes describe namespace production
```

Returns detailed information including resource quotas, limit ranges, and associated metadata.

### Check resource quotas in a namespace

```bash
hermes kubernetes get resourcequota --all-namespaces
```

Filters to a specific namespace:

```bash
hermes kubernetes get resourcequota --namespace=staging
```

### List limit ranges

```bash
hermes kubernetes get limitrange --all-namespaces
```

## Controlled Operations

### Create a resource quota

This example creates a quota limiting a development namespace to 2 CPU and 4 GiB memory:

```bash
hermes kubernetes create resourcequota dev-quota \
  --namespace=development \
  --hard=cpu=2,memory=4Gi
```

### Create a limit range

Ensures all pods in a namespace have sensible resource defaults:

```bash
hermes kubernetes create limitrange resource-limits \
  --namespace=development \
  --type=container \
  --default-cpu=500m --default-memory=256Mi \
  --min-cpu=100m --min-memory=64Mi \
  --max-cpu=1 --max-memory=512Mi
```

### Apply a LimitRange first, then a ResourceQuota

A common workflow pattern:

```bash
# 1. Apply limit range
hermes kubernetes apply -f limitrange.yaml

# 2. Apply resource quota
hermes kubernetes apply -f resourcequota.yaml

# 3. Deploy workload (validated against both)
hermes kubernetes apply -f deployment.yaml
```

## Read-Only Inspection Patterns

### Check quota usage status

```bash
hermes kubernetes get resourcequota compute-quota -n development -o json
```

Typical output:

```json
{
  "name": "compute-quota",
  "namespace": "development",
  "hard": {
    "cpu": "2",
    "memory": "4Gi"
  },
  "used": {
    "cpu": "800m",
    "memory": "1.5Gi"
  },
  "age": "5h"
}
```

### List pods exceeding resource norms

```bash
hermes kubernetes get pods --all-namespaces --sort-by='{.metadata.name}'
```

Then check individual pod resource allocation:

```bash
hermes kubernetes describe pod my-app -n staging
```

Look at the `Resource Limits` and `Requests` sections in the output.

## Pipeline Integration Example

Add these Hermes commands to a CI/CD job that validates namespace readiness before deployment:

```bash
#!/bin/bash
# Pre-deployment validation script

NAMESPACE=$1

# Check namespace exists
if ! hermes kubernetes get namespace "$NAMESPACE" > /dev/null 2>&1; then
  echo "ERROR: Namespace $NAMESPACE does not exist"
  exit 1
fi

# Check resource quota status
QUOTA_STATUS=$(hermes kubernetes get resourcequota -n "$NAMESPACE" -o json)
USED_CPU=$(echo "$QUOTA_STATUS" | jq '.used.cpu')
USED_MEMORY=$(echo "$QUOTA_STATUS" | jq '.used.memory')
HARD_CPU=$(echo "$QUOTA_STATUS" | jq '.hard.cpu')
HARD_MEMORY=$(echo "$QUOTA_STATUS" | jq '.hard.memory')

echo "Namespace: $NAMESPACE"
echo "CPU used: $USED_CPU / $HARD_CPU"
echo "Memory used: $USED_MEMORY / $HARD_MEMORY"

# Warn if > 80% consumed
CPU_PCT=$(echo "scale=0; $USED_CPU * 100 / $HARD_CPU" | bc)
MEM_PCT=$(echo "scale=0; $USED_MEMORY * 100 / $HARD_MEMORY" | bc)

if [ "$CPU_PCT" -gt 80 ]; then
  echo "WARNING: CPU usage above 80% ($CPU_PCT%)"
fi

if [ "$MEM_PCT" -gt 80 ]; then
  echo "WARNING: Memory usage above 80% ($MEM_PCT%)"
fi
```

Run as part of your deployment pipeline:

```bash
./validate-namespace.sh production
```

If warnings are raised, adjust resources or proceed with caution.

## Best Practices

| Practice | Recommendation |
|----------|----------------|
| **Start with LimitRanges** | Apply limit ranges before enabling resource quotas to ensure all workloads have defined boundaries |
| **Right-size requests** | Set resource requests based on actual workload baselines, not peak traffic |
| **Separate environments** | Use distinct namespaces with independent quotas for dev, staging, and production |
| **Review regularly** | Adjust quotas as team patterns evolve — automate reviews with Hermes scheduling |
| **Document limits** | Include quota information in onboarding docs for new teams |
| **Automate cleanup** | Use Hermes to remove temporary namespaces and associated quotas after feature branches |

## Conclusion

ResourceQuotas and LimitRanges are fundamental Kubernetes primitives for operating multi-tenant clusters responsibly. ResourceQuotas prevent namespace-level resource exhaustion, while LimitRanges ensure every pod has meaningful resource boundaries. Together they form the foundation of predictable, reliable cluster operations.

Start with a LimitRange across all your namespaces, then add ResourceQuotas tailored to team or environment needs. The upfront investment pays off by preventing the all-too-common "node pressure" incidents that disrupt services.

For cluster-specific guidance, consult your Kubernetes distribution's official documentation and Hermes Agent's complete command reference.

---
*This article draws on common Kubernetes administration patterns and Hermes Agent's operational capabilities. For production-specific guidance, test all commands in a staging environment first.*