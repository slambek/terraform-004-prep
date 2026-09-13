# 8c. Describe how to organize and use HCP Terraform workspaces and projects

> **Scope note:** Run/plan/apply mechanics are 8a. Teams, policy, health
> assessments, variable sets, and other collaboration/governance features
> are 8b (project **permission sets**, though, are covered here since
> they're core to *using* projects). CLI/API integration setup is 8d.

## Core Concept

A **workspace** is HCP Terraform's unit of infrastructure management — it
bundles configuration, state, and variables the way a persistent working
directory does for local Terraform, but as a first-class, permissioned
object. A **project** groups related workspaces (and Stacks) so you can
manage their access and settings collectively.

## Workspace contents

| Component | Local Terraform | HCP Terraform |
|---|---|---|
| Configuration | On disk | VCS repo, or periodically uploaded via API/CLI |
| Variable values | `.tfvars`, CLI args, shell env | Stored in the workspace |
| State | On disk or remote backend | Stored in the workspace |
| Credentials/secrets | Shell env or prompts | Stored in the workspace as sensitive variables |

HCP Terraform additionally retains, per workspace:

- **State versions** — backups of every previous state file (for history
  and recovery, not just the current state).
- **Run history** — every run's summary, logs, triggering change, and
  comments.

The workspace's resource count (shown at the top of its page) reflects
resources in the current state file — **both** managed resources and data
sources.

## HCP Terraform workspaces vs. Terraform CLI workspaces

These are two different features that happen to share a name.

| | HCP Terraform workspaces | Terraform CLI workspaces |
|---|---|---|
| Required? | Yes — you can't manage resources without at least one | No — entirely optional |
| Represents | A completely separate working directory equivalent; a unit of RBAC | Multiple state files sharing one working directory/config |
| Scope | All infrastructure collections in an org | Isolating environments (dev/staging/prod) under one config |

## Planning and organizing workspaces

Recommended practice: break monolithic configurations into smaller ones,
each with its own workspace and delegated ownership — e.g. split a
production environment into `networking-prod`, `app1-prod`,
`monitoring-prod` workspaces owned by different teams. This mirrors
microservice decomposition: teams change things in parallel, and
configurations become reusable for other environments (`app1-dev`, etc.).
Terraform Enterprise admins can cap the max workspaces per organization.

## Projects

Every workspace and Stack belongs to **exactly one** project.

- New workspaces default to the organization's **Default Project** (you
  can rename it, but never delete it).
- Specify a workspace's project at creation, or move it later.
- Project-level permissions are **more granular than org-level**, but
  **broader than per-workspace grants** — useful for scoping teams to a
  business unit, department, or technical area without per-workspace setup.
- In HCP Europe orgs, projects are created/managed through the HCP
  platform itself, not HCP Terraform directly.

### Project permission sets

| Set | Grants |
|---|---|
| **Admin** | Full project administration: read/modify/delete the project, create workspaces in it, move workspaces in/out, manage team access |
| **Maintain** | Create/manage workspaces and provision infrastructure within them — cannot delete the project, change its permissions, or move workspaces in/out |
| **Write** | Provision infrastructure in the project's workspaces only — no workspace/project management |
| **Read** | View project name and workspace details only — useful for teams that reference data without managing resources |

- The **"Manage Workspaces"** org-level team permission lets users create
  workspaces, but new ones always land in the **Default Project**; reaching
  other projects requires **"Manage Projects & Workspaces"** or admin on
  that specific project.
- The **"Manage all Projects"** org-level permission lets a team view,
  edit, delete, and assign access for **every** project in the org.
- Organization-wide permissions (e.g. "Manage all Workspaces") **supersede**
  narrower project/workspace grants — always check for org-level access
  when auditing who can reach a resource.
- Cross-workspace features like remote state sharing and run triggers are
  gated by these same access permissions — a team needs access to **both**
  workspaces involved to configure the link between them.

## Execution mode

Controls where plan/apply actually runs; settable at the **organization**,
**project**, or **workspace** level, each layer defaulting to inherit from
the one above unless overridden.

| Mode | Behavior |
|---|---|
| Organization Default | Inherits the org's setting (Remote or Local) |
| Remote | Runs on HCP Terraform's own infrastructure; full collaboration features |
| Local | HCP Terraform only stores/synchronizes **state**; plan/apply run on your own machines |
| Agent | An HCP Terraform agent you host executes runs, polling for work — no public ingress needed |

Any workspace created in a project **after** you change that project's
execution mode inherits the new default; existing workspaces are
unaffected. Stacks do not support Local execution mode.

## Exam Quick Facts

- A workspace is **required** in HCP Terraform (can't manage infra without
  one); a CLI workspace is **optional** and only isolates state files
  within one config.
- Every workspace/Stack has **exactly one** project — never zero, never
  more than one.
- Project permission sets, most to least privileged: **Admin → Maintain →
  Write → Read**.
- "Manage Workspaces" (org permission) still funnels new workspaces into
  the **Default Project** — it does not grant access to other projects.
- Execution mode inheritance: **workspace ← project ← organization**, and
  changing a project's mode only affects workspaces created **after** the
  change.
- Only **Remote or Agent** execution mode supports HCP Terraform's
  collaboration features (Sentinel, cost estimation, notifications, health
  assessments) — **Local** mode reduces the workspace to a state backend.
