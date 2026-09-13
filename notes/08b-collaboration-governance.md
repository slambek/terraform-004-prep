# 8b. Describe HCP Terraform collaboration and governance features

> **Scope note:** Run mechanics (plans, applies, run modes) are 8a.
> Workspace/project structure and permission sets for projects are 8c.
> `cloud` block setup, state migration, dynamic credentials, and the
> `tfe_outputs`/`terraform_remote_state` data sources are 8d — this file
> covers *what* run triggers connect, not the data source mechanics.

## Core Concept

HCP Terraform layers team-oriented features on top of the core Terraform
workflow: shared secrets, mandatory review, policy-as-code enforcement,
cost visibility, drift/health monitoring, and organization-wide reporting.

## Teams and permissions

- A **team** is a group of HCP Terraform users within an organization;
  belonging to at least one team makes you an organization member.
- Teams get permissions scoped to workspaces, projects, or the whole
  organization — never across organizations.
- Every organization has an **owners team**; its members ("organization
  owners") have full org access. The creator is its first member. Free
  orgs cap this team at 5 members; paid orgs are unlimited. You can never
  delete or empty the owners team — always add a replacement before
  removing the last member.
- Teams can hold their own **API tokens**, independent of any user.
- Manage teams via UI, the Teams/Team Members/Team Access APIs, or the
  `tfe_team`, `tfe_team_members`, `tfe_team_access` provider resources.

## Policy enforcement

Policy-as-code validates that Terraform plans comply with security rules
and best practices, enforced automatically during runs.

| Framework | Notes |
|---|---|
| Terraform policy (beta) | Native HCL-based; requires Terraform ≥ v1.16alpha (opt into pre-releases) |
| Sentinel | HashiCorp's policy-as-code language |
| OPA | Open Policy Agent, uses Rego |

- **Enforcement levels** (Sentinel): `advisory` (warns, doesn't block),
  `soft-mandatory` (blocks but can be overridden with permission),
  `hard-mandatory` (always blocks on failure, cannot override).
- Policies are grouped into **policy sets**; a set applies globally or to
  specific projects/workspaces/Stacks, and can only contain policies from
  **one** framework — apply multiple sets (different frameworks) to the
  same workspace if needed.
- Recommended source of truth is a **VCS repository** (policy-as-code
  workflow: standardization, auditability); you can also author policies
  directly in the UI or manage sets via the API.
- Terraform policy and Sentinel/OPA both evaluate for **workspaces**;
  Sentinel and OPA do **not** support Stacks — only Terraform policy does,
  evaluating per deployment.
- Free plan includes one policy set of up to five policies; connecting
  policy sets to a VCS repo or managing via API requires Standard/Premium.

## Run tasks

Run tasks call out to **external systems** (vulnerability scanners, cost
tools, custom checks) between the plan and apply stages of a run, sending
the plan's contents and using the response to allow or block the apply.
Partner integrations exist, or you can build your own against the API.

## Cost estimation

Before applying, workspaces can show total estimated cost and the cost
**delta** caused by the proposed change, for major cloud providers. Cost
estimates can also feed Sentinel policies (e.g. warn on large cost jumps).
Requires Standard/Premium; disabled automatically on any run using
`-target`, since targeting excludes resources and would produce a
misleading estimate.

## Health assessments

Automatic checks (Standard/Premium) verifying real infrastructure still
matches configuration:

- **Drift detection** — real-world settings vs. Terraform configuration.
  This is **configuration drift**, distinct from *state* drift: config
  drift means external changes invalidate your configuration; state drift
  (external changes that don't invalidate config) is instead handled by
  refresh-only operations (see 6d).
- **Continuous validation** — re-evaluates `check` blocks (and
  pre/postconditions) after the fact, e.g. verifying a website still
  returns HTTP 200, or a certificate hasn't expired.

Requirements: Terraform ≥ 0.15.4 for drift detection alone, ≥ 1.3.0 for
both checks; **remote or agent** execution mode; the workspace's last run
must have succeeded, and it must have at least one successful apply (no
assessments on workspaces with no real infrastructure).

- Assessments run as **non-actionable refresh-only** plans — never touch
  real infrastructure or configuration.
- Scheduled roughly every 24 hours from the more recent of last apply or
  last assessment; you can also trigger **on-demand** assessments (workspace
  admin only), which reset the schedule.
- A new run during an assessment **cancels** the in-progress assessment.
- An errored latest run **pauses** assessments until a successful run
  occurs again.
- To resolve detected drift: either apply to **overwrite** it (revert to
  config), or **update configuration** to adopt the drifted value.

## Variable sets

Reusable groups of variables applied across multiple workspaces (and
Stacks) at once, defined at the **organization** or **project** level.

- Scope options: apply globally (org-owned), to specific
  projects/workspaces/Stacks, or to an entire project (project-owned).
- **Precedence when two sets define the same key**: HCP Terraform resolves
  by **lexical (alphabetical) order** of the variable set names — the
  earlier name wins and shows the other as "OVERWRITTEN".
- A **workspace-specific** variable of the same key always overrides any
  variable set value.
- **Priority variable sets** invert this further: their values override
  even more specific scopes, including CLI flags and `.tfvars`/
  `.auto.tfvars` files.
- Local-execution-mode workspaces do **not** evaluate variable sets.
- Values are encrypted at rest (Vault transit backend); descriptions are
  stored in **plain text** — never put secrets in a variable description.
- Prefer environment variables over Terraform variables for credentials —
  Terraform variable values can appear in logs, state, or Sentinel mocks.

## Private registry

Mirrors the public Terraform Registry experience internally:

- **Public** modules/providers can be synced into your org's private
  registry automatically, centralizing docs/examples and signaling which
  ones are org-recommended.
- **Private** modules/providers are visible only within your org (or
  orgs configured to share, in Terraform Enterprise).
- The registry uses your **VCS** as source of truth, versioning modules via
  Git tags.
- Sentinel policies can restrict usage — e.g. requiring all non-root
  modules come from the private/public registry, or requiring recent
  versions.

## Explorer for workspace visibility

An organization-wide reporting view (requires **Organization owner** or
**View all workspaces** permission) covering four resource types:
Workspaces, Modules, Providers, Terraform versions.

- Built-in use cases: top module/provider versions by usage, latest
  Terraform versions in use, workspaces without VCS, workspaces with
  failed checks, drifted workspaces, and more.
- Custom queries use a filter-condition builder (field + operator + value);
  multiple conditions combine with **AND**.
- You can **save views** (query + column selection) to revisit later —
  HCP Terraform doesn't store historical results, only the query itself;
  reopening a saved view re-runs it live.

## Change requests

(Standard/Premium) A backlog of action items recorded directly on a
workspace — e.g. flagging a deprecated module version or a needed security
fix — so administrators can notify the responsible team. Created from
Explorer queries; the owning team archives a request once resolved.

## Run triggers

Connect a workspace to up to **20 source workspaces**; any successful apply
in a source workspace automatically **queues a run** in the connected
(downstream) workspace. Useful when a configuration relies on another
workspace's outputs via a data source.

- Configured on the **downstream** workspace (admin access required); you
  also need read-runs permission on the source workspace.
- Triggered runs do **not auto-apply** by default — a separate "Auto-apply
  run triggers" setting controls that, independent of the workspace's
  normal auto-apply setting.
- The queued run's details link back to the source workspace and the
  triggering apply.
- Actual cross-workspace **data access** (reading the source's outputs) is
  a separate mechanism — see 8d for `tfe_outputs` / `terraform_remote_state`.

## Exam Quick Facts

- Sentinel enforcement levels, strictest to loosest: **hard-mandatory →
  soft-mandatory → advisory**.
- A policy set holds policies from only **one** framework at a time, but a
  workspace/Stack can have multiple policy sets (different frameworks)
  attached.
- Sentinel and OPA support **workspaces only**; Terraform policy is the one
  framework that also supports **Stacks**.
- Variable set name-collision precedence is **lexical order of the set
  name** — not creation order, not scope specificity (project vs org).
- A **workspace-specific variable** always beats a variable set value for
  the same key; a **priority variable set** beats almost everything else,
  including CLI/.tfvars.
- Health assessments require the **last run to have succeeded**, and
  **remote or agent** execution mode — local-execution workspaces can't
  use them.
- Configuration drift ≠ state drift: assessments detect the former;
  `-refresh-only` operations reconcile the latter.
- Run triggers require an explicit "Auto-apply run triggers" opt-in;
  otherwise the queued downstream run still waits for manual apply.
