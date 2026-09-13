# 7c. Describe when and how to use verbose logging

> **Scope note:** This covers `TF_LOG*` environment variables and the
> troubleshooting/bug-report workflow. State-specific troubleshooting
> (drift, refresh) is 6d; state CLI inspection is 7b.

## Core Concept

Terraform emits detailed internal logs to stderr when the `TF_LOG`
environment variable is set. Logging is off by default; you turn it on
specifically when troubleshooting unexpected behavior or preparing a bug
report, since verbose logs are noisy and can contain sensitive data.

## Environment variables

| Variable | Purpose |
|---|---|
| `TF_LOG` | Master switch — set to a level to enable logging for both core and providers |
| `TF_LOG_CORE` | Enable logging for Terraform core only, at the given level |
| `TF_LOG_PROVIDER` | Enable logging for provider plugins only, at the given level |
| `TF_LOG_PATH` | Append all enabled logs to this file instead of only stderr |

```bash
export TF_LOG_CORE=TRACE
export TF_LOG_PROVIDER=TRACE
export TF_LOG_PATH=logs.txt
```

- `TF_LOG_PATH` only takes effect if `TF_LOG` (or `TF_LOG_CORE`/
  `TF_LOG_PROVIDER`) is also set — setting the path alone enables nothing.
- Unset a variable (e.g. `export TF_LOG_CORE=`) to stop that log stream;
  all of these are session-scoped and clear when the terminal closes.

## Log levels

In decreasing order of verbosity: **TRACE, DEBUG, INFO, WARN, ERROR**.

- `TRACE` is the level requested for **bug reports** — it's the most
  detailed and what the Terraform team needs to diagnose an issue.
- Setting `TF_LOG=JSON` outputs TRACE-and-above logs in a parseable JSON
  encoding — but this format is explicitly **not a stable interface** and
  may change without notice; it's meant for tooling, not general use.

## The four-layer troubleshooting model

When something goes wrong, narrow down which layer the error originates in:

1. **Language (HCL) errors** — Terraform core parses your configuration and
   reports line numbers and a syntax explanation directly.
2. **State errors** — state out of sync with real infrastructure; resolved
   via refresh, import, or `-replace` (see 6d / 7b), not via logging.
3. **Core errors** — bugs in Terraform's own graph/logic; a candidate for a
   GitHub issue against the Terraform core repo.
4. **Provider errors** — bugs in a specific provider plugin (auth, API
   mapping); reported against that provider's own repo.

Rule out language and state issues first — logging is primarily useful for
diagnosing the **core** and **provider** layers.

## Bug-reporting workflow

1. Confirm versions: `terraform version` (also flags if your CLI/providers
   are outdated).
2. Enable the appropriate log stream(s) at `TRACE` and set `TF_LOG_PATH`.
3. Reproduce the issue (e.g. `terraform refresh`, `plan`, or `apply`).
4. Inspect the log file: entries containing `provider.terraform-provider-<name>`
   indicate a provider-side issue; otherwise it's likely core.
5. File the issue against the correct repository (Terraform core vs. the
   specific provider), following that repo's bug report template, and
   attach the relevant log excerpt.

## Exam Quick Facts

- `TF_LOG_PATH` alone does nothing — you must also set `TF_LOG`,
  `TF_LOG_CORE`, or `TF_LOG_PROVIDER`.
- `TF_LOG_CORE` and `TF_LOG_PROVIDER` let you isolate **which** component's
  logs you collect; `TF_LOG` turns on both.
- `TRACE` is the level to use for a bug report — it's the most verbose.
- `TF_LOG=JSON` gives TRACE-level output in JSON, but the format is
  explicitly unstable/tooling-only, not a documented stable schema.
- The four troubleshooting layers, closest-to-user first: **language →
  state → core → provider** — always rule out language/state before
  assuming a core or provider bug.
- Environment variables set with `export` are session-scoped; they don't
  persist once the terminal closes unless added to shell profile config.
