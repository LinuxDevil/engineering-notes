---
title: "Infrastructure as Code"
tags: [devops, devops/iac, devops/cloud]
aliases: [IaC, Terraform, OpenTofu, Pulumi, Terraform vs OpenTofu]
updated: 2026-09-27
---
# Infrastructure as Code

<Note>
**TL;DR**

**Infrastructure as Code (IaC)** describes servers, networks, databases, DNS and permissions in versioned files, and a tool makes the real world match them. **Terraform** and its open-source fork **OpenTofu** use declarative HCL, a **state file** and a **plan → apply** workflow; **Pulumi** uses general-purpose languages; cloud-native options include CloudFormation/CDK, Bicep and Crossplane. Terraform moved to the non-open-source **BSL** in 2023 (IBM now owns HashiCorp); OpenTofu (MPL 2.0, Linux Foundation, CNCF sandbox since 2025) is the drop-in open alternative.
</Note>

## The idea

Clicking in a cloud console is fast once and terrible forever: nobody knows what exists, why, or how to recreate it. IaC gives you:
- **Reproducibility**: the same code builds dev, staging and prod.
- **Review**: infra changes go through PRs with a readable plan.
- **History and rollback**: Git log is your change log.
- **Drift detection**: compare reality with code.
- **Automation**: environments on demand (preview environments, disaster recovery).

### Declarative vs imperative, mutable vs immutable

| | Declarative (Terraform, CloudFormation, K8s) | Imperative (scripts, Ansible ad-hoc) |
|---|---|---|
| You write | The desired end state | The steps |
| Idempotent | By design | Only if you write it that way |

**Immutable infrastructure**: never patch servers in place; build a new image (Packer, containers) and replace. Configuration management tools (Ansible, Chef, Puppet) remain useful for VMs and on-prem, but containers + IaC cover most cloud cases.

## Terraform / OpenTofu workflow

{/* diagram source: /diagrams/diagram-171.mmd */}
![diagram](/diagrams/diagram-171.svg)

```hcl
# versions.tf
terraform {
  required_version = ">= 1.10"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.0" }
  }
  backend "s3" {
    bucket       = "acme-tf-state"
    key          = "prod/network/terraform.tfstate"
    region       = "eu-west-1"
    encrypt      = true
    use_lockfile = true      # native S3 locking, no DynamoDB table needed
  }
}

# main.tf
provider "aws" { region = var.region }

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 6.0"
  name    = "prod"
  cidr    = "10.0.0.0/16"
  azs             = ["eu-west-1a", "eu-west-1b", "eu-west-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  enable_nat_gateway = true
}

resource "aws_s3_bucket" "assets" {
  bucket = "acme-prod-assets"
  tags   = local.tags
  lifecycle { prevent_destroy = true }
}

variable "region" {
  type    = string
  default = "eu-west-1"
}
locals { tags = { env = "prod", owner = "platform", managed_by = "terraform" } }
output "vpc_id" { value = module.vpc.vpc_id }
```

```bash
terraform init            # or: tofu init
terraform fmt -check && terraform validate
terraform plan -out=tfplan
terraform apply tfplan
terraform state list
terraform import aws_s3_bucket.legacy legacy-bucket   # or an import {} block
terraform destroy          # careful
```

### Core concepts

- **Provider**: plugin that talks to an API (AWS, GCP, Azure, Kubernetes, Cloudflare, GitHub, Datadog...).
- **Resource**: something to manage; **data source**: something to read.
- **State**: maps resources in code to real IDs. Store it **remotely** (S3/GCS/Azure Blob/Terraform Cloud) with **locking** and **encryption**; it can contain secrets.
- **Module**: reusable, versioned package of resources. Build an internal module library ("paved road").
- **Workspaces / directories per environment**: most teams prefer separate state per environment and per component (network, data, apps) to limit blast radius.
- **`moved`, `import`, `removed` blocks**: refactor and adopt resources through code, not CLI surgery.

## Pulumi and others

```ts
// Pulumi (TypeScript): same idea, real programming language
import * as aws from "@pulumi/aws";
const bucket = new aws.s3.Bucket("assets", { tags: { env: "prod" } });
export const bucketName = bucket.id;
```

| Tool | Language | State | Notes |
|---|---|---|---|
| Terraform | HCL | Self-managed or HCP Terraform | Largest ecosystem; BSL licence |
| OpenTofu | HCL | Self-managed | Open source fork; state encryption, OCI registry support |
| Pulumi | TS/Python/Go/C#/Java/YAML | Pulumi Cloud or self-managed | Loops, tests, types from a real language |
| AWS CloudFormation / CDK | YAML/JSON / TS, Python... | Managed by AWS | AWS only; CDK synthesises CloudFormation |
| Azure Bicep | Bicep DSL | Managed by Azure | Azure only |
| Crossplane | Kubernetes CRDs | Kubernetes etcd | Continuous reconciliation, GitOps-friendly, platform APIs |
| Ansible | YAML playbooks | None | Config management / orchestration |

## IaC in CI/CD

- On PR: `fmt`, `validate`, `tflint`, security scanning (Checkov, Trivy config, tfsec successor), policy as code (OPA/Conftest, Sentinel), `plan` posted as a PR comment (Atlantis, Spacelift, env0, HCP Terraform, GitHub Actions).
- On merge: `apply` the **reviewed plan**, using OIDC credentials scoped per environment.
- Scheduled drift detection: `plan -detailed-exitcode` nightly; alert on changes.
- Cost estimation on PRs (Infracost). See [Cloud Fundamentals](/Software%20Engineering/DevOps/Cloud%20Fundamentals) for FinOps.

<Info>
**🆕 2025–2026**

- **Licensing background**: on 2023-08-10 HashiCorp moved Terraform (from 1.6) to the **Business Source License 1.1**, which forbids competing commercial use. The community forked the last MPL version as **OpenTofu** (Linux Foundation). **IBM completed its acquisition of HashiCorp on 2025-02-27**. OpenTofu was **accepted into the CNCF as a Sandbox project on 2025-04-23**.
- **OpenTofu 1.10 (2025-06-24)**: OCI registry support for providers and modules, **native S3 state locking** (`use_lockfile`, no DynamoDB), experimental OpenTelemetry tracing. **1.11 (2025-12-09)**: ephemeral resources, write-only attributes, `enabled` meta-argument. **1.12 (2026-05-14)**; latest patch 1.12.6 (2026-08-19); **1.13 at RC1** (2026-09-17). OpenTofu has had **state encryption** since 1.7.
- **Terraform 1.14 (2025-11-19)**: `terraform query` with list resources in `*.tfquery.hcl` (find existing infra and generate import config) and a new top-level **`action` block** for provider-defined imperative operations (e.g. invoke a Lambda, invalidate CloudFront). **1.16.0 released 2026-08-26**; 1.17 in beta.
- The two tools remain largely compatible (HCL, state format, provider protocol) but are diverging feature by feature; pick one per codebase and test before switching.
</Info>

## Pitfalls

<Warning>
**Common pitfalls**

- Local state files, or remote state without locking: two applies corrupt state.
- One giant state for everything: slow plans, huge blast radius, everyone blocked on locks.
- Secrets in `.tfvars` committed to Git, or visible in state. Encrypt state, use secret managers, mark variables `sensitive`, use ephemeral/write-only values where supported.
- Clicking in the console to "fix quickly": drift, then the next apply reverts it.
- Unpinned provider/module versions: a minor bump changes behaviour during an incident.
- `terraform apply` without reviewing the plan, especially lines with `-/+` (replace) on databases.
</Warning>

## Senior perspective

- Structure by **blast radius and ownership**: network, shared data, per-service stacks; separate accounts/projects per environment.
- Guardrails over gatekeeping: module library + policy as code + automated plans lets app teams self-serve safely.
- Decide Terraform vs OpenTofu deliberately: licence exposure (are you building a competing product?), needed features (state encryption vs HCP Terraform integrations), vendor support. Keep the decision reversible by avoiding tool-specific features until needed.
- Plan for state disasters: versioned buckets, backups, documented recovery.

## Interview questions

<Accordion title="What is Terraform state and why does it matter?">
A JSON mapping of resources in configuration to real-world objects and their attributes. Terraform uses it to compute diffs. It must be shared (remote backend), locked during operations and protected because it can contain secrets.
</Accordion>

<Accordion title="Terraform vs OpenTofu?">
OpenTofu forked from Terraform 1.5 after the 2023 BSL licence change; it is MPL 2.0 under the Linux Foundation (CNCF sandbox since April 2025). Same HCL and providers; OpenTofu adds features like state encryption and OCI registries, while Terraform has HCP Terraform integration, `terraform query` and actions. Choose based on licence, features and support.
</Accordion>

<Accordion title="How do you handle drift?">
Detect it with scheduled `plan` runs; decide whether code or reality is right; either apply to revert, or update code (and `import` resources created manually). Prevent it with restricted console permissions and GitOps-style workflows.
</Accordion>

<Accordion title="Terraform vs Ansible?">
Terraform provisions and manages the lifecycle of infrastructure resources declaratively with state. Ansible configures systems and orchestrates procedures (install packages, edit files) and is stateless and mostly procedural. They are complementary.
</Accordion>

<Accordion title="How would you structure IaC for 20 teams?">
Shared, versioned modules owned by a platform team; separate state per environment and component; policy as code in CI; plans on PR and applies on merge with OIDC credentials; templates for new services in the developer portal.
</Accordion>

## Related
- [DevOps Index](/Software%20Engineering/DevOps/DevOps%20Index) · [Cloud Fundamentals](/Software%20Engineering/DevOps/Cloud%20Fundamentals) · [CI-CD Pipelines](/Software%20Engineering/DevOps/CI-CD%20Pipelines) · [GitOps - Argo CD & Flux](/Software%20Engineering/DevOps/GitOps%20-%20Argo%20CD%20%26%20Flux) · [Platform Engineering & Internal Developer Platforms](/Software%20Engineering/DevOps/Platform%20Engineering%20%26%20Internal%20Developer%20Platforms)

## Sources
- [HashiCorp: Terraform docs](https://developer.hashicorp.com/terraform/docs)
- [HashiCorp blog: adopting the BSL (2023-08-10)](https://www.hashicorp.com/en/blog/hashicorp-adopts-business-source-license)
- [IBM newsroom: IBM completes acquisition of HashiCorp (2025-02-27)](https://newsroom.ibm.com/2025-02-27-ibm-completes-acquisition-of-hashicorp,-creates-comprehensive,-end-to-end-hybrid-cloud-platform)
- [OpenTofu docs](https://opentofu.org/docs/) · [OpenTofu releases](https://github.com/opentofu/opentofu/releases) · [CNCF project page: OpenTofu](https://www.cncf.io/projects/opentofu/)
- [hashicorp/terraform v1.14.0 release notes](https://github.com/hashicorp/terraform/releases/tag/v1.14.0)
- [Pulumi docs](https://www.pulumi.com/docs/)
- Kief Morris, *Infrastructure as Code* (O'Reilly, 3rd ed.)
