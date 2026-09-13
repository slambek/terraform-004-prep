# 3c. Validate a Terraform configuration

## What `terraform validate` Does

`terraform validate` checks that configuration is **syntactically valid and internally consistent** — correctness of attribute names, value types, block structure — using only the configuration itself.

**What it does NOT check:**

* Remote services — it never calls provider APIs.
* Remote state.
* Whether input variables are actually assigned real-world-valid values (that's the job of `plan`, which includes an implied validation check *plus* checks against a specific run context — target workspace, actual variable values, state).

## Prerequisites

`validate` needs an **initialized working directory** with providers and modules installed — run `terraform init` first (see 3b). If you only need to validate without touching any backend, you can initialize with:

```bash
terraform init -backend=false
```

## Usage

```bash
terraform validate
terraform validate -json     # machine-readable output for editor/CI integration
terraform validate -no-color
```

It's safe to run automatically and often — e.g. as an editor save-hook, or as a CI test step for reusable modules — because it never touches real infrastructure or remote state.

## `-json` Output Shape

The JSON object includes:

* `valid` (bool) — overall result.
* `error_count` / `warning_count` — always `0` errors when `valid` is `true` (warnings alone don't invalidate a config).
* `diagnostics` — array of objects, each with `severity` (`"error"`/`"warning"`), `summary`, `detail`, and an optional `range` (file + start/end source position) pinpointing the problem.

## Common Errors Validate Catches

* **Cycle errors** — circular logic in the dependency graph, e.g. two `aws_security_group` resources whose `ingress` blocks each reference the other's ID directly. Terraform can't determine creation order because each depends on the other existing first.
  * **Fix pattern:** break the cycle by moving the interdependent rules out of the resource blocks and into separate `aws_security_group_rule` resources that reference security group IDs — this lets both groups be created first (no interdependent config), then the rules attached afterward.
* **Invalid reference / invalid `for_each` value** — e.g. using a splat expression (`aws_security_group.*.id`, which produces a **list**) as a `for_each` value, which requires a **map** or **set**. Fix by deriving a map with a `locals` block instead.
* **Missing resource instance key** — referencing `aws_instance.web_app.id` when the resource has `for_each` set; Terraform requires indexing a specific instance (or a `for` expression across all instances, e.g. `[for instance in aws_instance.web_app : instance.id]`) once a resource is no longer single-instance.
* **Missing / unsupported argument** — mismatched arguments after a module version bump (a new module version may require different input names entirely).

**Important:** Terraform stops at the **first class of error** it finds during validation — fixing one error and re-running `validate` is normal; expect multiple passes for a broken configuration.

## Exam Quick Facts

* `validate` = **syntax + internal consistency only** — no provider API calls, no state, no real variable-value checks.
* Requires `terraform init` (or `init -backend=false`) first.
* `plan` includes an implied validation check *plus* real provider/state context — `validate` is the lighter, standalone version.
* Safe and cheap to run repeatedly/automatically (editor hooks, CI).
* Terraform reports and stops at one class of error at a time — expect iterative fix → re-validate cycles.
