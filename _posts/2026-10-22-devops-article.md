---
layout: post
title: "Terraform Remote Backends: S3, DynamoDB Locking, and State Migration"
date: 2026-10-22 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [terraform, s3, dynamodb, remote-backend, state, devops, cloud-infrastructure]
description: "Practical guide to configuring Terraform remote backends using Amazon S3 for state storage and DynamoDB for locking, plus migration patterns for existing local state."
---

# Terraform Remote Backends: S3, DynamoDB Locking, and State Migration

Terraform stores state locally by default, which works for individual developers but breaks down completely when multiple team members or CI/CD pipelines need to apply changes simultaneously. A remote backend solves this by storing state in a collaborative storage system with concurrency control. This article walks through configuring a remote backend using Amazon S3 for state persistence and DynamoDB for lock coordination, plus safe migration patterns for existing local state.

## Why Remote Backends Matter

When team members run `terraform apply` concurrently without a remote backend, the last writer wins — previous changes are silently overwritten. Remote backends provide:

- **State sharing** — all team members read from the same source of truth.
- **Concurrency safety** — locking prevents two operations from modifying state simultaneously.
- **Auditability** — history of state changes is visible and recoverable.
- **CI/CD compatibility** — pipelines can apply Terraform changes without local state files.

## Configuring S3 Backend with DynamoDB Locking

The following `backend.tf` configures Terraform to use Amazon S3 for state storage and DynamoDB for lock coordination. Adjust the bucket name, key prefix, and region to match your environment.

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

**Bucket policy** — the S3 bucket must allow the Terraform IAM principal to put objects and retrieve them. A minimal policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::my-terraform-state/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/terraform-locks"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:DescribeTable"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/terraform-locks"
    }
  ]
}
```

**DynamoDB table** — create a table with `LockID` as the partition key and a numeric attribute (e.g., `Status`) as the sort key. Use On-Demand capacity for most workloads; provisioned capacity is fine for low-frequency apply operations.

## Migration from Local to Remote State

Migrating an existing local state to a remote backend is a common requirement. Terraform’s `state mv` command moves state from one backend to another without losing history.

```bash
# 1. Initialize with the new backend (backend config only, no state sync yet)
terraform init \
  -backend-config="bucket=my-terraform-state" \
  -backend-config="key=production/terraform.tfstate" \
  -backend-config="region=us-east-1" \
  -backend-config="dynamodb_table=terraform-locks" \
  -reconfigure

# 2. Move the local state into the new remote backend
terraform state mv \
  -state=local \
  -state-out="s3://my-terraform-state/production/terraform.tfstate" \
  "" \
  "aws_instance.example"
```

After migration, verify the state is accessible:

```bash
terraform plan
```

If the plan reads from the remote backend and shows the expected resources, the migration succeeded.

**Important:** Always run `terraform force-unlock` if a prior operation left a stale lock, and coordinate with team members during the migration window to avoid concurrent modifications.

## Best Practices

1. **Use a dedicated bucket** — never store Terraform state in a bucket that also holds application artifacts or logs.
2. **Enable bucket versioning** — provides a safety net if state is accidentally overwritten or deleted.
3. **Lock timeouts** — if a `terraform apply` is interrupted, the DynamoDB lock expires after 180 seconds by default. Adjust with `lock_timeout` in the backend config if your operations take longer.
4. **Separate environments** — use distinct keys within the same bucket (e.g., `production/`, `staging/`, `dev/`) to avoid key collisions.
5. **Backup the state** — periodically copy the S3 object to a separate bucket or export it via `terraform state pull > backup.tfstate` for disaster recovery.

## Conclusion

A properly configured remote backend is foundational for any team using Terraform at scale. S3 provides durable state storage, DynamoDB offers reliable lock coordination, and `terraform state mv` makes migration painless. Follow the patterns above to bring your team’s Terraform workflow into a collaborative, auditable, and CI/CD‑friendly state.

*This article is intended for operational reference. Always test backend changes in a non-production workspace and verify IAM permissions before applying to shared environments.*
