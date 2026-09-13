# 7b. Use the CLI to inspect state

> **Scope note:** State's purpose, refresh/`-refresh-only`, drift, and the
> `moved`/`removed` blocks and cross-state migration live in 6d — this file
> is about the `terraform state` subcommand family and related inspection
> commands as day-to-day tools, not the drift/refactor workflows they get
> used in. `terraform import` mechanics are in 7a.

## Core Concept

`terraform state <subcommand>` provides advanced, script-friendly state
inspection and modification without hand-editing the state JSON. All
subcommands work identically against local or remote state — remote reads
and writes just take a network round trip.

## Subcommands

```
terraform state list
terraform state show ADDRESS
terraform state mv SOURCE DESTINATION
terraform state pull
terraform state push
terraform state rm ADDRESS
terraform state replace-provider
```

| Command | What it does |
|---|---|
| `list` | Lists resource addresses tracked in state (read-only) |
| `show ADDRESS` | Prints one resource's attributes from state |
| `mv` | Renames a resource address, or moves it between two state files |
| `pull` | Downloads current state, prints to stdout |
| `push` | Overwrites remote state from a local file (dangerous, protected by lineage/serial checks — see 6d) |
| `rm` | Removes a resource from state without destroying the real object |
| `replace-provider` | Updates the provider address recorded for resources |

`terraform show` (no `state` prefix) gives a **human-readable** dump of the
whole state — or, given a saved plan file, the plan's prior state — and
supports `-json` for machine-readable output.

## Design characteristics

- **Command-line friendly**: output is structured for piping into `grep`,
  `awk`, and similar tools; prefer chaining these over parsing the state
  JSON directly.
- **Automatic backups**: every subcommand that *modifies* state (`mv`, `rm`,
  `push`, `replace-provider`) writes a backup file first. You cannot disable
  this — remove old backup files manually if you don't want them. Read-only
  commands (`list`, `show`, `pull`) write **no** backups.
- Backup file path is controlled with `-backup=FILENAME` (same mechanism as
  the local-backend legacy flag in 6a).
- Remote state behaves identically to local for all `state` subcommands —
  same CLI usage, just slower due to network calls.

## `terraform state mv` — rename or relocate

```bash
# Rename within the same state
terraform state mv aws_instance.old_name aws_instance.new_name

# Move into a different state file (legacy cross-state approach — full
# workflow with pull/push in 6d)
terraform state mv -state-out=../other/terraform.tfstate \
  aws_instance.example aws_instance.example
```

- Updates the **state only** — it does not touch your `.tf` configuration.
  After moving, you must add/remove the corresponding resource blocks
  yourself or the next `plan` will propose to destroy/recreate.
- Destination names must be unique within the destination state; `mv` can
  rename as it moves to avoid collisions.

## Replacing a resource via CLI

> Full `-replace` semantics (and its relationship to the deprecated
> `terraform taint`) are covered in **3d/3e** — this is just the practical
> `state list` → `-replace` workflow for finding the address to target.

```bash
terraform state list                        # find the address
terraform plan  -replace="aws_instance.example"
terraform apply -replace="aws_instance.example"
```

## Exam Quick Facts

- Every state-**modifying** `state` subcommand always writes a backup; you
  cannot turn this off — only relocate it with `-backup`.
- Read-only subcommands (`list`, `show`, `pull`) never write backups.
- `state mv` changes **state**, never configuration — config drift after a
  move is on you to reconcile.
- `-replace` needs a resource **address** — `state list` is the fastest way
  to find one (full `-replace`/`taint` semantics: 3d/3e).
- `terraform show -json` can read either the latest state **or** a saved
  plan file's captured prior state.
- All `terraform state` subcommands work the same against local or remote
  state — the only difference is round-trip latency.
