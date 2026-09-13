# 3e. Apply changes to infrastructure with Terraform

## Two Modes of `terraform apply`

| Mode | How | Behavior |
| :--- | :--- | :--- |
| **Automatic plan mode** | `terraform apply` (no file argument) | Generates a new plan (as if running `terraform plan`), **prompts for approval**, then executes it. Supports all the same planning modes/options as `plan` (3d) — `-destroy`, `-refresh-only`, `-replace`, `-target`, `-var`, `-var-file`, etc. |
| **Saved plan mode** | `terraform apply tfplan` | Executes the **exact** actions in a previously saved plan file — **no confirmation prompt** (passing the file *is* the approval). You cannot add extra planning modes/options here; the plan file already locked those decisions in. |

`-auto-approve` skips the interactive prompt in automatic plan mode (ignored/unnecessary with a saved plan file, since that never prompts anyway). ⚠️ Using `-auto-approve` means no human reviews the plan before it executes — recommended only when you're also certain nothing outside Terraform can change the infrastructure concurrently.

## What Happens During an Apply

1. **Locks** the workspace's state — prevents concurrent Terraform runs against the same state. If a lock already exists, Terraform errors out immediately rather than proceeding.
2. **Creates a plan** and waits for approval (or uses the saved plan file, skipping the prompt).
3. **Executes** the plan's steps using the installed providers — in **parallel where possible**, **sequentially where one resource depends on another** (see the dependency graph in 3d).
4. **Updates state** with a snapshot of the new resource state.
5. **Unlocks** state.
6. **Reports** the changes made and any output values.

## Error Handling During Apply

If an error occurs mid-apply, Terraform:

1. Logs and reports the error.
2. **Updates the state file** with whatever changes did succeed before the error.
3. Unlocks state.
4. Exits.

**Terraform does not auto-rollback a partially-completed apply.** Your infrastructure may be left in a state that's a mix of old and new — you must resolve the underlying issue and re-apply to reconcile.

Common causes of apply-time errors:

* A change made to a resource **outside** of Terraform (e.g. someone deleted a bucket manually) between plan and apply.
* Transient networking errors.
* An expected provider/API error (duplicate name, hitting a service limit).
* An unexpected upstream error (e.g. an internal server error).
* A bug in the provider or in Terraform itself.

When Terraform detects that a real object changed outside its control since the last apply, the next plan/apply explicitly reports it under **"Objects have changed outside of Terraform"** before proposing a new set of actions to reconcile.

## Replacing Individual Resources

Two flags scope an apply to specific resources — see 3d for full flag semantics:

* **`-replace=ADDRESS`** — force destroy-and-recreate a specific resource with the *same* configuration (e.g. an unhealthy instance). List current addresses first with `terraform state list`.
* **`-target=ADDRESS`** — apply changes to only a specific resource/module and its dependencies. Reserved for troubleshooting/recovery — not routine use.

## Exam Quick Facts

* Automatic plan mode **prompts**; saved plan mode **does not** (the saved file is the approval).
* `apply` **locks state** for the duration of the run, and **unlocks** even after an error.
* Apply executes in parallel where the dependency graph allows, sequential where required.
* On error: state is updated with partial progress, then Terraform **exits without rolling back** — no automatic undo.
* `-replace` recreates with the same config; `-target` scopes an apply to a subset of resources — both are exceptional-use tools, not routine workflow.
