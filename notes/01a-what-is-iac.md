# 1a. Explain what IaC is

## Definition
Infrastructure as Code (IaC) is the process of managing and provisioning 
computer data centers through machine-readable definition files, rather 
than physical hardware configuration or interactive configuration tools.

## Key Benefits
1. **Speed & Safety:** Automated provisioning reduces human error and speeds up releases.
2. **Consistency (Idempotency):** Executing the same code results in the exact same infrastructure.
3. **Version Control:** Infrastructure changes are tracked in Git (auditing, rollbacks, code reviews).
4. **Reusability:** Easily duplicate environments (dev, stage, prod).

## Declarative vs Imperative
- **Imperative (Bash/Ansible):** Focuses on HOW to achieve a state (step-by-step).
- **Declarative (Terraform):** Focuses on WHAT the final state should look like.