# 4a. Use and differentiate `resource` and `data` blocks

## Core Distinction

```text
resource block → creates AND manages a real infrastructure object (CRUD)
data block     → READS existing information only — never creates or modifies
```

A **resource** is any infrastructure object you want Terraform to own the full lifecycle of. What resource types are available depends entirely on which **providers** you've installed (see 2a–2c).

## What Happens to Resources on `apply`

When you apply a configuration, Terraform performs whichever of these are needed to reconcile config with reality:

1. **Creates** resources in configuration that don't yet exist as real objects.
2. **Destroys** resources that exist in state but are no longer in configuration.
3. **Updates in place** resources whose arguments changed, if the provider API supports in-place updates for that argument.
4. **Destroys and re-creates** resources whose changed arguments *can't* be updated in-place (remote API limitation — the provider schema marks such arguments as "force new").
5. **Updates the state file** so config, real infrastructure, and state all match.

## Data Sources

A `data` block is associated with exactly **one** data source and can only **read** from it — never create or modify. Provider documentation defines which data sources exist and what arguments/attributes each supports.

```hcl
data "aws_ami" "example" {
  most_recent = true
  owners      = ["self"]
  tags = {
    Name = "app-server"
  }
}
```

Reference the result with `data.<TYPE>.<LABEL>.<ATTRIBUTE>`, e.g. `data.aws_ami.example.id`.

## When Terraform Reads a Data Source: Plan vs. Apply

Terraform *tries* to query data sources during the **planning** phase, but will **defer the read to the apply phase** when it can't yet predict the arguments needed to run the query. This happens when:

* The data block depends (directly or via a `local`) on a **Terraform-managed resource** that's scheduled to change in the current plan.
* A `precondition`/`postcondition` on the data block depends, directly or indirectly, on a resource that's changing.
* An argument in the data block refers to a value that can only be **computed during apply**.

When deferred, the data source's interpolated attributes show as `(known after apply)` in the plan, and downstream resources referencing that data can't be provisioned until apply. When none of a data block's arguments depend on unresolved values, Terraform reads it during the normal **refresh** step (before planning), so the diff reflects real fetched values.

⚠️ Adding `depends_on` to a data block on Terraform ≥0.13 always forces the read to defer to apply — earlier versions (0.12 and below) can produce unintended behavior with this pattern.

## Specialized / Local-Only Data Sources

Some data sources don't query external infrastructure at all — they generate data that exists only for the current operation and get recalculated on every plan:

* `template_file` (HashiCorp `template` provider) — renders a template.
* `local_file` (HashiCorp `local` provider) — reads a local file.
* `aws_iam_policy_document` (`aws` provider) — renders an IAM policy JSON document.

## Meta-Arguments Available on Both Block Types

Both `resource` and `data` blocks accept several Terraform-native **meta-arguments** that control *how* Terraform creates/reads them, rather than *what* they represent. Each has its own dedicated coverage elsewhere in this prep:

* `count` / `for_each` — multiple instances (4b/4d).
* `provider` — alternate/aliased provider config (2b).
* `depends_on` — explicit dependency (4f).
* `lifecycle` (`precondition`/`postcondition` sub-blocks) — custom validation (4g).

## Exam Quick Facts

* `resource` = full CRUD lifecycle; `data` = **read-only**, never creates/modifies.
* Apply performs up to 5 things per resource: create, destroy (removed from config), in-place update, destroy-and-recreate (forced replacement), and state sync.
* A data source read is deferred to **apply** only when its arguments depend on values not yet known during plan.
* Adding `depends_on` to a data block **always** forces an apply-time read (Terraform 0.13+).
* Specialized data sources (`template_file`, `local_file`, `aws_iam_policy_document`) generate/read local data, not remote infrastructure state.
