# 6a. Describe the local backend

> **Scope note:** Generic backend block syntax, other backend types, partial
> configuration, and credentials handling live in 6c. Locking mechanics
> (force-unlock, lock IDs) live in 6b.

## Core Concept

The `local` backend is Terraform's **default backend** — if you don't declare
a `backend` block at all, Terraform uses `local`. It stores state as a plain
JSON file on the local filesystem, performs all operations (plan/apply)
locally, and locks the state file using OS-level file system APIs.

## Configuration

```hcl
terraform {
  backend "local" {
    path = "relative/path/to/terraform.tfstate"
  }
}
```

Supported arguments:

| Argument | Required | Description |
|---|---|---|
| `path` | No | Path to the state file. Defaults to `terraform.tfstate` in the root module. |
| `workspace_dir` | No | Path used to store state for non-default workspaces. |

If you omit the `backend` block entirely, this is functionally identical to
declaring `backend "local" {}` with default arguments.

## Using local state as a data source

```hcl
data "terraform_remote_state" "foo" {
  backend = "local"

  config = {
    path = "${path.module}/../../terraform.tfstate"
  }
}
```

## Legacy CLI flags (local backend only)

These predate remote backends and are kept for backward compatibility. They
**only affect configurations using the `local` backend** (or no backend
block at all) — they have no effect on any other backend type.

- `-state=FILENAME` — override the file Terraform reads the prior state from.
- `-state-out=FILENAME` — override the file Terraform writes new state to.
  If you set `-state` without `-state-out`, Terraform reuses the `-state`
  filename for output too, **overwriting the input file**.
- `-backup=FILENAME` — override the auto-generated backup filename. Use
  `-backup=-` to disable backups entirely.

Using all three of these overrides bypasses Terraform's normal
workspace-based filename selection — if you rely on multiple workspaces,
you'd need to pick distinct filenames yourself.

## Exam Quick Facts

- `local` is the **default** backend — no `cloud`/`backend` block = local.
- It is one of the few backend types that provides **both** storage *and*
  locking (via system/file APIs), no extra service required.
- `path` defaults to `terraform.tfstate` relative to the **root module**.
- `-state`, `-state-out`, and `-backup` are legacy, local-backend-only flags
  — never mention them for remote backends.
- Only `path` and `workspace_dir` are valid arguments in a `backend "local"`
  block; nothing else is supported.
