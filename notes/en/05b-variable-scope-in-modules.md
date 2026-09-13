# 5b. Describe variable scope within modules

> **Scope note:** the `variable`/`output`/`locals` block syntax itself (arguments, `sensitive`, `validation`, etc.) is covered fully in **4c**. This note is specifically about *what crosses a module boundary and how*.

## Each Module Has Its Own Isolated Namespace

Variables, locals, and resource names declared in one module are **not visible** in another module — including between a parent and its child. A child module cannot see the parent's `var.*` or `local.*` values just because it's "nested" inside the same configuration; every module only sees:

* its **own** `variable` declarations (populated by whatever the caller passes in),
* its **own** `locals`,
* its **own** resource/data references,
* and its **own** child modules' outputs (via `module.<name>.<output>`).

## How Values Flow *In*: Module-Specific Inputs

A module's `variable` blocks define its input interface. The **caller** supplies values as arguments directly inside the `module` block — these are matched by name to the callee's declared variables:

```hcl
# child module declares:
variable "bucket_name" { type = string }

# caller supplies it as a module argument:
module "website_s3_bucket" {
  source      = "./modules/aws-s3-static-website-bucket"
  bucket_name = "my-unique-bucket-2024"
}
```

Variables without a `default` become **required arguments** on the module block — omitting them is an error. This is exactly analogous to how root-module variables work (4c) — the only difference is *where* the value comes from (a `module` block argument, instead of `-var`/`.tfvars`/env vars).

## How Values Flow *Out*: Outputs Only

The **only** way data leaves a module is through its `output` blocks. A parent references a child's output as `module.<LABEL>.<OUTPUT_NAME>`:

```hcl
output "vpc_public_subnets" {
  value = module.vpc.public_subnets
}
```

If a module doesn't explicitly output a value, nothing about that resource is visible to the caller — not even indirectly. This is why designing a module's outputs deliberately matters as much as designing its inputs.

## Provider Configuration Is a Special Case: Implicit Inheritance

Unlike variables, **provider configuration is not passed as a normal input** — a child module implicitly **inherits** the default provider configuration from its caller automatically. This is why **child modules should generally not contain their own `provider` blocks** — let the root module configure providers, and every descendant module reuses that same configuration.

To use a **non-default (aliased)** provider configuration inside a child module, the parent must explicitly map it via the `providers` meta-argument on the `module` block, and the child module must declare it expects that alias via `configuration_aliases` in its own `required_providers` block (see 2b for alias mechanics):

```hcl
# root module
provider "aws" {
  alias  = "usw1"
  region = "us-west-1"
}

module "tunnel" {
  source    = "./tunnel"
  providers = {
    aws.src = aws.usw1
  }
}
```

## Design Pattern: Dependency Inversion

Because a module can't see anything outside its own declared variables, well-designed modules **accept their dependencies as input variables** rather than creating those dependencies internally. For example, a `consul_cluster` module needing a VPC and subnets should take `vpc_id` and `subnet_ids` as variables, rather than creating its own VPC — this keeps it usable alongside other infrastructure sharing the same network, and lets the caller swap in the ID from a resource, a data source, or a completely different module later without changing `consul_cluster` at all.

```hcl
module "consul_cluster" {
  source     = "./modules/aws-consul-cluster"
  vpc_id     = module.network.vpc_id
  subnet_ids = module.network.subnet_ids
}
```

## Data-Only Modules

A module doesn't need any `resource` blocks at all — a module containing only `data` sources (see 4a) is valid, and useful for encapsulating **how** some piece of shared information is retrieved (e.g. querying a shared network's IDs), so that callers don't need to know or care whether that data comes from a live API query, a `terraform_remote_state` lookup, or something else. The scoping rules are identical: only what the data-only module explicitly outputs is visible to its caller.

## Exam Quick Facts

* A module cannot see another module's variables/locals — only its **own**, plus whatever it explicitly declares as `variable` inputs and imports via `module.<name>.<output>`.
* Values enter a module **only** as arguments on the `module` block, matched to that module's declared variables.
* Values leave a module **only** through `output` blocks — nothing implicit.
* Provider configuration is the exception: it's **implicitly inherited** from the caller by default; explicit aliasing requires both the `providers` meta-argument *and* `configuration_aliases` in the child.
* Best practice: child modules should generally **not** declare their own `provider` blocks.
* Dependency inversion = accept dependencies as variables instead of creating them internally — keeps modules composable.
