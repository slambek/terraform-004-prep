# 8a. Use HCP Terraform to create infrastructure

> **Scope note:** Workspace/project structure and execution-mode settings
> live in 8c. Policy checks, run tasks, cost estimation, and other
> collaboration/governance features that hook into a run are in 8b.
> Connecting the CLI to HCP Terraform (`cloud` block, `terraform login`,
> state migration, dynamic credentials) is in 8d.

## Core Concept

HCP Terraform is a managed application that runs Terraform for you in a
consistent, shared remote environment instead of on your own workstation.
It organizes work into **workspaces**, executes **runs** (plan then apply)
against them, and layers in review, secrets storage, and policy on top of
the normal Terraform workflow.

## Three ways to trigger runs

| Workflow | How it works |
|---|---|
| **VCS-driven** (primary) | Workspace linked to a repo; merges to the tracked branch auto-start plan+apply; pull requests auto-start a speculative plan and post a check |
| **CLI-driven** | Local `terraform plan`/`apply` execute **remotely** in HCP Terraform, streaming logs back to your terminal, using the CLI integration (`cloud` block) |
| **API-driven** | Any tooling drives runs via the HCP Terraform API — most flexible, most setup required |

VCS integration is optional — without it you can still get remote execution
and other HCP Terraform features via the API or CLI.

## Remote operations

By default (**Execution Mode: Remote**), HCP Terraform runs Terraform on its
own disposable virtual machines rather than your workstation. This is what
unlocks Sentinel/OPA policy checks, cost estimation, notifications, and
consistent run visibility. Execution mode is a workspace/project setting —
see 8c for **Local** and **Agent** modes.

## Runs and workspaces

Every run happens **in the context of a workspace**, which supplies the
configuration, variables, and state — the same role a persistent working
directory plays for local Terraform.

- Each workspace keeps a **queue** of runs and processes them in order; a
  new run stays **pending** until the current one finishes, since an
  in-flight run could change what a later run should do.
- Starting a run **locks** the workspace to a specific configuration
  version and variable set — later variable/code changes only affect
  future runs, not ones already queued/planning/awaiting apply.
- Two operation types **ignore** both the queue and workspace locks:
  **plan-only runs**, and the **planning stage** of saved plan runs
  (applying a saved plan still locks/queues normally).

## Plan and apply

HCP Terraform always plans first, then applies that plan's exact output —
it never re-derives the apply from scratch.

- By default it **waits for user approval** before applying; workspaces can
  be configured to **auto-apply** successful plans instead.
- Some plans can never auto-apply: those queued by run triggers, or queued
  by a user without apply permission for the workspace.
- A plan with **no changes** ends as "Planned and finished" — HCP Terraform
  does not run an apply step for a no-op plan (the **allow empty apply**
  run mode is the explicit override — see below).

## Speculative plans

Plan-only runs used to preview changes during editing/review — they never
apply.

- Don't wait for the run queue, since they can't affect real infrastructure.
- Three triggers: a VCS pull request (posts a check with a link to the
  plan), running `terraform plan` locally with the CLI integration
  configured, or an API run against a configuration version marked
  `speculative`.
- A failed/canceled speculative plan can be **retried** (same configuration
  version) if you have permission to queue plans for the workspace.

## Saved plans

*Requires Terraform CLI ≥ 1.6.0 for the CLI workflow.*

```bash
terraform plan -out <FILE>
terraform apply <FILE>
terraform show <FILE>       # inspect before applying
```

- Unlike normal runs, the **planning stage** skips the run queue (like
  plan-only runs); **applying** it still requires the workspace be
  unlocked and behaves like a normal queued/locking run.
- **Never auto-applies**, even if the workspace has auto-apply enabled —
  it only applies on manual confirmation.
- HCP Terraform **automatically discards** a saved plan if another run
  applies (or state otherwise changes) before you confirm it, or if it
  sits unconfirmed for a few weeks.

## Other run modes

Selectable via CLI flag, API option, or UI run-type dropdown:

| Mode | Purpose |
|---|---|
| Destroy | Plans to destroy all objects regardless of config |
| Refresh-only | Updates state to match real infra without changing infra (see 6d for the CLI-level mechanics) |
| Allow empty apply | Lets a no-op plan still apply — used to force a **state version upgrade**, e.g. after bumping the workspace's Terraform version |
| Replace | Forces destroy+recreate of specific resources (`-replace=ADDRESS`) |
| Targeted | Restrict the plan to specific resource addresses — exceptional use only |

## Workspace Terraform version

Each workspace has an assigned Terraform version used for **all** its
remote runs, ensuring consistent behavior. New workspaces default to the
latest version; a workspace migrated from local state inherits the CLI
version used at migration time. You can:

- Change the workspace's version anytime in its settings.
- Override the version **just for a speculative plan** to test upgrade
  compatibility before committing to a workspace-wide change.
- After bumping a major version that changes the state file format, run an
  **allow-empty-apply** run to upgrade the stored state version even when
  there are no infrastructure changes to make.

## Exam Quick Facts

- VCS-driven is the primary/default workflow; CLI-driven and API-driven are
  the alternatives when you don't want a linked repo.
- A workspace processes runs **strictly in order** — new runs wait as
  "pending" unless they're plan-only or a saved-plan's planning stage.
- Speculative plans **can never be applied** — they exist purely to preview
  changes and run policy checks during review.
- A saved plan is auto-discarded if the workspace's state changes before
  you confirm it, or after sitting too long unconfirmed.
- No-op plans don't auto-apply by themselves — **allow empty apply** is the
  specific run mode that forces an apply anyway, mainly for state-version
  upgrades.
- Locking a workspace pins a run to a specific config version + variable
  values; changing either afterward affects only *future* runs.
