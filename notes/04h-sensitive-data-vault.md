# 4h. Manage sensitive data, including secrets management with Vault

## The Core Problem

State and plan files can contain **detailed infrastructure data**, including secrets (passwords, tokens) — and by default Terraform stores state as **plaintext** locally. Terraform gives you three distinct tools depending on what you actually need:

| Goal | Tool |
| :--- | :--- |
| Hide a value from CLI/UI output, but it's still fine to store in state | `sensitive = true` (variable/output) |
| Never store the value in state or plan files at all | `ephemeral = true`, an `ephemeral` block, or a write-only argument |
| Fetch/manage the secret itself (not just hide it) | An external secrets tool, e.g. the **Vault provider** |

## `sensitive` — Redact From Output, Still Stored in State

```hcl
variable "database_password" {
  type      = string
  sensitive = true
}
```

* Redacts the value (and anything derived from it) in `plan`/`apply` CLI output — shown as `(sensitive value)`.
* **Still recorded in cleartext in state and plan files.** `terraform output -json` / `-raw` also reveal it in plaintext (see 4c).
* If you need both hiding *and* non-storage, combine `sensitive` with `ephemeral` (below) on a variable or child-module output.

## Ephemeral Values — Omit From State and Plan Entirely (Terraform 1.10+)

Ephemeral values exist only at **run time**; Terraform never writes them to state or plan files. Because nothing is persisted, you must capture any value you actually want to keep (e.g. a generated password) in a separate, non-ephemeral resource/output if you need it later.

Four ways to create an ephemeral value:

1. **`ephemeral` argument on a variable** — usable in other ephemeral contexts (e.g. a `provider` block).
2. **`ephemeral` argument on a child-module output** — **not allowed on root-module outputs.**
3. **The `ephemeral` block** — declares a temporary ephemeral *resource* (e.g. `ephemeral "random_password" "db_password" {}`) that exists only for the current operation.
4. **A write-only argument on a managed resource** (Terraform 1.11+) — provider-defined, conventionally named `..._wo` with a paired `..._wo_version` integer. The provider uses the value during the operation, then Terraform discards it — never stored.

```hcl
ephemeral "random_password" "db_password" {
  length = 16
}

resource "aws_db_instance" "example" {
  password_wo         = ephemeral.random_password.db_password.result
  password_wo_version = 1   # bump this to signal the value changed
}
```

Because Terraform can't "diff" a write-only value (it's never stored), the paired `_version` argument is how you tell Terraform the value changed — incrementing it triggers the provider to use the new value in the next apply.

Ephemeral values can only be **referenced** in a limited set of contexts: `locals`, another ephemeral variable, an ephemeral child-module output, a write-only resource argument, another `ephemeral` block, `provider` block configuration, and provisioner/connection blocks.

## Vault Provider — Injecting Real Secrets, Not Just Hiding Them

The **Vault provider** lets Terraform read from, write to, and configure HashiCorp Vault — commonly used to fetch **short-lived, dynamically-generated credentials** (e.g. scoped AWS IAM keys) instead of storing long-lived static credentials on a developer's machine or in configuration.

**Pattern:** a Vault Admin configures an AWS Secrets Engine + Vault role (scoped IAM policy, short TTL) in Vault; a Terraform Operator's configuration then requests short-lived AWS credentials from that Vault role at plan/apply time and uses them to configure the `aws` provider — Terraform never touches a long-lived AWS key.

**Benefits:** operators manage one Vault role instead of many long-lived, multi-scoped static credentials; a leaked run-scoped credential is only useful for the length of its TTL; permissions can be tightened centrally by editing the Vault role, immediately restricting what future Terraform runs can do.

**Caveats:**

* Terraform still has **no mechanism to redact secrets read via a provider/data source** — anything read from Vault (or written to it) is recorded **in cleartext in state and plan files**, exactly like any other value. Treat those files as sensitive regardless of Vault's involvement.
* The generated token/credential TTL must be long enough to cover the **entire** apply — if an apply (or the wait for confirmation) runs longer than the TTL, the credentials expire mid-run and the operation fails. Increasing the TTL to compensate also widens the exposure window if the credential leaks — a real tradeoff, not a free fix.

## State Security Best Practices (Applies Regardless of Sensitivity Tooling)

Because `sensitive`, ephemeral values used incorrectly, and even Vault-sourced secrets can all still end up in state:

* **Store state remotely** rather than as a local plaintext file.
* **Encrypt state at rest** — e.g. HCP Terraform encrypts automatically and lets you supply your own keys; the S3 backend can encrypt with `encrypt = true`; GCS supports customer-managed keys.
* **Use access controls** to restrict who can read state.
* **Use audit logs** to track state access over time.
* Never commit `.tfvars` files containing secrets, or plan files (binary or JSON — see 3d), to version control.

## Exam Quick Facts

* `sensitive` hides values from **CLI/UI output only** — the value is still in state/plan files in cleartext.
* `ephemeral` (variable/output/block) and write-only resource arguments (`_wo` + `_wo_version`) are the only ways to keep a value **out of state and plan files entirely**.
* `ephemeral` outputs are child-module-only — never valid on the root module.
* A write-only argument's paired `_version` argument is how you signal "this value changed" since Terraform can't diff a value it never stores.
* The Vault provider supplies **short-lived, dynamically scoped** credentials instead of long-lived static ones — but does **not** redact anything it reads/writes from state or plan files.
* Regardless of tooling, treat state and plan files as sensitive artifacts: store remotely, encrypt at rest, restrict access, audit.
