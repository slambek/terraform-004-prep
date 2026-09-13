# HashiCorp Certified: Terraform Associate (004) — Prep Notes

Personal prep repository for the **HashiCorp Certified: Terraform
Associate (004)** certification.

All objectives across sections 1–8 are covered. Notes are organized as:
**one `.md` file in `notes/` = one exam objective**, with explicit scope
notes where topics overlap, so facts live in one place instead of being
scattered across multiple files.

## Repository structure

```text
.
├── README.md          # this file
├── CHEATSHEET.md       # condensed cram sheet for exam day
└── notes/              # detailed breakdown, one file = one objective
    ├── 01a-what-is-iac.md
    ├── 01b-advantages-of-iac.md
    ├── ...
    └── 08d-cli-integration.md
```

## Sections and objectives

| ✓ | Objective | File |
| :---: | --- | --- |
| [x] | 1a. Explain what IaC is | [01a-what-is-iac.md](notes/01a-what-is-iac.md) |
| [x] | 1b. Describe advantages of IaC patterns | [01b-advantages-of-iac.md](notes/01b-advantages-of-iac.md) |
| [x] | 1c. Explain how Terraform manages multi-cloud, hybrid cloud, and service-agnostic workflows | [01c-multicloud-hybrid.md](notes/01c-multicloud-hybrid.md) |
| [x] | 2a. Install and version Terraform providers | [02a-install-providers.md](notes/02a-install-providers.md) |
| [x] | 2b. Describe how Terraform uses providers | [02b-provider-usage.md](notes/02b-provider-usage.md) |
| [x] | 2c. Write Terraform configuration using multiple providers | [02c-multiple-providers.md](notes/02c-multiple-providers.md) |
| [x] | 2d. Explain how Terraform uses and manages state | [02d-state-fundamentals.md](notes/02d-state-fundamentals.md) |
| [x] | 3a. Describe the Terraform workflow | [03a-terraform-workflow.md](notes/03a-terraform-workflow.md) |
| [x] | 3b. Initialize a Terraform working directory | [03b-terraform-init.md](notes/03b-terraform-init.md) |
| [x] | 3c. Validate a Terraform configuration | [03c-terraform-validate.md](notes/03c-terraform-validate.md) |
| [x] | 3d. Generate and review an execution plan for Terraform | [03d-terraform-plan.md](notes/03d-terraform-plan.md) |
| [x] | 3e. Apply changes to infrastructure with Terraform | [03e-terraform-apply.md](notes/03e-terraform-apply.md) |
| [x] | 3f. Destroy Terraform-managed infrastructure | [03f-terraform-destroy.md](notes/03f-terraform-destroy.md) |
| [x] | 3g. Apply formatting and style adjustments to a configuration | [03g-terraform-fmt.md](notes/03g-terraform-fmt.md) |
| [x] | 4a. Use and differentiate `resource` and `data` blocks | [04a-resource-vs-data-blocks.md](notes/04a-resource-vs-data-blocks.md) |
| [x] | 4b. Refer to resource attributes and create cross-resource references | [04b-resource-references.md](notes/04b-resource-references.md) |
| [x] | 4c. Use variables and outputs | [04c-variables-and-outputs.md](notes/04c-variables-and-outputs.md) |
| [x] | 4d. Understand and use complex types | [04d-complex-types.md](notes/04d-complex-types.md) |
| [x] | 4e. Write dynamic configuration using expressions and functions | [04e-dynamic-expressions-functions.md](notes/04e-dynamic-expressions-functions.md) |
| [x] | 4f. Define resource dependencies in configuration | [04f-resource-dependencies.md](notes/04f-resource-dependencies.md) |
| [x] | 4g. Validate configuration using custom conditions | [04g-validate-custom-conditions.md](notes/04g-validate-custom-conditions.md) |
| [x] | 4h. Manage sensitive data, including secrets management with Vault | [04h-sensitive-data-vault.md](notes/04h-sensitive-data-vault.md) |
| [x] | 5a. Explain how Terraform sources modules | [05a-module-sources.md](notes/05a-module-sources.md) |
| [x] | 5b. Describe variable scope within modules | [05b-variable-scope-in-modules.md](notes/05b-variable-scope-in-modules.md) |
| [x] | 5c. Use modules in configuration | [05c-using-modules.md](notes/05c-using-modules.md) |
| [x] | 5d. Manage module versions | [05d-module-versions.md](notes/05d-module-versions.md) |
| [x] | 6a. Describe the local backend | [06a-local-backend.md](notes/06a-local-backend.md) |
| [x] | 6b. Describe state locking | [06b-state-locking.md](notes/06b-state-locking.md) |
| [x] | 6c. Configure remote state using the backend block | [06c-remote-state-backend-block.md](notes/06c-remote-state-backend-block.md) |
| [x] | 6d. Manage resource drift and Terraform state | [06d-resource-drift-state-mgmt.md](notes/06d-resource-drift-state-mgmt.md) |
| [x] | 7a. Import existing infrastructure into your Terraform workspace | [07a-import-existing-infrastructure.md](notes/07a-import-existing-infrastructure.md) |
| [x] | 7b. Use the CLI to inspect state | [07b-cli-inspect-state.md](notes/07b-cli-inspect-state.md) |
| [x] | 7c. Describe when and how to use verbose logging | [07c-verbose-logging.md](notes/07c-verbose-logging.md) |
| [x] | 8a. Use HCP Terraform to create infrastructure | [08a-hcp-terraform-create-infrastructure.md](notes/08a-hcp-terraform-create-infrastructure.md) |
| [x] | 8b. Describe HCP Terraform collaboration and governance features | [08b-collaboration-governance.md](notes/08b-collaboration-governance.md) |
| [x] | 8c. Describe how to organize and use HCP Terraform workspaces and projects | [08c-workspaces-and-projects.md](notes/08c-workspaces-and-projects.md) |
| [x] | 8d. Configure and use HCP Terraform integration | [08d-cli-integration.md](notes/08d-cli-integration.md) |

**Progress: 8 / 8 sections, 37 / 37 objectives.**

## How to use this

* **`CHEATSHEET.md`** — for quick review on exam day. Only the facts that
  actually trip people up on the test: edge cases, defaults, "X never
  does Y", distinctions between similar concepts. Grouped by topic rather
  than by file — duplication across objectives has been removed.
* **`notes/`** — for deep-dive study. Each file covers exactly one
  objective and includes a `> Scope note` where a topic overlaps with
  another objective — this points you to where the full mechanics live if
  something looks trimmed down.

Inside each notes file: **Core Concept** → objective details → **Exam
Quick Facts** at the end — a condensed list of that file's specific
gotchas (in `CHEATSHEET.md` these lists are merged together and
deduplicated).