# Terraform Associate (004) — Exam-Day Cheatsheet

Only what actually confuses people or gets asked on the test. For the full
breakdown — links to `notes/en/` at the end of each block.

---

## 1. Core Philosophy (IaC, providers, state — basic concepts)

* **Declarative** (Terraform) = **WHAT**; **Imperative** (Bash/Python) = **HOW, step-by-step**.

* **Idempotency** ≠ **Immutability**: idempotent = re-running produces the same result; immutable = a resource is not modified in place, but recreated.

* **Day 0** (provisioning: VPC, VM, DB) → Terraform. **Day 1+** (OS patches, application configuration) → Ansible/Chef etc. Terraform can participate in both, but that's not its primary role.

* Terraform Core **does not know** about AWS/Azure/K8s — that knowledge lives entirely in the **provider plugin**; Core talks to it over **RPC**.

* The only **built-in provider** (powers `terraform_remote_state`) is `terraform.io/builtin/terraform`; it does not need `required_providers`, unlike all the others.

* Terraform **requires** state — it's not optional. Three roles of state: (1) mapping config → real object, (2) metadata (dependencies, provider config), (3) attribute cache for performance (the most optional of the three).

* Every real object must map to **exactly one** resource instance in state.

* **Never** hand-edit the state file.

📄 [01a](notes/en/01a-what-is-iac.md) · [01b](notes/en/01b-advantages-of-iac.md) · [01c](notes/en/01c-multicloud-hybrid.md) · [02b](notes/en/02b-provider-usage.md) · [02d](notes/en/02d-state-fundamentals.md)

---

## 2. Providers

* `required_providers` (inside the `terraform {}` block) = **which provider and which versions are allowed**. The `provider {}` block = **how it's configured** (region, auth). `version` **never** goes inside `provider {}` — only in `required_providers`.

* Provider version **is tracked** in `.terraform.lock.hcl` (commit it to VCS). `terraform init` without `-upgrade` reuses the pinned version; `-upgrade` is the only thing that reconsiders newer allowed versions.

* The source address resolves to `registry.terraform.io` by default if no hostname is given.

* **Alias** (`provider "aws" { alias = "west" }`) = multiple **configurations of the same** provider. **Multiple providers** (2c) = multiple **different** providers in one configuration. This is a classic exam trap — don't confuse them.

* If **all** configurations of a provider have an alias — Terraform creates an **implied empty default**, which will error out if the provider has required arguments.

* Terraform resolves a resource's provider by its **type prefix** (`aws_instance` → `aws`), so the named local name matters.

* Cross-provider dependencies (e.g. `random` → `aws`) resolve automatically through a single unified resource graph — no special syntax is needed.

📄 [02a](notes/en/02a-install-providers.md) · [02b](notes/en/02b-provider-usage.md) · [02c](notes/en/02c-multiple-providers.md)

---

## 3. Core CLI Workflow

* `terraform init` order: **backend → child modules → providers → lock file**. Always safe to rerun. Changing a module/provider version or the backend block **requires** re-running `init`.

* `terraform validate` = only **syntax + internal consistency**, no API calls and no real context of variable values. Requires an initialized directory (`init -backend=false` works). Stops at the **first class** of errors — expect several iterations.

* `terraform fmt` = only **style** (no customization options), it does not check correctness against the provider — that's `validate`'s job. Order: `fmt` → `validate` → `plan` → `apply`.

* `terraform plan`: **refresh → diff → propose**, it doesn't change anything by itself. Three mutually exclusive modes: **Normal** (default), **Destroy** (`-destroy`), **Refresh-only** (`-refresh-only`, only updates state/outputs to match reality, doesn't touch the config).

  * `-target` / `-replace` — **exceptional use**, not routine.

  * Without `-out` → **speculative plan** (view-only, for PR review). With `-out` → a plan file that can be applied **exactly as-is**.

  * Saved plan files can contain **sensitive values in plaintext** — never commit them.

  * `-detailed-exitcode`: `0` no changes / `1` error / `2` there are changes.

* `terraform apply`: automatic-plan-mode **asks for confirmation**; saved-plan-mode (`apply tfplan`) does **not** (the file itself is the confirmation), and you cannot add extra planning flags there. On an error mid-apply: **state is updated with the partially successful changes**, the lock is released, **there is no automatic rollback**.

* `terraform destroy` = literally `terraform apply -destroy`. To remove a single resource without destroying everything — just delete/comment out the block and run `apply`; `destroy` is not needed for that.

* The `-target` flag appears in `plan`/`apply`/`destroy` — everywhere it's a "last resort", it's preferable to split up the configuration instead.

📄 [03a](notes/en/03a-terraform-workflow.md)–[03g](notes/en/03g-terraform-fmt.md)

---

## 4. Configuration Language

**Resource vs Data:**

* `resource` = full CRUD lifecycle; `data` = **read-only**, never creates/changes anything.

* A data source read is deferred until **apply** only if its arguments depend on values not yet known at plan time (e.g. it depends on a managed resource that's changing). `depends_on` on a data block **always** forces an apply-time read (0.13+).

**References & shape:**

* Without `count`/`for_each` → the reference is an **object**. With `count` → a **list**. With `for_each` → a **map** (key ≠ index).

* Splat (`[*]`) works on **lists** (including `count` resources), **not** directly on `for_each` maps — use `values(...)[*]` first.

* Resource address: `[module path][resource spec]`; no module path — root module.

**Variables & Outputs:**

* Without `default`, a variable is **required** from the caller. `default` must be a **literal** — it cannot reference other objects.

* Value-assignment precedence (low → high): env vars (`TF_VAR_*`) → `terraform.tfvars` → `*.auto.tfvars` → `-var-file` → `-var`.

* `sensitive = true` is **required** on an output if the value is derived from a sensitive resource/variable — otherwise it's an error. But `terraform output <name>` and `-json` **do not redact** sensitive values — only the "all outputs at once" human-readable view is redacted.

* `ephemeral` (1.10+) on an output — **child modules only**, never root.

**Complex Types:**

* Collection types (`list`, `map`, `set`) — **one** element type. Structural types (`object`, `tuple`) — **different** types per a fixed schema.

* Object ↔ map conversion can be **lossy** (extra keys are dropped).

* `optional(TYPE, DEFAULT)` substitutes DEFAULT both when the value is **absent** and when it's explicitly **`null`** — the only way to guarantee a non-null value inside a module.

* `any` is a placeholder, not a type; use it only when the value passes through without inspection.

**Expressions & Functions:**

* Named values (`var.`, `local.`, `path.*`, `terraform.workspace`) are **not real objects** — you can't iterate over them with `for`.

* You can't write your own functions in HCL — only provider-defined functions (`provider::name::func()`).

* `file()` never interpolates; `templatefile()` does interpolate.

**Dependencies:**

* **Implicit** (referencing an attribute) — the preferred way, no extra syntax needed. **Explicit** (`depends_on`) — only when a real dependency isn't visible through references; works on `resource`/`module`/`data`/`output`. Costs apply time (removes parallelism).

* Order within a `.tf` file **has no effect** on execution order — only the dependency graph does. Destroy runs in **reverse** order from create.

**Validation (order — a frequent exam question):**

```text
1. Input variable validation   — before the plan is generated
2. Preconditions               — after the plan, before create/read
3. Postconditions              — after apply/reading a data source
4. Check blocks                — the last step of plan/apply, does NOT block
```

* Only `check` blocks don't block the operation (warning + continue); all the others — halt.

* `output` only supports `precondition` (no `self`, no `postcondition`).

* A `data` block inside a `check` cannot be referenced from outside it.

**Sensitive Data & Vault:**

* `sensitive = true` hides the value only from **CLI/UI output** — it remains **plaintext** in state/plan.

* The only ways to **avoid storing** a value at all: `ephemeral` (variable/output/block) and write-only arguments (`_wo` + `_wo_version`, 1.11+).

* The Vault provider gives you **short-lived** credentials instead of static ones — but it **doesn't redact** anything in state/plan; whatever is read from Vault still ends up in state in plaintext.

📄 [04a](notes/en/04a-resource-vs-data-blocks.md)–[04h](notes/en/04h-sensitive-data-vault.md)

---

## 5. Modules

* `source` — must be a **literal string** (no expressions). Changing `source` **or** `version` always requires `terraform init`.

* Registry format: `<NAMESPACE>/<NAME>/<PROVIDER>`. Private registry = the same plus a hostname prefix.

* **Module versions are NOT pinned in `.terraform.lock.hcl`** (unlike providers!) — without an exact version (`=`, not `>=`/`~>`) you can end up with different versions on different machines.

* Each module is an **isolated namespace**: a child module can't see the parent's `var.*`/`local.*`. Values go in **only** through the `module` block's arguments, and come out **only** through `output`.

* **Provider configuration is the exception**: it's inherited **implicitly** from the caller. Child modules generally **should not** have their own `provider {}` blocks. For a non-default alias you need **both** the `providers` meta-argument on the parent **and** `configuration_aliases` in the child module.

* `count`/`for_each` on a `module` block are mutually exclusive, just like on `resource`/`data`.

* Never ship with a module: `terraform.tfstate`, `.terraform/`, `*.tfvars`.

* Recommendation: a **flat** module tree (one level of nesting), wired together with expressions from the root module.

📄 [05a](notes/en/05a-module-sources.md)–[05d](notes/en/05d-module-versions.md)

---

## 6. State, Backends & Drift

* **`local`** — the default backend (no `backend` block = local). Provides **both** storage **and** locking (via file APIs), with no external service.

* State locking is **automatic** for any operation that might write to state, but it's **optional** — it depends on backend support. A failed lock **stops** the operation — it doesn't proceed "silently" without a lock.

* `force-unlock LOCK_ID` — only for **your own** stuck lock; you cannot force an active lock held by someone else (risk of corruption).

* Only **one** `backend` block per configuration; `backend` **cannot** reference variables/locals/data sources. The `cloud` block and the `backend` block are **mutually exclusive**.

* `-refresh=false` skips the auto-refresh on `plan`/`apply`. `-refresh-only` is an explicit mode: it shows what would change **in state only**, without touching infrastructure/config. The deprecated `terraform refresh` does the same thing but **without a chance to review before applying** — use `-refresh-only` instead.

* **`removed` + `import` blocks** (≥1.7) — the recommended way to move a resource between state files (config-driven, leaves a trace in the config). `terraform state mv` is the legacy alternative, requiring a manual `pull`/`push` on a remote backend.

  * `removed { lifecycle { destroy = false } }` — removes it from state **without deleting** the real object (a key edge case).

  * `moved` — renaming/moving **within a single** state, not a cross-state migration.

* `terraform state push` is protected against **differing lineage** and a **higher serial** on the destination (bypassable with `-force`, not recommended without a backup).

* Configuration drift (external changes that break the config) ≠ state drift (external changes that don't break the config) — the latter is fixed via `-refresh-only`.

📄 [06a](notes/en/06a-local-backend.md)–[06d](notes/en/06d-resource-drift-state-mgmt.md)

---

## 7. Maintaining Infrastructure

* `terraform import` binds **exactly one** resource per call; it needs an already-existing (even if empty) `resource` block in the config **before** importing.

* The config-driven `import` block (≥1.5) + `plan -generate-config-out=FILE` is safer: you can preview the plan beforehand, and import+modification happen **in a single step** — but the generated config **cannot** be applied directly, only after review/pruning.

* Complex imports (e.g. a Network ACL) create **multiple** state entries — these need to be manually reflected in the config, otherwise Terraform will plan a **destroy**.

* `terraform state` subcommands (`list`, `show`, `mv`, `pull`, `push`, `rm`, `replace-provider`) work the same way for local/remote state (the only difference is network latency). All **modifying** subcommands write a backup automatically — this can't be disabled, only the path can be changed (`-backup`).

* `-replace` is the modern recommended replacement for the deprecated `terraform taint`.

* `TF_LOG` — the master log switch; `TF_LOG_CORE`/`TF_LOG_PROVIDER` — isolated per component; `TF_LOG_PATH` **enables nothing by itself** — you still need one of the levels too.

* Log levels (descending): **TRACE → DEBUG → INFO → WARN → ERROR**. `TRACE` — for bug reports.

* 4 layers of diagnostics, closest to the user first: **language (HCL) → state → core → provider**.

📄 [07a](notes/en/07a-import-existing-infrastructure.md)–[07c](notes/en/07c-verbose-logging.md)

---

## 8. HCP Terraform

* Three ways to run runs: **VCS-driven** (primary), **CLI-driven** (via the `cloud` block, streams logs locally), **API-driven**.

* A workspace processes runs **strictly one at a time**; except for **plan-only runs** and the **planning stage of a saved plan** — these ignore the queue.

* A **speculative plan** is preview-only, **never applied**. A **saved plan**, on the other hand, is **never auto-applied**, even if the workspace has auto-apply enabled; and it is automatically discarded if state changes before it's confirmed.

* A no-op plan does not trigger an apply by itself — it needs the explicit **Allow empty apply** mode (usually for upgrading the state file version).

* Sentinel enforcement levels, from strictest to softest: **hard-mandatory → soft-mandatory → advisory**. Sentinel/OPA only support **workspaces**; only **Terraform policy** (the native HCL framework) supports both **and Stacks**.

* A conflict on a single key across variable sets is resolved by the **lexicographic order of the set's name** (not creation order!). A workspace-specific variable always **wins** over a variable set; a **priority variable set** wins over almost everything, including CLI flags.

* Health assessments require: the last run was **successful**, execution mode is **Remote or Agent** (Local is not supported), at least one successful apply in the history.

* Projects: every workspace/Stack has **exactly one** project. Project permissions, descending: **Admin → Maintain → Write → Read**. Org-level "Manage Workspaces" permission still drops new workspaces into the **Default Project**.

* Execution mode is inherited **workspace ← project ← organization**; changing it at the project level only affects **future** workspaces.

* Run triggers require a separate opt-in "Auto-apply run triggers" — the trigger itself does **not** auto-apply the run it starts.

* For reading outputs across workspaces: `tfe_outputs` (recommended for HCP Terraform/Enterprise, pulls only the outputs via the API) is **preferable** to `terraform_remote_state` (which requires access to the **entire** state, plus explicit permission from the source workspace).

* The `cloud` block has no `prefix` argument (a legacy concept from the `remote` backend) — use `tags` instead. `terraform login` is **interactive only** — automation needs credentials configured manually. `terraform import` **always** runs **locally**, even with CLI integration configured.

📄 [08a](notes/en/08a-hcp-terraform-create-infrastructure.md)–[08d](notes/en/08d-cli-integration.md)