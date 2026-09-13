# HashiCorp Certified: Terraform Associate (004) — Prep Notes

Prep repository for the **HashiCorp Certified: Terraform Associate (004)** certification.

All objectives across sections 1–8 are covered. Notes are organized as:
**one `.md` file in `notes/en/` = one exam objective**, with explicit scope
notes where topics overlap, so facts live in one place instead of being
scattered across multiple files.

## Repository structure

```text
.
├── README.en.md          # English README
├── README.ru.md          # Russian README
├── CHEATSHEET.en.md      # English condensed cram sheet
├── CHEATSHEET.ru.md      # Russian condensed cram sheet
└── notes/
    ├── en/               # English notes
    │   ├── 01a-what-is-iac.md
    │   ├── 01b-advantages-of-iac.md
    │   ├── ...
    │   └── 08d-cli-integration.md
    └── ru/               # Russian notes
        ├── 01a-what-is-iac.md
        ├── 01b-advantages-of-iac.md
        ├── ...
        └── 04h-sensitive-data-vault.md
```

## Sections and objectives

| Objective                                                                                   | File                                                                                              |
| ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| 1a. Explain what IaC is                                                                     | [01a-what-is-iac.md](notes/en/01a-what-is-iac.md)                                                 |
| 1b. Describe advantages of IaC patterns                                                     | [01b-advantages-of-iac.md](notes/en/01b-advantages-of-iac.md)                                     |
| 1c. Explain how Terraform manages multi-cloud, hybrid cloud, and service-agnostic workflows | [01c-multicloud-hybrid.md](notes/en/01c-multicloud-hybrid.md)                                     |
| 2a. Install and version Terraform providers                                                 | [02a-install-providers.md](notes/en/02a-install-providers.md)                                     |
| 2b. Describe how Terraform uses providers                                                   | [02b-provider-usage.md](notes/en/02b-provider-usage.md)                                           |
| 2c. Write Terraform configuration using multiple providers                                  | [02c-multiple-providers.md](notes/en/02c-multiple-providers.md)                                   |
| 2d. Explain how Terraform uses and manages state                                            | [02d-state-fundamentals.md](notes/en/02d-state-fundamentals.md)                                   |
| 3a. Describe the Terraform workflow                                                         | [03a-terraform-workflow.md](notes/en/03a-terraform-workflow.md)                                   |
| 3b. Initialize a Terraform working directory                                                | [03b-terraform-init.md](notes/en/03b-terraform-init.md)                                           |
| 3c. Validate a Terraform configuration                                                      | [03c-terraform-validate.md](notes/en/03c-terraform-validate.md)                                   |
| 3d. Generate and review an execution plan for Terraform                                     | [03d-terraform-plan.md](notes/en/03d-terraform-plan.md)                                           |
| 3e. Apply changes to infrastructure with Terraform                                          | [03e-terraform-apply.md](notes/en/03e-terraform-apply.md)                                         |
| 3f. Destroy Terraform-managed infrastructure                                                | [03f-terraform-destroy.md](notes/en/03f-terraform-destroy.md)                                     |
| 3g. Apply formatting and style adjustments to a configuration                               | [03g-terraform-fmt.md](notes/en/03g-terraform-fmt.md)                                             |
| 4a. Use and differentiate `resource` and `data` blocks                                      | [04a-resource-vs-data-blocks.md](notes/en/04a-resource-vs-data-blocks.md)                         |
| 4b. Refer to resource attributes and create cross-resource references                       | [04b-resource-references.md](notes/en/04b-resource-references.md)                                 |
| 4c. Use variables and outputs                                                               | [04c-variables-and-outputs.md](notes/en/04c-variables-and-outputs.md)                             |
| 4d. Understand and use complex types                                                        | [04d-complex-types.md](notes/en/04d-complex-types.md)                                             |
| 4e. Write dynamic configuration using expressions and functions                             | [04e-dynamic-expressions-functions.md](notes/en/04e-dynamic-expressions-functions.md)             |
| 4f. Define resource dependencies in configuration                                           | [04f-resource-dependencies.md](notes/en/04f-resource-dependencies.md)                             |
| 4g. Validate configuration using custom conditions                                          | [04g-validate-custom-conditions.md](notes/en/04g-validate-custom-conditions.md)                   |
| 4h. Manage sensitive data, including secrets management with Vault                          | [04h-sensitive-data-vault.md](notes/en/04h-sensitive-data-vault.md)                               |
| 5a. Explain how Terraform sources modules                                                   | [05a-module-sources.md](notes/en/05a-module-sources.md)                                           |
| 5b. Describe variable scope within modules                                                  | [05b-variable-scope-in-modules.md](notes/en/05b-variable-scope-in-modules.md)                     |
| 5c. Use modules in configuration                                                            | [05c-using-modules.md](notes/en/05c-using-modules.md)                                             |
| 5d. Manage module versions                                                                  | [05d-module-versions.md](notes/en/05d-module-versions.md)                                         |
| 6a. Describe the local backend                                                              | [06a-local-backend.md](notes/en/06a-local-backend.md)                                             |
| 6b. Describe state locking                                                                  | [06b-state-locking.md](notes/en/06b-state-locking.md)                                             |
| 6c. Configure remote state using the backend block                                          | [06c-remote-state-backend-block.md](notes/en/06c-remote-state-backend-block.md)                   |
| 6d. Manage resource drift and Terraform state                                               | [06d-resource-drift-state-mgmt.md](notes/en/06d-resource-drift-state-mgmt.md)                     |
| 7a. Import existing infrastructure into your Terraform workspace                            | [07a-import-existing-infrastructure.md](notes/en/07a-import-existing-infrastructure.md)           |
| 7b. Use the CLI to inspect state                                                            | [07b-cli-inspect-state.md](notes/en/07b-cli-inspect-state.md)                                     |
| 7c. Describe when and how to use verbose logging                                            | [07c-verbose-logging.md](notes/en/07c-verbose-logging.md)                                         |
| 8a. Use HCP Terraform to create infrastructure                                              | [08a-hcp-terraform-create-infrastructure.md](notes/en/08a-hcp-terraform-create-infrastructure.md) |
| 8b. Describe HCP Terraform collaboration and governance features                            | [08b-collaboration-governance.md](notes/en/08b-collaboration-governance.md)                       |
| 8c. Describe how to organize and use HCP Terraform workspaces and projects                  | [08c-workspaces-and-projects.md](notes/en/08c-workspaces-and-projects.md)                         |
| 8d. Configure and use HCP Terraform integration                                             | [08d-cli-integration.md](notes/en/08d-cli-integration.md)                                         |

## How to use this

* **`CHEATSHEET.en.md`** — for quick review on exam day. Only the facts that
  actually trip people up on the test: edge cases, defaults, "X never
  does Y", distinctions between similar concepts. Grouped by topic rather
  than by file — duplication across objectives has been removed.
* **`notes/en/`** — for deep-dive study in English. Each file covers exactly
  one objective and includes a `> Scope note` where a topic overlaps with
  another objective — this points you to where the full mechanics live if
  something looks trimmed down.
* **`notes/ru/`** — Russian translations of the available notes.

Inside each notes file: **Core Concept** → objective details → **Exam
Quick Facts** at the end — a condensed list of that file's specific
gotchas (in `CHEATSHEET.en.md` these lists are merged together and
deduplicated).