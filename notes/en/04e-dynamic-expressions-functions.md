# 4e. Write dynamic configuration using expressions and functions

## Named Values Reference Table

| Syntax | Refers to |
| :--- | :--- |
| `<RESOURCE_TYPE>.<NAME>` | A managed resource (see 4b for full behavior). |
| `var.<NAME>` | An input variable — auto-converted to match its `type` constraint. |
| `local.<NAME>` | A local value. |
| `module.<MODULE_NAME>` | A child module's outputs — object (no `count`/`for_each`), map (`for_each`), or list (`count`) of objects, mirroring resource reference shape (4b). |
| `data.<TYPE>.<NAME>` | A data resource (see 4a). |
| `path.module` | Filesystem path of the module containing the expression. Avoid in write operations — local module invocations share source dirs, risking race conditions. |
| `path.root` | Filesystem path of the configuration's root module. |
| `path.cwd` | Original working directory before any `-chdir`. Prefer `path.root`/`path.module` where possible. |
| `terraform.workspace` | Name of the currently selected workspace. |

⚠️ These are **not real objects** — you can't iterate over `aws_instance` itself in a `for` expression, and you can't substitute square-bracket notation for the dot-separated path.

**Block-local values** (only valid inside specific block contexts): `count.index` (in `count` resources), `each.key` / `each.value` (in `for_each` resources), `self` (in `provisioner`/`connection` blocks, and in `precondition`/`postcondition` referring to the enclosing resource — see 4g).

**Portability warning:** `path.cwd` and `terraform.workspace` embed context about *where*/*how* Terraform is being run. Using them directly in a resource argument (e.g. baking `terraform.workspace` into a globally-unique name) can break if the module is called more than once, or run from a different machine/directory. Prefer accepting a `prefix` input variable and letting the *caller* decide whether to derive it from the workspace.

## Conditional Expressions

```text
condition ? true_value : false_value
```

```hcl
locals {
  name = (var.name != "" ? var.name : random_id.id.hex)
}
```

Common uses: computing a fallback value, toggling `count` between 0/1/N based on a boolean flag, or conditionally leaving an optional attribute unset (`null`, see 4d).

```hcl
resource "aws_instance" "ubuntu" {
  count                       = var.high_availability ? 3 : 1
  associate_public_ip_address = count.index == 0 ? true : false
}
```

## Splat Expressions

`[*]` iterates over a list, pulling one attribute from every element — see 4b for full mechanics on resources/nested blocks. General form: `list_expr[*].attribute`.

## `for` Expressions

Transform a collection into another list, map, or object:

```hcl
[for instance in aws_instance.web_app : instance.id]                 # list
{for k, device in aws_instance.example.device : k => device.size}    # map/object
```

Needed whenever a `for_each`-based resource's attributes must be flattened into a list output (splat alone won't work on a map — see 4b), or when reshaping one collection's keys/values into another.

## Built-In Functions

```text
function_name(arg1, arg2, ...)
```

* You **cannot** author your own functions in the configuration language — only providers can expose custom **provider-defined functions**, called as `provider::<local_name>::<function>(...)`.
* Experiment interactively with `terraform console`.

### Frequently Used Functions

| Function | Purpose |
| :--- | :--- |
| `templatefile(path, vars_map)` | Renders a template file, interpolating `${key}` placeholders from the given map — e.g. generating an EC2 user-data script dynamically. |
| `lookup(map, key, default)` | Retrieves a value from a map by key, with an optional fallback if the key is missing. |
| `file(path)` | Reads a file's raw contents as-is — **no interpolation**; only use for files that don't need per-run modification (e.g. an SSH public key). |
| `slice(list, start, end)` | Returns a sub-list — `end` is exclusive. |
| `merge(map1, map2, ...)` | Combines maps, later arguments' keys win on conflict. |
| `regexall(pattern, string)` | Returns all regex matches — useful inside `validation` blocks (4g) to enforce character-set rules. |
| `jsonencode(value)` | Serializes any value to a JSON string — the standard escape hatch for `any`-typed values (4d). |
| `length(collection)` | Element count — usable in outputs, conditions, and `for_each` sizing checks. |

### `templatefile` Example

```hcl
resource "aws_instance" "web" {
  user_data = templatefile("user_data.tftpl", {
    department = var.user_department
    name       = var.user_name
  })
}
```

The `.tftpl` file references `${department}` and `${name}` — Terraform interpolates them at plan/apply time, making the script reusable across different variable values.

## Exam Quick Facts

* Named values are **not real objects** — no square-bracket substitution, no iterating over a resource type itself.
* `path.cwd` and `terraform.workspace` risk breaking module portability if baked directly into resource arguments — prefer an input variable.
* Splat (`[*]`) works on **lists**; use a `for` expression to reshape a `for_each` **map** into a list or another map.
* You cannot define custom functions in HCL — only providers can add functions (`provider::name::function()`).
* `file()` never interpolates; `templatefile()` does.
* `terraform console` is the tool for experimenting with function behavior before using it in configuration.
