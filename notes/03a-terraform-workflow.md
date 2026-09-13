# 3a. Describe the Terraform workflow

## The Core Loop: Write → Plan → Apply

```text
Write   → author Terraform configuration
Plan    → preview the changes before touching anything
Apply   → provision the reproducible infrastructure
```

This is a **loop**, not a one-time sequence — every change to infrastructure starts the cycle over.

> **Scope note:** this note covers the *concept* of the workflow and how it scales from individual → team → HCP Terraform. The mechanics of each command (`init`, `validate`, `plan`, `apply`, `destroy`, `fmt`) are covered individually in **3b–3g**.

## As an Individual Practitioner

* **Write** — edit config in your editor, `terraform init`, then iterate with `terraform plan` as a tight feedback loop to catch syntax errors early.
* **Plan** — once the loop looks good, commit to version control. `terraform apply` shows the final plan for confirmation before touching anything.
* **Apply** — approve with `yes`, Terraform provisions the infrastructure, then you push your repo for safekeeping.

This closely parallels writing application code as an individual: edit → test → commit → push.

## As a Team

Collaboration adds process around each step:

* **Write** — team members work in **version control branches** to avoid collisions; merge conflicts are resolved through the normal git workflow. Running plans locally becomes harder as sensitive input variables (API keys, certs) pile up, which pushes teams toward running Terraform in a shared **CI environment** instead of individual laptops.
* **Plan** — **speculative plans** (a plan generated without `-out`, with no intent to apply) get attached to pull requests so teammates can review the *intent* of a change and its *risk* before it merges — this is where teams decide whether the change should happen now or wait for a maintenance window.
* **Apply** — after merge, the team reviews the final **concrete plan** run against the shared branch and latest state — this can differ from the PR's speculative plan (merge order, infra drift since review). The team asks: will this disrupt service? Who needs to know? Then applies, sometimes watching together.

For some teams this loop runs a few times a week; for others, many times a day.

## Enhanced by HCP Terraform

HCP Terraform doesn't change the three steps — it centralizes and streamlines the collaboration points around them:

* **Write** — centralized, secure storage for **input variables and state**; the `cloud` block in `terraform {}` connects the CLI to an HCP Terraform workspace so teammates all plan against the same latest state.
* **Plan** — automatically runs a speculative plan when a PR is opened and posts a status update (changed / no changes) directly on the PR, with a link to the full plan detail.
* **Apply** — presents the concrete plan for team review/discussion after merge, then streams the apply progress live to anyone watching.

## Exam Quick Facts

* The three-step loop is always **Write → Plan → Apply**, and it repeats for every change.
* **Speculative plan** = a plan run without intent to actually apply it (no `-out` needed) — used for PR review.
* The concrete plan applied after merge can legitimately differ from the speculative plan reviewed pre-merge.
* HCP Terraform enhances (centralized state/variables, plan visibility on PRs, live apply streaming) — it does not replace the core Write/Plan/Apply loop.
