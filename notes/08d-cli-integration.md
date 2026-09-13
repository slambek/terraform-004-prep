# 8d. Configure and use HCP Terraform integration

> **Scope note:** What workspaces/projects/execution mode *are* is 8c;
> what runs and run modes do is 8a; collaboration/governance features
> (run triggers as a concept, variable sets, policy) are 8b. This file is
> about wiring the Terraform **CLI** to HCP Terraform and connecting
> workspaces to each other's data.

## Core Concept

The `cloud` block in your `terraform` configuration block links a local
working directory to one or more HCP Terraform workspaces, enabling the
**CLI-driven run workflow**: local `terraform` commands execute remotely,
with HCP Terraform managing state so you don't maintain a separate backend.

## Connecting the CLI

1. **Provide credentials** — run `terraform login` (recommended) or set a
   user token directly in configuration.
2. **Add a `cloud` block** (a member of the `terraform` block):

```hcl
terraform {
  cloud {
    organization = "my-org"
    hostname     = "app.terraform.io"   # optional, this is the default

    workspaces {
      project = "networking-development"
      tags = {
        layer  = "networking"
        source = "cli"
      }
    }
  }
}
```

   - `organization` — required, must already exist.
   - `workspaces.name` — pin to one existing workspace by name. **Mutually
     exclusive** with `tags`.
   - `workspaces.tags` — map (or legacy key-only list) of tags; Terraform
     links to any existing workspaces with matching tags, or offers to
     create one with those tags on init if none match.
   - `workspaces.project` — scopes tag/name matching to workspaces within
     a named project.
   - The `cloud` block does **not** support a `prefix` argument (unlike the
     legacy `remote` backend) — migrate `prefix` usage to `tags` instead.
3. **Run `terraform init`** — required after adding/changing the `cloud`
   block.
4. **Migrate state** (optional, only if a prior backend already has state).

### `terraform login`

```bash
terraform login [hostname]
```

- Interactive only — opens a browser on the machine running Terraform; use
  manual credential configuration for unattended/automation scenarios.
- Defaults to `app.terraform.io` if no hostname given; HCP Europe orgs use
  `app.terraform.io/eu`.
- By default writes the API token in **plain text** to
  `credentials.tfrc.json`; you can configure a credentials helper to store
  tokens elsewhere (e.g. your org's secrets manager) instead.
- Works with any server implementing the login protocol, not just HCP
  Terraform/Terraform Enterprise.

## Migrating state

| Starting point | What to do |
|---|---|
| Local or `local`/other backend, no HCP Terraform yet | Add `cloud` block, `terraform init`, confirm migration prompt |
| Already on the legacy `remote` backend | Replace `backend "remote" {}` with `cloud {}` (keeps using the same workspaces) |
| Bulk migration across many state files | Use the `tf-migrate` CLI tool (separate download) |
| No CLI access desired | Migrate via the HCP Terraform **API** directly |

**Local → HCP Terraform (CLI):** `terraform init` detects the new `cloud`
block and prompts to migrate. Since HCP Terraform workspaces **must** have
a name, Terraform may prompt you to rename CLI workspaces (which represent
environments under one config) into distinct HCP Terraform workspaces —
common pattern: `<COMPONENT>-<ENVIRONMENT>-<REGION>` (e.g.
`networking-prod-us-east`).

**`remote` backend → `cloud` block:**

```hcl
terraform {
-  backend "remote" {
+  cloud {
     organization = "my-org"
     workspaces {
-      prefix = "my-app-"
+      tags = {
+        app = "mine"
+      }
     }
   }
}
```

After migrating a `prefix`-based config to tags, refer to workspaces by
their **full name** with the CLI (e.g. `terraform workspace select
my-app-prod`, not the old prefix-relative form).

**Via API (manual, scriptable):** base64-encode the state file, compute an
MD5 hash, `POST` to create the workspace if needed, **lock** it, `POST` the
state (with the MD5) to create a state version, then **unlock** it.

**Requirements for any migration:** Terraform ≥ 1.1 for the `cloud` block
(use `remote` backend on older versions); stop all Terraform operations
against the source state first; only migrate into workspaces that have
**never** performed a run.

## Excluding files from upload

CLI-driven remote plan/apply uploads a copy of your working directory.
Add a `.terraformignore` file at the config root to exclude paths
(`.gitignore`-style rules: `#` comments, blank lines ignored, trailing `/`
for directories, leading `!` to negate — avoid negation in large trees, it
hurts performance). Without this file, Terraform still excludes `.git/`
and `.terraform/` (except `.terraform/modules`) by default.

## Import via CLI integration

`terraform import` does **not** support remote execution — even with a
`cloud` block configured, import always runs **locally**; the workspace
only stores the resulting state. Because it runs locally, workspace
environment variables aren't available — any provider credentials the
import needs must be set in your local shell.

## Cross-workspace data sharing

Two data sources read another workspace's root-level outputs:

| Data source | Notes |
|---|---|
| `terraform_remote_state` | Works with local or any remote backend; needs the reading workspace to be explicitly granted access to the source's state; grants access to **all** of that state, not just outputs |
| `tfe_outputs` (from the `tfe` provider) | **Recommended** for HCP Terraform/Enterprise — fetches only outputs via the API, without requiring full state access |

Before either works cross-workspace in HCP Terraform, the source workspace
must explicitly allow the consumer via **remote state sharing** settings.
`tfe_outputs` calls also need a `TFE_TOKEN` (team or user API token) set as
an environment variable in the consuming workspace so it can call the API.
This mechanism is what a **run trigger** (8b) typically pairs with — the
trigger queues the downstream run, and one of these data sources is how
that run actually reads the upstream workspace's values.

## Dynamic credentials

An alternative to storing long-lived cloud credentials as workspace
variables. Uses **OIDC**: each run authenticates to the cloud provider
(AWS, GCP, Azure, or Vault) using a signed workload identity token, and
receives short-lived, per-run credentials in return — nothing static to
leak or rotate.

- You first establish a **trust relationship**: the cloud provider is
  configured to accept HCP Terraform's OIDC tokens (verified via HCP
  Terraform's TLS cert / OIDC discovery), scoped to a role and policy.
- The trust conditions check OIDC claims, notably:
  - `aud` (audience) — a unique string identifying this provider
    configuration, preventing a token meant for one provider from working
    against another.
  - `sub` (subject) — encodes org, project, workspace, and run phase
    (`organization:...:project:...:workspace:...:run_phase:plan|apply`),
    scoping exactly which workspace(s) can use the trust relationship.
- Enable per-workspace via environment variables, e.g. for Vault:
  `TFC_VAULT_PROVIDER_AUTH=true`, plus provider address/role/namespace
  variables. Each supported provider has its own `TFC_<PROVIDER>_*` set.
- Always scope claims to your specific organization/workspace — omitting
  this could let another HCP Terraform organization's runs authenticate
  against your trust relationship.
- Changing your org/project/workspace **name** can break an existing trust
  relationship, since the `sub` claim is name-based.

## Exam Quick Facts

- `cloud { workspaces { name = ... } }` and `tags = { ... }` are **mutually
  exclusive** — pick one addressing mode per configuration.
- The `cloud` block has **no `prefix` argument** — that's a `remote`-backend-
  only legacy concept; use `tags` instead when migrating.
- `terraform login` is **interactive-only**; automation must configure
  credentials manually instead.
- `terraform import` **always runs locally** even with the CLI integration
  configured — workspace env vars are not available to it.
- Prefer `tfe_outputs` over `terraform_remote_state` for HCP
  Terraform/Enterprise cross-workspace reads — it doesn't require exposing
  the entire source state.
- Dynamic credentials trade static, long-lived cloud keys for short-lived,
  per-run OIDC-derived credentials — nothing to store or rotate as a
  workspace variable.
- Renaming an org/project/workspace can silently break a dynamic
  credentials trust relationship built on its old `sub` claim value.
