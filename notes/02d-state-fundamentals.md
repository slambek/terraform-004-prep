# 2d. Explain how Terraform uses and manages state

## Core Concept

Terraform **requires** a state file to function — it is not optional. State is the database that lets Terraform know which real-world object corresponds to which resource block in your configuration.

> **Scope note:** this objective is the *conceptual* foundation of state. Backend types (local vs remote), state locking, and drift-handling workflows are their own exam objectives — **Section 6 (6a–6d)**. Day-to-day CLI inspection commands (`terraform show`, `state list`, `import`) belong to **Section 7 (7a–7b)**; `state mv`'s basic usage is also in 7b, but its cross-state-file migration workflow lives in **6d**. This note only covers *why state exists and what it contains*.

## Why Terraform Needs State

### 1. Mapping Configuration to the Real World

A resource block like `resource "aws_instance" "foo"` is just a label in your configuration. Terraform needs a way to know that this label corresponds to the real object with ID `i-abcd1234`. State provides that mapping.

* Early Terraform prototypes tried using cloud tags instead of a state file — this failed because **not all resources/providers support tags**.
* **Each remote object must map to exactly one resource instance.** If the same real-world object is bound to two resource instances, the mapping becomes ambiguous and Terraform behaves unpredictably.

### 2. Metadata

Beyond the ID mapping, state tracks metadata Terraform needs to operate correctly, including:

* **Resource dependencies** — normally inferred from configuration, but if you delete a resource block entirely, the configuration no longer has that information. Terraform keeps a copy of the last-known dependency order *in state* so it still knows the correct order to destroy things in.
* A pointer to which **provider configuration** (including aliases) was most recently used for a resource.

### 3. Performance (Attribute Caching)

State caches the attribute values of every resource. On `plan`, Terraform normally queries providers to refresh this cache and compare real infrastructure against configuration.

* For large infrastructures, refreshing every resource on every run is slow (API round-trips + rate limits), so cached state is sometimes treated as the record of truth (e.g. via `-refresh=false` or `-target`).
* This caching is the **most optional** of state's roles — everything else (mapping, metadata) is a hard requirement.

## What's Actually Inside the State File

State is stored as JSON (`terraform.tfstate` by default, locally). Top-level structure:

```json
{
  "version": 4,
  "terraform_version": "1.7.0",
  "serial": 18,
  "lineage": "0c41e079-...",
  "outputs": {},
  "resources": []
}
```

Each entry in `resources` records:

* `mode` — `"managed"` (a `resource` block) or `"data"` (a `data` block).
* `type` and `name` — e.g. `aws_instance` / `example`.
* `provider` — which provider (and alias) manages it.
* `instances` → `attributes` — the cached attribute values.
* `dependencies` — the resources it depends on (see Metadata above).

**Rule of thumb:** never hand-edit this file. Manual edits can desynchronize state from real infrastructure and cause unintended destroy/recreate on the next `apply`.

## State and the Core Workflow

* `terraform plan` and `terraform apply` automatically **refresh** state by default before comparing it to configuration — this is how drift is detected (full drift-handling workflow is 6d).
* Terraform uses the diff between **state**, **configuration**, and **real infrastructure** to compute what needs to change. State is what makes this a *targeted* diff instead of a full teardown/rebuild every time.

## Local State (Default) vs Remote State (Preview)

By default, Terraform stores state as a **local file** in the current working directory. This is fine to get started, but doesn't work for teams — everyone needs to operate on the *same* state to avoid conflicting changes.

* **Local backend** mechanics and **remote backends** (shared state, locking) are covered in full under **Section 6 (6a, 6c)** — this note only flags that the choice exists.

## Exam Quick Facts

* State is **required**, not optional — Terraform cannot reliably manage infrastructure without it.
* State's three jobs: **(1) map config → real objects, (2) track metadata (dependencies, provider), (3) cache attributes for performance.**
* Each real-world object must map to **exactly one** resource instance in state.
* `plan`/`apply` refresh state automatically by default.
* Never manually edit the state file.
* Local file storage is the default; shared/remote state is a separate topic (Section 6).