# Terraform Associate (004) High-Yield Cheatsheet

## ⚡ Core Philosophy & Exam Triggers

| Concept | Exam Trigger / Keyword | Key Definition |
| :--- | :--- | :--- |
| **Declarative** | *WHAT, not HOW* | You specify desired end-state; Terraform handles creation steps. |
| **Imperative** | *HOW (step-by-step)* | Bash/Python scripts specifying explicit procedural steps. |
| **Idempotency** | *Same result every time* | Executing the code N times produces the exact same state without side effects. |
| **Immutable** | *Replace > Modify* | Resources are destroyed and recreated instead of modified in-place. |
| **State File** | *Single Source of Truth* | Maps HCL code configuration to real-world infrastructure resources. |
| **Resource Graph** | *Dependency resolution* | Determines resource dependencies and provisions non-dependent resources in parallel. |
| **Provider** | *Plugins / APIs* | Translates HCL code into target service/cloud API calls. |

---

## 🛠 Essential CLI Commands

```bash
# Workflow Commands
terraform init          # Downloads providers, initializes backends, creates .terraform folder
terraform plan          # Previews changes (diff between code, state, and real cloud)
terraform apply         # Executes planned changes and updates the state file
terraform destroy       # Destroys all resources managed by the current state file

# Maintenance & Format
terraform fmt           # Formats code into standard HCL canonical style
terraform validate      # Checks syntax, internal consistency, and required attributes
terraform show          # Displays human-readable output of state file or plan file
terraform state list    # Lists all resources currently tracked in the state file

# Troubleshooting
terraform refresh       # Updates local state with real-world infrastructure (deprecated in favor of apply -refresh-only)

```

---

## 🎯 Section 1 Quick Recall (IaC & Workflows)

* **Day 0 vs Day 1:**
* **Day 0 (Provisioning):** VPCs, VMs, Storage, Databases $\rightarrow$ **Terraform**
* **Day 1+ (Configuration/Ops):** OS updates, App deployments $\rightarrow$ **Ansible / Docker / Chef**


* **Multi-Cloud Formula:** `Providers` + `Single HCL Syntax` + `Resource Graph` + `Unified CLI Workflow`
* **Sentinel:** Policy-as-Code framework in HCP Terraform / Enterprise evaluated **before `apply**`.