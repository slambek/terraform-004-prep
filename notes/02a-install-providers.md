# 2a. Install and version Terraform providers

## Core Concept

**Terraform providers** are plugins that allow Terraform to interact with cloud platforms, SaaS services, and other APIs.

Providers are **versioned separately from Terraform** and are primarily distributed through the **Terraform Registry**.

Terraform must know:

* **which provider** is required,
* **where to get it**,
* **which versions are allowed**.

## Key Exam Concepts

* **`required_providers`:**

  * Declares the providers a module requires.
  * Must be inside the top-level **`terraform`** block.
  * A provider requirement contains:

    * **Local name** — name used inside the configuration.
    * **Source address** — identifies where the provider comes from.
    * **Version constraint** — defines which versions are allowed.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.5.0"
    }
  }
}
```

* **Provider source address:**

  * Identifies the provider and its source.
  * Format:

```text
[hostname/]namespace/type
```

* Example:

```text
hashicorp/aws
```

* If the hostname is omitted, Terraform uses **`registry.terraform.io`**.

* **Local provider name:**

  * The name used to reference the provider in the configuration.
  * Example:

```hcl
aws = {
  source  = "hashicorp/aws"
  version = ">= 4.5.0"
}
```

* `aws` → **local name**

* `hashicorp/aws` → **source address**

* **Version constraints:**

  * Define which provider versions Terraform is allowed to use.

  * The `version` argument belongs in **`required_providers`**.

  * **Minimum version:**

```hcl
version = ">= 1.0"
```

Allows `1.0` and newer versions.

* **Pessimistic constraint:**

```hcl
version = "~> 1.0.4"
```

Allows compatible patch-level updates within the `1.0` minor release.

* **Exact version:**

```hcl
version = "3.1.0"
```

Only `3.1.0` is allowed.

* **`provider` block:**

  * Configures an already-declared provider.
  * Used for provider-specific settings such as region or endpoint.

```hcl
provider "aws" {
  region = "us-west-2"
}
```

**Important distinction:**

* `required_providers` → **which provider and which versions**
* `provider` block → **how the provider is configured**

**Exam trap:** Do not put `version` inside the `provider` block. Version constraints belong in `required_providers`.

## Core Workflow

1. **Declare the provider**

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

2. **Configure the provider**

```hcl
provider "aws" {
  region = "us-west-2"
}
```

3. **Initialize Terraform**

```bash
terraform init
```

`terraform init`:

* finds required providers,
* selects provider versions according to the constraints and lock file,
* downloads and installs providers,
* creates or updates **`.terraform.lock.hcl`**.

4. **Use Terraform**

```bash
terraform plan
terraform apply
```

Terraform uses the provider version selected in the lock file.

5. **Upgrade providers intentionally**

```bash
terraform init -upgrade
```

This searches for newer provider versions that satisfy the configured version constraints and updates the lock file.

## Dependency Lock File

**`.terraform.lock.hcl`** records the **specific provider versions selected by Terraform** and their **checksums**.

* **Version constraint** → defines which versions are **allowed**.
* **Lock file** → records which version Terraform has **selected**.

Example:

```hcl
version = ">= 4.5.0"
```

This allows `4.5.0` or newer.

If `.terraform.lock.hcl` records:

```text
4.5.0
```

then normal:

```bash
terraform init
```

will normally continue using `4.5.0`.

To intentionally consider newer allowed versions:

```bash
terraform init -upgrade
```

**Commit `.terraform.lock.hcl` to version control.**

The lock file also contains **checksums** that Terraform uses to verify provider packages.

## Exam Quick Facts

* **Version constraint** (`required_providers`) = what's **allowed**. **Lock file** = what's actually **selected**.
* `version` never goes inside the `provider` block — only inside `required_providers`.
* `terraform init` installs providers and writes/updates `.terraform.lock.hcl`; `terraform init -upgrade` is the only thing that reconsiders newer allowed versions.
* Commit `.terraform.lock.hcl` to version control.