# 6b. Describe state locking

> **Scope note:** Which backend types exist and how to configure them is in
> 6c. Local-backend-specific locking (system/file APIs) is noted in 6a but
> the general locking behavior described here applies to any locking-capable
> backend.

## Core Concept

State locking prevents two Terraform operations from writing to the same
state at once, which could corrupt it. Locking is **automatic** for every
operation that could write state, and is **optional** — it depends entirely
on whether the backend implements it.

## Behavior

- Locking happens silently — you don't see a message unless acquiring the
  lock is taking longer than expected, in which case Terraform prints a
  status message. No message does **not** mean locking isn't happening.
- If lock acquisition fails, Terraform **halts** the operation; it does not
  proceed unlocked.
- You can disable locking for most commands with `-lock=false`. This is not
  recommended.
- Not all backends support locking — check each backend's own docs page.

```bash
terraform apply -lock=false
```

## Force Unlock

If a lock fails to release automatically (e.g. crashed process, network
drop), use:

```bash
terraform force-unlock LOCK_ID
```

- Terraform requires the unique **lock ID** as a nonce so you can only target
  a specific, known lock — Terraform prints this ID when a locking
  operation fails.
- Only use `force-unlock` to clear **your own** stuck lock. Force-unlocking
  a lock someone else is actively using can lead to concurrent writers and
  state corruption.

## Exam Quick Facts

- Locking is **automatic**, not something you trigger manually — you only
  ever *disable* it (`-lock=false`) or *force clear* it (`force-unlock`).
- `-lock=false` skips locking for that single command invocation; it doesn't
  change any persistent configuration.
- `force-unlock` requires a **lock ID**, shown in the original failure
  output — you can't force-unlock without it.
- A backend lacking lock support simply performs no locking at all — no
  error, no warning during normal operation, it just isn't protected.
- Failed lock acquisition **stops** the run entirely; Terraform never
  silently proceeds against a locked state.
