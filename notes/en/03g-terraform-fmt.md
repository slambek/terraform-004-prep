# 3g. Apply formatting and style adjustments to a configuration

## What `terraform fmt` Does

`terraform fmt` rewrites configuration files in the target directory to match Terraform's **canonical style** — consistent indentation, alignment, spacing. It's intentionally **opinionated with no customization options**, because its whole purpose is consistency across codebases, not personal preference.

Any configuration Terraform itself generates already follows this style — running `fmt` on hand-written files keeps everything consistent.

## What It Catches vs. What It Doesn't

`fmt` only parses HCL for **formatting issues** — things like invalid characters or malformed block syntax that prevent it from even understanding the file structure (e.g. a stray `$` where an interpolation was intended: `Name = $var.name-learn`). It reports these as errors because it can't safely reformat code it can't parse.

**It does not check correctness against providers** — wrong attribute names, missing required arguments, type mismatches, cycles. That's the job of `terraform validate` (3c). The recommended sequence is:

```bash
terraform fmt        # fix style / catch unparseable syntax
terraform validate   # check correctness against provider schemas
```

## Usage

```bash
terraform fmt [options] [target...]
```

By default it scans the **current directory**. A target can instead be a specific directory, a specific file, or stdin (`-`).

| Flag | Effect |
| :--- | :--- |
| `-list=false` | Don't list filenames with formatting inconsistencies. |
| `-diff` | Show the diff of formatting changes. |
| `-write=false` | Don't overwrite files (implied automatically by `-check` or when input is stdin). |
| `-check` | Exit `0` if already formatted, non-zero otherwise (and lists offending files) — doesn't rewrite anything. Useful as a CI gate. |
| `-recursive` | Also process subdirectories (off by default — only the target directory is processed). |
| `-no-color` | Disable colored output. |

## Exam Quick Facts

* `fmt` = **style/formatting only** — canonical spacing/indentation, no provider-aware correctness checking.
* No customization options — it's deliberately opinionated for cross-codebase consistency.
* Recommended order: `fmt` → `validate` → `plan` → `apply`.
* `-check` is the non-destructive CI-friendly variant (reports, doesn't rewrite).
* `-recursive` is required to touch subdirectories — default scope is the current directory only.
