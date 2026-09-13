# 6d. Manage resource drift and Terraform state

> **Scope note:** Backend configuration syntax lives in 6c; locking lives in
> 6b; local-backend specifics live in 6a. This file covers state's purpose,
> refresh/drift behavior, state inspection/refactor commands, and the
> `moved`/`removed` blocks.

## Core Concept

Terraform state is the record binding your configuration's resource
instances to real-world remote objects. Before any plan/apply, Terraform
**refreshes** — reconciling state with actual infrastructure — so it can
compute an accurate diff. "Drift" is any mismatch between state (or config)
and real infrastructure caused by changes made **outside** Terraform.

## Why State Exists

- Maps configuration resource instances to real objects (via IDs).
- Tracks metadata and improves performance on large infrastructures.
- State format is JSON, but **direct file editing is discouraged** — use the
  `terraform state` CLI subcommands, which insulate you from format changes
  across Terraform versions.
- Terraform expects a strict **one-to-one** mapping between configured
  resource instances and remote objects. Breaking this (e.g. via
  `terraform import` or `state rm`) is your responsibility to reconcile.

## Inspecting & Modifying State (CLI)

> The full `terraform state` subcommand catalog (`list`, `show`, `mv`,
> `pull`, `push`, `rm`, `replace-provider`) as day-to-day tooling lives in
> **7b**; `terraform import` is covered fully in **7a**. This section only
> covers the guardrails specific to migration/drift work.

`terraform state push` guardrails (bypassable with `-force`, not
recommended without a backup first):

- **Differing lineage** (unique ID assigned at state creation) → refused,
  since it implies you're pushing an unrelated state.
- **Higher serial** on the destination → refused, since the destination has
  changes yours doesn't reflect.

## Refresh & Refresh-Only Mode

> Flag semantics for `-refresh=false` and the three mutually-exclusive
> planning modes (including `-refresh-only`) are covered in **3d** — this
> section only covers how refresh-only fits into a drift workflow.

- Every `plan`/`apply` does an **implicit in-memory refresh** first (3d).
- `terraform plan -refresh-only` / `terraform apply -refresh-only` explicitly
  surface drift: they show what *state* would change to match real
  infrastructure, **without touching real infrastructure or config**.
- The deprecated `terraform refresh` subcommand does the same update but
  **silently overwrites state with no review** — `-refresh-only` is the
  safer, preferred alternative because you can inspect and choose not to
  apply it.
- A refresh-only apply can also update **output values** — relevant if other
  workspaces consume them via `terraform_remote_state` (see 8d).

## Drift Detection & Remediation Workflow

1. `terraform plan -refresh-only` → review detected differences.
2. `terraform apply -refresh-only` → accept, updating **state only** to
   match reality (no infra changes).
3. If the drifted object should be config-managed, add a matching resource
   block, then `terraform import RESOURCE.NAME id` to bind it.
4. Run a normal `terraform plan`/`apply` afterward to reconcile config with
   the now-accurate state (e.g. re-attach a security group Terraform no
   longer knew about).

## Refactoring State — Splitting Configuration

Reasons to split state: long applies, resources with different lifecycle
cadence, resources moving to a new team, or extracting a reusable module.
Grouping guidance: separate by volatility/rate of change, stateful vs.
stateless, and team ownership boundaries.

Before migrating, identify **inter-resource dependencies** — prefer dynamic
references over hardcoding:

- A provider's own data source (e.g. `aws_vpc` lookup).
- `tfe_outputs` data source, for HCP Terraform/Enterprise cross-workspace
  outputs.
- `terraform_remote_state` data source for any other remote or local backend
  (requires explicit workspace access permissions in HCP Terraform — see 8d
  for the full comparison with `tfe_outputs`).
- `terraform graph` to visualize dependencies before you cut anything.

### Migration approach 1 — `removed` + `import` blocks (recommended, Terraform ≥ 1.7)

Preferred because the blocks leave a config-driven record of the move.

In the **source** config:

```hcl
removed {
  from = aws_instance.example
  lifecycle {
    destroy = false   # keep the real object; just drop it from state
  }
}
```

Run `plan`/`apply` — this removes the binding from state **without**
destroying the resource.

In the **destination** config:

```hcl
resource "aws_instance" "example" {
  instance_type = "t3.micro"
  ami           = data.aws_ami.example.id
}

import {
  id = "i-07b510cff5f79af00"
  to = aws_instance.example
}
```

Run `plan`/`apply` — this binds the existing object into the new state
without recreating it. Terraform can even auto-generate the resource block
for you from an `import` block via `terraform plan -generate-config-out=FILE`
(Terraform ≥ 1.5), but you must review, commit, and re-plan the generated
config — you can't apply directly off generated config.

### `removed` block reference

```hcl
removed {
  from = aws_instance.example
  lifecycle {
    destroy = true   # default: also destroys the real resource
  }
}
```

- `from` (required) — address of the resource to drop from state.
- `lifecycle.destroy` — `true` (default) destroys the real object too;
  `false` only removes the state binding, handing the object off elsewhere.
- Supports `connection` / `provisioner` blocks, but **only destroy-time
  provisioners** are valid here, and `when` is required on any provisioner
  nested in a `removed` block.

### `moved` block reference

Renames/relocates a resource address **without** destroying it — used for
in-place refactors (e.g. renaming a resource, moving it into a module),
not cross-state migration.

```hcl
moved {
  from = aws_instance.a
  to   = aws_instance.b
}
```

Terraform checks state for an object at `from`, renames it to `to`, and
plans against the **new** address — so no destroy/recreate happens.

### Migration approach 2 — `terraform state mv` (legacy)

```bash
terraform state pull > source.tfstate
terraform state pull > destination.tfstate     # run from destination dir

terraform state mv \
  -state source/source.tfstate \
  -state-out destination/destination.tfstate \
  aws_instance.example aws_instance.example

terraform state push source.tfstate            # from source dir
terraform state push destination.tfstate       # from destination dir
```

- Works directly on local state files if using the `local` backend; with a
  remote backend you must `pull` both states down first and `push` both
  back up after.
- Requires Terraform ≥ 1.0. Riskier than `removed`+`import` (manual
  pull/push has some chance of corrupting remote state) — HashiCorp
  recommends `removed`/`import` for new migrations.
- After moving, update both configs to match (remove from source, add to
  destination) and `plan` each to confirm **zero** planned changes before
  merging.

## Exam Quick Facts

- `-refresh-only` **never** modifies real infrastructure — only the state
  file (and outputs). Plain `plan`/`apply` refresh is implicit and separate.
- The `refresh` subcommand is deprecated in favor of `-refresh-only` because
  it overwrites state with **no chance to review** first.
- `removed { lifecycle { destroy = false } }` drops a resource from state
  **without** deleting the real object — this is the key "does NOT destroy"
  edge case to remember.
- `moved` is for **renaming/relocating within the same state**; it is not
  how you move a resource to a *different* state file.
- `terraform state mv` across state files needs manual `pull`/`push` on a
  remote backend; it's legacy — HashiCorp recommends `removed`+`import`
  instead for new work.
- `terraform state push` protections (`lineage` mismatch, higher `serial`)
  can both be bypassed with `-force`, which is explicitly discouraged
  without a `state pull` backup first.
- Auto-generated config from `import -generate-config-out` **cannot** be
  applied directly — you must review/commit it and re-plan.
