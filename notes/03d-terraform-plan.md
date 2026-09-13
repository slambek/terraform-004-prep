# 3d. Generate and review an execution plan for Terraform

## What `terraform plan` Does (Default / "Normal" Mode)

1. **Refreshes** — reads the current state of already-existing remote objects so Terraform's state is up to date.
2. **Diffs** — compares the refreshed state to the current configuration.
3. **Proposes** — a set of change actions that would make real infrastructure match configuration.

`plan` **never makes changes** — it's read-only against your infrastructure (though it does update in-memory/cached state during refresh, and can write that to the state file if you explicitly save with `terraform apply` afterward — plan itself does not persist state changes to disk in normal mode).

If nothing needs to change, `plan` reports that no actions are needed.

## Planning Modes (Mutually Exclusive)

| Mode | Flag | Goal |
| :--- | :--- | :--- |
| **Normal** (default) | *(none)* | Change remote system to match configuration. |
| **Destroy** | `-destroy` | Plan to destroy all remote objects, leaving empty state. Equivalent to `terraform destroy` (3f). |
| **Refresh-only** | `-refresh-only` | Only reconcile Terraform's state/output values with changes made to remote objects **outside** Terraform — doesn't propose config-driven changes. |

You can't combine modes — activating one disables "normal" mode, and modes can't be combined with each other.

## Planning Options

* **`-refresh=false`** — skip the refresh step (faster, more API-request-friendly, but risks an incomplete/incorrect plan since external drift is ignored). Cannot combine with `-refresh-only` (that mode *is* the refresh).
* **`-replace=ADDRESS`** — plan to destroy-and-recreate a specific resource instance even though config hasn't changed (e.g. a degraded instance). Repeatable for multiple resources. Cannot combine with `-destroy`. (Supersedes the deprecated `terraform taint`.)
* **`-target=ADDRESS`** — scope planning to a resource/module instance and whatever it depends on. **Exceptional-use only** — routine use risks undetected drift elsewhere; prefer splitting large configs into smaller, independently-applied ones instead.
* **`-var 'NAME=VALUE'`** / **`-var-file=FILENAME`** — set input variable values on the command line; `-var-file` is generally preferred over `-var` because you avoid shell-quoting headaches for complex types (list/map/set need a full Terraform expression as the value; primitives — string/number/bool — are a plain string).

## Other Options (I/O and Reporting, Not Plan Content)

* **`-out=FILE`** — save the plan to a binary file you can later pass to `apply` to execute *exactly* those actions. Without `-out`, you get a **speculative plan** — description only, no intent to apply (used for PR review, per 3a).
* **`-json`** — machine-readable plan output.
* **`-detailed-exitcode`** — `0` = no changes, `1` = error, `2` = changes present. Useful for CI branching logic.
* **`-parallelism=n`** (default `10`) — cap concurrent graph operations.
* **`-compact-warnings`** — condense warning output to summaries only.

⚠️ **Saved plan files can contain sensitive data in cleartext** (even values obscured in terminal output) — never commit a plan file (binary or JSON) to version control.

## How Terraform Builds the Plan: The Dependency Graph

*(Advanced/background topic — not required for routine use, but shows up conceptually on the exam.)*

Terraform builds a graph before creating a plan:

* **Node types:** Resource Node (one per resource, or per `count`/`for_each` instance), Provider Configuration Node (represents fully configuring a provider), Resource Meta-Node (grouping convenience only, for `count > 1`).
* **Build order (simplified):** add resource nodes → attach provisioners → add explicit `depends_on` edges → add orphaned resources from state (resources removed from config but still in state) → map resources to their provider configuration nodes → parse interpolations/attribute references into dependency edges → add a single root node → split any resource being destroyed-and-recreated into separate destroy/create nodes (destroy order often differs from create order) → validate no cycles exist and there's a single root.
* **Walking the graph:** depth-first, **in parallel** where dependencies allow — a node runs as soon as all its dependencies have finished. Concurrency is capped by `-parallelism` (default 10). This is *not* how Terraform handles cloud API rate limits, though — providers (e.g. AWS) typically implement their own backoff/retry for that.

This is exactly why a **cycle** (two resources each referencing the other, see 3c) fails validation — the graph can't be walked if it isn't a DAG.

## Anatomy of a Saved Plan File (`-json` view)

* **`terraform_version` / `format_version`** — ensures the same Terraform version applies what generated the plan.
* **`configuration`** — a snapshot of your config *as written*, including `provider_config` (with locked version constraints) and `root_module.resources` / `module_calls` (with cross-references recorded, which is how Terraform knows operation order).
* **`variables`** — recorded input variable values — **including values marked `sensitive`**, stored in plaintext. (Environment-variable-supplied values are *not* recorded this way.)
* **`resource_changes`** — per-resource `actions` (`create`/`update`/`delete`/`replace` combos), `before`/`after`/`after_unknown` values, and `before_sensitive`/`after_sensitive` redaction lists.
* **`planned_values`** — the resolved "after" state for every resource, structured as one clean object per resource (used by tools like Sentinel policy checks and HCP Terraform cost estimation).
* **`prior_state`** — present only when a state file already existed before this plan — a snapshot of state exactly as it was pre-plan.

## Exam Quick Facts

* Default plan behavior = **refresh → diff → propose**; it changes nothing by itself.
* Three mutually exclusive modes: **Normal**, **Destroy** (`-destroy`), **Refresh-only** (`-refresh-only`).
* `-target` and `-replace` are for **exceptional circumstances**, not routine workflow.
* No `-out` → speculative plan (review-only). With `-out` → a plan file you can `apply` exactly.
* Plan files may contain **cleartext sensitive values** — never commit them.
* `-detailed-exitcode`: `0` no changes / `1` error / `2` changes present.
