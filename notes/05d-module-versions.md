# 5d. Manage module versions

## The `version` Argument

```hcl
module "consul" {
  source  = "hashicorp/consul/aws"
  version = ">= 0.10.0"
}
```

* Accepts the **same constraint operators** as provider versions (`>=`, `~>`, exact — see 2a).
* **Only valid when `source` points at a registry** (public Terraform Registry or a private registry) — modules sourced from a local path, Git, S3, HTTP, etc. have no separate version concept: they're always whatever code exists at that source location. A local-path module in particular is loaded from the *same* repository as its caller and therefore always shares the caller's version — there's nothing separate to pin.
* Changing `version` — like changing `source` — **requires re-running `terraform init`**.
* If you omit `version` for a registry module, Terraform installs the **latest** version — strongly discouraged for anything beyond quick experimentation, since a later run could silently pull in breaking changes.

## Registry Modules Follow Semantic Versioning

Every module published on the Terraform Registry is versioned, and versions **must syntactically follow semantic versioning** (`MAJOR.MINOR.PATCH`); publishers are encouraged to follow semver's full meaning (breaking changes bump MAJOR), not just the syntax.

## ⚠️ Module Versions Are NOT Recorded in the Dependency Lock File

This is the single most important exam distinction versus **providers** (2a):

| | Providers | Modules |
| :--- | :--- | :--- |
| Tracked in `.terraform.lock.hcl`? | **Yes** | **No** |
| Re-running `init` without `-upgrade`... | ...re-selects the exact locked version | ...**always re-resolves to the newest version matching the constraint** |
| To pin a version deterministically | Rely on the lock file | Use an **exact** version constraint (no `>=`/`~>`) in the module block itself |

Because there's no module lock file, the *only* way to guarantee Terraform selects the same module version on every `init` is to write an **exact constraint** — a range like `>= 0.10.0` can silently resolve to a different (newer) version on a teammate's machine or in CI than it did for you.

## Resolution Behavior on `terraform init`

* If no acceptable version of a module is installed yet, Terraform downloads the **newest version matching the constraint**.
* `terraform init -upgrade` explicitly re-checks for newer module versions (same flag as for providers, 3b) and updates the installed copy.
* Local-path modules are **not versioned** at all in this sense — Terraform just re-reads whatever's currently on disk; editing a local module's files takes effect immediately without re-running `init`/`get`.

## Upgrading a Module Version Can Break Your Configuration

Bumping a module's `version` can change its expected input arguments or exposed outputs entirely — this isn't hypothetical, it's a normal part of a module's own semantic versioning (a MAJOR bump signals exactly this). After changing a module version:

1. Run `terraform init` (required, since `source`/`version` changed).
2. Run `terraform validate` — expect possible **"Unsupported argument"** / **"Missing required argument"** errors if the new version's variable interface changed (see 3c/4g for validation mechanics).
3. Update the calling `module` block's arguments to match the new interface.
4. Re-validate before planning/applying.

## Exam Quick Facts

* `version` is valid **only** for registry-sourced modules — never for local, Git, S3, HTTP, etc.
* Module versions are **never recorded in `.terraform.lock.hcl`** — that file tracks providers only.
* Without an exact constraint, `terraform init` can install a **different, newer** module version on a different machine/run than it did before — the module equivalent of "no lock file" drift.
* `terraform init -upgrade` re-checks for newer versions for both providers **and** modules.
* A module version bump can change required arguments/outputs — always re-`validate` after upgrading.
