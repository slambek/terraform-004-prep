# 6c. Configure remote state using the backend block

> **Scope note:** `local`-backend-specific arguments/flags are in 6a. Locking
> mechanics are in 6b. State inspection/refactoring commands (`state pull`,
> `state push`, `state mv`, `moved`/`removed` blocks, drift) are in 6d.

## Core Concept

A nested `backend` block inside the top-level `terraform` block tells
Terraform where to store its state remotely instead of on local disk, so
multiple people/automation can share and lock the same state.

```hcl
terraform {
  backend "remote" {
    organization = "example_corp"

    workspaces {
      name = "my-app-prod"
    }
  }
}
```

## Rules & Limitations

- A configuration can declare **only one** `backend` block.
- A `backend` block **cannot reference named values** — no input variables,
  locals, or data source attributes inside it.
- Values declared inside a `backend` block cannot be referenced elsewhere in
  the configuration.
- If the configuration has a `cloud` block, it **cannot also** have a
  `backend` block.
- Don't configure a `backend` block when using HCP Terraform/Terraform
  Enterprise workspaces — those manage state automatically.
- Terraform ships with a fixed set of built-in backend types; you **cannot**
  load additional backends as plugins.
- Backend-specific arguments (bucket names, tokens, etc.) live in the body
  of the block and vary per backend type — check that backend's own docs
  page.

## Credentials & Sensitive Data

- Avoid hardcoding credentials in the backend block or via `-backend-config`
  — Terraform writes the merged backend config in **plain text** to
  `.terraform/terraform.tfstate` (the *backend* config file, not your real
  state) and into any saved plan files.
- Prefer environment variables or the target system's conventional
  credentials file.
- `terraform apply` on a previously saved plan file uses the backend config
  **captured at plan time**, not the current settings — if that used
  time-limited credentials, they may expire before apply runs.

## Initializing the Backend

Any time you add, remove, or change backend configuration, you must re-run:

```bash
terraform init
```

- This creates/updates the local `.terraform/` directory (never commit it —
  it can contain credentials).
- `.terraform/terraform.tfstate` here stores **backend config metadata**,
  not your infrastructure's actual state (`terraform.tfstate`), which lives
  in the remote backend itself.
- Changing backends prompts Terraform to offer **state migration** to the
  new backend, preserving existing state.

## Partial Configuration

You can omit some/all backend arguments from the config and supply them at
`init` time instead — useful when automation injects values.

```hcl
# state.tf
terraform {
  backend "s3" {
    bucket  = ""
    key     = ""
    region  = ""
    profile = ""
  }
}
```

Ways to supply the rest:

| Method | Example |
|---|---|
| File | `terraform init -backend-config="./state.config"` |
| CLI key/value | `terraform init -backend-config="KEY=VALUE"` |
| Interactive | Terraform prompts for missing **required** values (never for optional ones) |

A standalone backend config **file** (not a `.tf` file) lists attributes
flat, without wrapping them in a block:

```hcl
# state.config (or recommended: *.backendname.tfbackend, e.g. config.s3.tfbackend)
bucket  = "your-bucket"
key     = "your-state.tfstate"
region  = "eu-central-1"
profile = "Your_Profile"
```

- CLI `-backend-config` key/value pairs are visible in shell history — avoid
  for secrets.
- When mixed, precedence is: command-line options **override** the main
  configuration, and later `-backend-config` flags override earlier ones.
- With partial configuration, you must still declare at minimum an **empty**
  backend block naming the type: `backend "consul" {}`.

## Changing or Removing the Backend

- You can change backend type or arguments any time; Terraform detects the
  diff and asks to reinitialize.
- Reconfiguring even the *same* backend still prompts a migration question
  — you can decline ("no").
- With multiple workspaces present, Terraform asks whether to copy **all**
  workspaces to the new backend.
- To remove remote state entirely: delete the `backend` block and
  reinitialize; Terraform offers to migrate state back to `local`.

## Exam Quick Facts

- Only **one** `backend` block per configuration — ever.
- `backend` blocks **cannot** use variables/locals/data source references —
  values must be literal (or supplied via partial config).
- `cloud` block and `backend` block are **mutually exclusive**.
- Any backend config change → you must run `terraform init` again before
  plan/apply/state operations will work.
- Recommended sensitive-value delivery is environment variables — not
  `-backend-config` literals or hardcoded arguments — because both persist
  in plain text on disk (`.terraform/` and plan files).
- Partial config still requires a non-empty **type label** in the backend
  block, even if every argument is blank.
