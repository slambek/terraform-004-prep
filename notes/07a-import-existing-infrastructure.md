# 7a. Import existing infrastructure into your Terraform workspace

> **Scope note:** This file covers bringing infrastructure that Terraform has
> never managed under management for the first time (the `terraform import`
> command and the config-driven `import` block workflow). Using `import` +
> `removed` together to migrate a resource that's *already* Terraform-managed
> from one state file to another is covered in 6d — don't re-derive that
> workflow here, just note the command exists.

## Core Concept

Importing binds an existing real-world object to a resource address in
Terraform state so Terraform can manage its full lifecycle going forward.
Terraform can only inspect the object's **current** attributes — it cannot
import health, intent, or history, and it does not detect relationships
between resources for you.

## Two ways to import

| Approach | Command/config | Notes |
|---|---|---|
| Legacy CLI | `terraform import ADDRESS ID` | Imperative, one resource at a time, no plan preview beforehand |
| Config-driven (≥ 1.5) | `import` block + `terraform plan`/`apply` | Safer: previewable, CI/CD-friendly, can auto-generate config |

### `terraform import` (CLI)

```bash
terraform import aws_instance.foo i-abcd1234
terraform import module.foo.aws_instance.bar i-abcd1234
terraform import 'aws_instance.baz[0]' i-abcd1234              # count
terraform import 'aws_instance.baz["example"]' i-abcd1234      # for_each
```

- **ADDRESS** is any valid resource address (root or inside a module).
- **ID** format is resource/provider-specific (e.g. an EC2 instance ID vs. a
  Route53 zone ID) — consult the provider's docs.
- Imports **only one resource at a time** — it cannot import an entire
  collection (e.g. a whole VPC) in one call.
- You must already have a (possibly empty) matching `resource` block in
  configuration before importing — you can leave arguments unset and fill
  them in afterward based on `terraform plan` output.
- Requires provider configuration to be resolvable from `.tf` files in
  `-config` (defaults to the working directory), environment variables, or
  interactive prompts. Provider config used for import **cannot depend on a
  data source** — only variables.

Useful flags:

| Flag | Purpose |
|---|---|
| `-config=path` | Directory of config that configures the provider (default: cwd) |
| `-var` / `-var-file` | Supply variable values (only relevant with `-config`) |
| `-lock=false`, `-lock-timeout` | Locking control during the operation |
| `-provider=provider` | *Deprecated* — override the provider used for import |
| `-ignore-remote-version` | Only for HCP Terraform CLI integration / `remote` backend |
| `-state`, `-state-out`, `-backup` | Legacy, **local backend only** |

### Complex imports

Some resources import as more than one state entry — e.g. importing an AWS
network ACL also creates one `aws_network_acl_rule` state entry per rule.
These secondary resources won't already exist in your configuration, so you
must add matching resource blocks yourself, or Terraform will plan to
**destroy** them on the next run.

### Config-driven import (`import` block)

```hcl
import {
  id = "i-abcd1234"
  to = aws_instance.example
}
```

Both `id` and `to` are required. Workflow:

1. Add the `import` block (anywhere in config).
2. Run `terraform plan -generate-config-out=generated.tf` — Terraform
   locates the object, and writes a full resource block (every argument,
   including defaults) to the named file. **This file is not applied
   automatically** — you must review, edit, and move it into your real
   config before committing.
3. **Prune** the generated config to required arguments and any values that
   differ from provider defaults; leaving every generated argument can
   trigger unwanted diffs or replacements (e.g. a `null` default the
   provider actually needs as `[]`).
4. Re-run `terraform plan` to confirm the plan is now a **no-op import**
   (or only non-destructive in-place changes) before applying.
5. `terraform apply` performs the import and any queued changes together —
   config-driven import can import *and* modify a resource in one step,
   unlike the CLI form.

## Limitations

- Import uses only the **current** reported state of the infrastructure —
  it can't tell you about health, original intent, or unmanaged filesystem
  state inside e.g. a container.
- Doesn't generate relationships/dependencies between resources — add
  `depends_on` or references yourself.
- Doesn't tell you which default attributes are safe to omit — you decide
  during pruning.
- Not all providers/resources support import.
- Importing a resource doesn't guarantee Terraform can safely destroy and
  recreate it — it may depend on unmanaged infrastructure.
- Take a state backup before importing new infrastructure.

## Exam Quick Facts

- `terraform import` binds **exactly one** resource per invocation — never
  a whole collection.
- A matching (even empty) `resource` block **must already exist** in config
  before you run `terraform import ADDRESS ID`.
- Config-driven import's `-generate-config-out` output is a **starting
  point** — applying it unpruned commonly causes unwanted replacements.
- `import` block requires exactly two arguments: `id` and `to`.
- Complex imports create extra state entries you must manually mirror in
  config, or Terraform will plan to destroy them.
- `-state`/`-state-out`/`-backup` on `terraform import` only apply to the
  **local** backend; `-ignore-remote-version` only applies to `remote`
  backend / HCP Terraform CLI integration.
