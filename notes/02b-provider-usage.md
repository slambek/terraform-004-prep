# 2b. Describe how Terraform uses providers

> **Scope note:** provider configurations *inside modules* (passing providers to child modules, `configuration_aliases`) belong to **Section 5 (Modules)** — that's where they'll be covered, since they need module context to make sense. This note stays focused on providers at the root-module level.

## 1. Provider Plugin Architecture

Terraform has two parts:

```text
Terraform Configuration
        ↓
Terraform Core   (config, state, dependency graph, plan, apply)
        ↓ RPC
Provider Plugin  (auth, API calls, resources, data sources)
        ↓
Cloud / SaaS / API
```

Terraform Core has no built-in knowledge of AWS, Azure, Kubernetes, etc. — that knowledge lives entirely in the provider plugin, and Core talks to it over RPC.

## 2. What a Provider Implements

* **Resources** — let Terraform create/manage infrastructure (`resource "aws_instance" "example" {}`). Every resource type is implemented by exactly one provider.
* **Data sources** — let Terraform *read* existing information instead of creating anything (`data "aws_ami" "example" {}`).

```text
resource → creates / manages
data     → reads only
```

## 3. Provider Configuration (the `provider` block)

The `provider` block configures **how** Terraform uses an already-declared provider (region, endpoint, auth, etc.) — it does **not** install the provider or set its version (that's `required_providers`, see 2a).

```hcl
provider "aws" {
  region = var.region   # can reference variables / other known-before-apply values
}
```

Credentials are often better supplied via environment variables or other external mechanisms than hardcoded in config, to keep secrets out of version control.

## 4. Provider Aliases — Multiple Configs of the *Same* Provider

Use `alias` when you need more than one configuration of the same provider (e.g. two AWS regions):

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}
```

Select the aliased configuration explicitly on a resource:

```hcl
resource "aws_instance" "example" {
  provider = aws.west
}
```

*(Contrast with 2c: aliases = multiple configs of **one** provider; 2c = multiple **different** providers in one config — easy to confuse on the exam.)*

## 5. Default Configuration & the Implied Empty Default

A `provider` block **without** `alias` is the default configuration — any resource that doesn't set `provider = ...` uses it.

If **every** configuration of a provider has an alias, there's no explicit default, so Terraform creates an **implied empty default configuration**. A resource with no `provider` meta-argument would then try to use that empty config — which errors out if the provider has required arguments.

```text
provider "aws" { alias = "east" ... }
provider "aws" { alias = "west" ... }
        ↓
aws  →  implied empty default (dangerous if aws needs required args)
```

## Exam Quick Facts

* Terraform Core ↔ Provider Plugin communicate over **RPC**; Core has no service-specific logic of its own.
* Every resource type belongs to exactly one provider; a **resource** manages, a **data source** only reads.
* The `provider` block configures usage (region, auth, etc.) — it never sets version or installs anything.
* No `alias` → default configuration. Every config aliased → implied *empty* default (can error if the provider requires arguments).
* Provider-config-inside-modules is a separate topic — see Section 5.