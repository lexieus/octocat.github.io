---
layout: post
title: "Writing Reusable Terraform Modules: Best Practices for DevOps Engineers"
date: 2026-11-08 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [terraform, infrastructure-as-code, modules, devops, cloud-infra]
description: "A practical guide to writing modular, reusable, and maintainable Terraform modules for infrastructure-as-code projects, including input/output patterns, versioning, and consumption strategies."
---

# Writing Reusable Terraform Modules: Best Practices for DevOps Engineers

Terraform modules are the primary unit of reusability in infrastructure-as-code. A well-designed module encapsulates complexity, hides implementation details, and provides a clean interface for consumers. Poorly designed modules become technical debt, forcing repeated modifications across many configurations.

This article walks through the patterns and practices that separate production-grade modules from ad-hoc scripts.

## 1. Module Structure and Conventions

Every Terraform module should follow a consistent layout so that consumers can intuitively navigate its contents without reading documentation.

```
terraform/aws-vpc/
  main.tf
  variables.tf
  outputs.tf
  versions.tf
  README.md
  examples/
    ec2/
      main.tf
      variables.tf
```

### Recommended file layout

- **`main.tf`** — resources and data sources that compose the module's primary functionality.
- **`variables.tf`** — input variables with descriptions, types, and default values where appropriate.
- **`outputs.tf`** — exported values that consumers use to reference created resources.
- **`versions.tf`** — version constraints for providers and dependencies, keeping the root `version` block lean.
- **`README.md`** — usage instructions, input/output tables, and examples. This is essential for discoverability in private registries or shared drives.
- **`examples/`** — one or more minimal working examples. Prefer separate directories per use-case (e.g., `ec2/`, `alb/`) rather than a single monolithic example.

### Example: Minimal `variables.tf`

```hcl
variable "name" {
  description = "A unique identifier for the resource group, used in tags and naming."
  type        = string
}

variable "cidr_block" {
  description = "The CIDR block for the VPC."
  type        = string
}

variable "tags" {
  description = "Additional tags to apply to all resources."
  type        = map(string)
  default     = {}
}
```

### Example: Minimal `outputs.tf`

```hcl
output "vpc_id" {
  description = "The ID of the created VPC."
  value       = aws_vpc.main.id
}

output "private_subnet_ids" {
  description = "IDs of the private subnets."
  value       = aws_subnet.private.*.id
}
```

### Example: Minimal `versions.tf`

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

## 2. Input Design: Fewer Primitives, More Composability

A common mistake is exposing every AWS API parameter as a variable. Instead, define variables at the domain level and let the module handle internal translation.

### Anti-pattern: Exposing too many primitives

```hcl
# Avoid: consumers must know internal details
variable "subnet_cidr" { type = string }
variable "az_name" { type = string }
variable "enable_dns" { type = bool }
```

### Better: Group related values into structured arguments

```hcl
variable "cidr_config" {
  description = "Configuration for CIDR block allocation."
  type = object({
    vpc      = string
    private  = string
    public   = string
    isolation = string
  })
}
```

This keeps the module's public surface area small while preserving flexibility internally.

## 3. Output Strategy: What Consumers Actually Need

Exports should be minimal and focused. A module should output identifiers, not computed attributes that the consumer can derive.

### Good outputs

- Resource IDs (`vpc_id`, `sg_id`)
- Connection details (`db_endpoint`, `elb_dns_name`)
- Names and ARNs for referencing in other modules

### Avoid exporting

- Calculated CIDR blocks (derive from inputs)
- Intermediate resource names (unless required for external reference)
- Raw attribute maps that serve no downstream purpose

### Example: Purposeful outputs

```hcl
output "vpc_id" {
  description = "The ID of the VPC."
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "IDs of public subnets, for use in routing or security group references."
  value       = aws_subnet.public.*.id
}
```

## 4. Versioning and Publishing

Modules should be versioned independently of the configurations that consume them. The two most common approaches are the Terraform Registry and private module sources.

### Using the Terraform Registry

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  # ... inputs
}
```

### Using a private registry or Git source

```hcl
module "vpc" {
  source  = "git::https://github.com/your-org/terraform-modules//vpc"
  version = "main"
}
```

Or with a tag:

```hcl
version = "v3.2.1"
```

### Publishing a module to the Registry

1. Add a `README.md` with input/output tables and usage examples.
2. Create a `main.tf`, `variables.tf`, and `outputs.tf` at the root.
3. Initialize the directory: `terraform init`.
4. Run the prepublish checks: `terraform fmt`, `terraform validate`.
5. Tag a version and push: `git tag v1.0.0 && git push origin v1.0.0`.
6. The Registry automatically picks up the tag and publishes the module.

## 5. Consuming Modules in GitOps Workflows

In a GitOps pipeline, modules are typically stored in a dedicated `modules/` directory within the same repository as your applications. This keeps version control tight and avoids network dependencies during pipeline execution.

### Example: Module directory structure

```
infra/
  modules/
    vpc/
      main.tf
      variables.tf
      outputs.tf
  environments/
    prod/
      main.tf
    staging/
      main.tf
```

### Example: Using a local module

```hcl
module "vpc_prod" {
  source = "./modules/vpc"
  name   = "prod"
  cidr_block = "10.0.0.0/16"
}
```

This pattern works well with ArgoCD or other GitOps controllers, since the module code lives in the same Git repo as the application manifests.

## 6. Common Pitfalls to Avoid

| Pitfall | Result | Fix |
|---|---|---|
| No `versions.tf` | Provider version drift across teams | Always pin provider versions in a dedicated file |
| Outputs that duplicate inputs | Confusion about what to use | Outputs should be resource IDs or ARNs; derive other values from inputs |
| Undocumented variables | Team members guess types and defaults | Write a `README.md` for every module, even internal ones |
| Mixing resource types without logic | Modules that do too much | Split concerns: one module per AWS service or infrastructure component |
| Hardcoding values in examples | Examples not reusable | Use input variables in examples; show default values |

## 7. Quick Checklist for Module Reviews

Before merging a new module or bumping a version, verify:

- [ ] All input variables have descriptions and types in `variables.tf`
- [ ] All outputs have descriptions and return meaningful values in `outputs.tf`
- [ ] `versions.tf` pins the required Terraform version and provider versions
- [ ] `README.md` includes a usage example with variable tables
- [ ] Module does not export intermediate names or computed values unless necessary
- [ ] Module works in isolation with `terraform init` and `terraform apply`
- [ ] No hardcoded AWS account IDs, region strings, or names in example manifests

## Conclusion

Reusable Terraform modules are a force multiplier for any DevOps team. They reduce duplication, enforce consistent patterns, and make infrastructure changes safer and more predictable. Start by extracting one frequently-used configuration into a module, follow the structure and conventions above, and iterate based on real usage feedback.

*Always test modules in a dedicated workspace before promoting to production. Module design is iterative — gather feedback from consumers, adjust variable names and types, and refine outputs based on actual integration patterns.*

---
*This article draws on common Terraform module patterns and production infrastructure setups. Adjust variable names, resource types, and module boundaries to match your organization's cloud providers, naming conventions, and operational practices.*