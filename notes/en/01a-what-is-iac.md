# 1a. Explain what IaC is

## Core Concept

Infrastructure as Code (IaC) manages infrastructure using human-readable configuration files instead of manual GUI/CLI interactions.

**IaC allows you to build, change, and manage infrastructure in a safe, consistent, and repeatable way.**

## Key Exam Concepts

* **Declarative vs Imperative:**

  * **Declarative (Terraform):** You define **WHAT** the desired end-state should be.
  * **Imperative (Bash, Python CLI):** You define **HOW** to achieve a state via step-by-step commands.

* **Vendor-Agnostic / Multi-Cloud:**

  * Terraform uses a single configuration language (HCL) and consistent workflow across multiple providers.
  * Examples: AWS, GCP, Azure, Kubernetes.
  * Cloud-native tools are generally tied to their respective platforms (for example, CloudFormation for AWS).

* **Day 0 vs Day 1+:**

  * **Day 0 (Provisioning):** Initial infrastructure setup — VPCs, VMs, databases, storage.
  * **Day 1+ (Configuration/Ops):** Subsequent configuration and operational changes — OS updates, patches, application configuration.
  * Terraform can participate across the infrastructure lifecycle, while tools such as Ansible or Chef are commonly used for configuration management.

* **Idempotency:**

  * Re-running the same desired configuration produces the same intended end state without unnecessary changes.
  * This makes infrastructure changes consistent, repeatable, and predictable.

* **Immutable Infrastructure:**

  * Infrastructure components can be replaced rather than modified in-place.
  * Terraform takes an immutable approach to infrastructure.

* **State File:**

  * Terraform uses the state file to track real infrastructure.
  * State helps Terraform determine what changes are required to make real infrastructure match the configuration.

## Core Workflow

1. **Scope** → Identify the required infrastructure.
2. **Author (Write)** → Define resources in declarative HCL files (`.tf`).
3. **Initialize (`terraform init`)** → Initialize the working directory and install required providers/plugins and configure the backend.
4. **Plan (`terraform plan`)** → Preview the changes Terraform intends to make.
5. **Apply (`terraform apply`)** → Execute the planned changes in the correct dependency order.