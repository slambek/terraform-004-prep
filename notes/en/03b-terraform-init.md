# 3b. Initialize a Terraform working directory

## What `terraform init` Does

`terraform init` is the **first command to run** after writing a new configuration or cloning an existing one. It prepares the working directory by performing, in order:

1. **Backend initialization** — reads the backend config in the root module and initializes it. (Backend types/config are detailed in Section 6 — this note only covers that init is the trigger.)
2. **Child module installation** — finds `module` blocks and retrieves module source code.
3. **Provider plugin installation** — finds direct/indirect provider references and installs the matching plugins, writing/updating `.terraform.lock.hcl` (full lock-file mechanics are in 2a — this note only covers *when* that write happens).

It is **always safe to run multiple times** — it never deletes existing configuration or state, though later re-runs can produce errors if things changed (e.g. backend or module version changes) that need an explicit flag to resolve.

## Key Flags

| Flag | Effect |
| :--- | :--- |
| `-upgrade` | Ignore recorded lock-file selections; install the newest version allowed by constraints, for **both** providers and modules. |
| `-backend=false` | Skip backend initialization. Useful for offline validation; some other init steps require an initialized backend, so use only when the directory was already initialized once. |
| `-reconfigure` | Discard existing backend configuration; does **not** migrate state. |
| `-migrate-state` | Attempt to copy existing state to a newly configured backend. |
| `-force-copy` | Suppresses `-migrate-state` confirmation prompts (answers "yes" automatically); also enables `-migrate-state`. |
| `-get=false` | Skip child module installation. |
| `-plugin-dir=PATH` | Install providers only from a local directory (one-time override — for routine use, configure a filesystem mirror globally instead). |
| `-from-module=SOURCE` | Copy a module into an **empty** target directory before running the rest of init — a shorthand for checking out a starter config. |
| `-lockfile=readonly` | Suppress lock-file changes; only verify checksums against what's already recorded. Conflicts with `-upgrade`. |
| `-input=true/false` | Whether to prompt for missing input; `false` errors instead of prompting — useful for automation. |
| `-lock` / `-lock-timeout` | Control state locking during init's state-related operations. |

## Re-Running Init After Config Changes

Terraform requires re-initialization whenever you:

* Add, remove, or change the **version of a module or provider**.
* Add, remove, or change the **backend or `cloud` block**.
* Clone a repo containing Terraform config for the first time.

If you skip this, `validate`/`plan`/`apply` will detect it and prompt you to re-run `init` (and sometimes specifically `init -upgrade`, e.g. when a lock-file-recorded provider version no longer satisfies a changed constraint).

## What Gets Created

* **`.terraform.lock.hcl`** — dependency lock file (see 2a for full semantics).
* **`.terraform/`** directory — Terraform's own cache, containing:
  * `.terraform/modules/` — `modules.json` (maps every module key, including the implicit root module, to its source and local directory) plus local copies of any **remote** modules. Local modules are referenced directly from their configured path and picked up immediately on edit; remote modules are only refreshed by `init -upgrade` or `terraform get`.
  * `.terraform/providers/` — cached provider plugin binaries, laid out as `[hostname]/[namespace]/[name]/[version]/[os_arch]`.

**Never** commit `.terraform/` to version control, and never hand-edit its contents — it's fully managed by Terraform and its structure can change between versions.

## Exam Quick Facts

* `init` order: **backend → modules → providers → lock file**.
* Always safe to re-run; never deletes config or state.
* `-upgrade` is the only flag that reconsiders newer provider/module versions against the lock file.
* Changing a module/provider version or the backend/`cloud` block **requires** re-running `init`.
* `.terraform/` is a local cache — never commit it, never edit it by hand.
