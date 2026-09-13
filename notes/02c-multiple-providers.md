# 2c. Write Terraform configuration using multiple providers

## Core Concept

A single Terraform configuration can use **more than one provider at the same time** — for example `aws` + `random`, or `aws` + `cloudflare` + `github`. Terraform Core doesn't care how many providers are involved; it builds one **resource graph** across all of them and resolves dependencies automatically, even across provider boundaries.

**Important distinction (exam trap):**

* **2b (aliases)** → multiple **configurations of the *same* provider** (`aws` + `aws.west`).
* **2c (this note)** → multiple **different providers** used together in one configuration (`aws` + `random`).

These are separate concepts that are easy to mix up on the exam.

## Declaring Multiple Providers

Each provider you use must be listed in `required_providers`, inside the top-level `terraform` block:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}
```

* Each provider has its **own independent version constraint** (details in 2a).
* Each provider gets its **own `provider` block** if it needs configuration (details in 2b). `random` needs no configuration, so `provider "random" {}` can be omitted entirely — Terraform assumes an empty default configuration.

```hcl
provider "aws" {
  region = "us-west-2"
}

provider "random" {}
```

## Cross-Provider Resource References

Resources from different providers can reference each other's attributes. Terraform detects this and adds it as an **implicit dependency** in the resource graph, regardless of which provider owns which resource.

```hcl
resource "random_pet" "name" {
  length = 2
}

resource "aws_instance" "web" {
  ami           = "ami-a0cfeed8"
  instance_type = "t2.micro"

  tags = {
    Name = random_pet.name.id   # cross-provider reference: random -> aws
  }
}
```

* `random_pet.name` belongs to the `random` provider.
* `aws_instance.web` belongs to the `aws` provider.
* Terraform will create `random_pet.name` **first**, because `aws_instance.web` depends on its output.

## How Terraform Resolves the Local (Provider) Name

Terraform infers which provider implements a resource type from the **prefix of the resource type name**:

```text
aws_instance      → prefix "aws"     → aws provider
random_pet        → prefix "random"  → random provider
google_compute_*  → prefix "google"  → google provider
```

This is why using a provider's **preferred local name** matters (see 2a/2b): it lets Terraform infer the provider automatically without needing the `provider = ...` meta-argument on every resource.

## Multi-Cloud / Multi-Service Example

Nothing changes conceptually when the providers represent different cloud platforms or SaaS tools — the same `required_providers` + `provider` block pattern applies to each one:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

provider "cloudflare" {
  api_token = var.cloudflare_token
}
```

Terraform's single resource graph coordinates dependency order across `aws` and `cloudflare` resources exactly as it did with `aws` and `random` above. This is the mechanism referenced in 1c ("Multi-Cloud Formula").

## Provider Source Tiers (Terraform Registry)

When adding a new provider to a configuration, its Registry badge tells you who maintains it — useful context when picking providers for a real config:

| Tier | Maintained by | Namespace example |
| :--- | :--- | :--- |
| **Official** | HashiCorp | `hashicorp`, `ansible` |
| **Partner Premier** | Vetted technology partner | Third-party org |
| **Partner** | Technology partner | Third-party org |
| **Community** | Individual/community maintainers | Maintainer's account |
| **Archived** | No longer maintained | `hashicorp` or third-party |

## Built-in Provider (No Declaration Needed)

Terraform ships with exactly **one built-in provider**: it powers the `terraform_remote_state` data source. Its source address is `terraform.io/builtin/terraform`. Because it's built into Terraform Core, it does **not** need a `required_providers` entry — unlike every other provider used in a configuration.

## Exam Quick Facts

* A configuration can freely mix any number of **different** providers.
* Each provider is declared independently in `required_providers` with its own `source` and `version`.
* Terraform automatically resolves dependencies **across** providers via the resource graph — no special syntax needed for cross-provider references.
* Terraform infers the provider for a resource from the resource type's prefix (`aws_*` → `aws`).
* Mixing multiple **different** providers (2c) ≠ multiple **configurations of one** provider via `alias` (2b).
* The built-in `terraform` provider (for `terraform_remote_state`) is the one exception that needs no `required_providers` entry.