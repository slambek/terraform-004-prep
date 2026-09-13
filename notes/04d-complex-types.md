# 4d. Understand and use complex types

## Type Constraints — Where They're Used

Type constraints look like expressions but are special syntax valid **only** in a variable block's `type` argument.

## Primitive Types

| Type | Meaning |
| :--- | :--- |
| `string` | Sequence of Unicode characters. |
| `number` | Whole or fractional numeric value. |
| `bool` | `true` / `false`. |

Terraform auto-converts between these where the value makes sense: `true` ↔ `"true"`, `15` ↔ `"15"`, etc.

## Complex Types: Collection vs. Structural

**Collection types** — group multiple values of **one** element type:

| Type | Description |
| :--- | :--- |
| `list(TYPE)` | Ordered sequence, indexed from 0. |
| `map(TYPE)` | Lookup table, string-keyed. |
| `set(TYPE)` | Unordered collection of unique values. |

`list` and `map` alone (no type argument) are shorthand for `list(any)` / `map(any)` — legacy compatibility syntax; prefer the explicit form in new code.

**Structural types** — group values of **different** types under a fixed schema:

| Type | Schema syntax | Description |
| :--- | :--- | :--- |
| `object({KEY=TYPE, ...})` | `{ name = string, age = number }` | Named attributes, each its own type. A value with *extra* keys still matches — the extras are discarded during conversion. |
| `tuple([TYPE, ...])` | `[string, number, bool]` | Fixed-length sequence, exact element count required, each position independently typed. |

## Conversion Rules Between Complex Types

Terraform automatically converts between "similar kinds" whenever possible:

* **Objects ↔ maps** — a map (or larger object) converts to an object if it has *at least* the required keys (extra attributes are dropped — lossy).
* **Tuples ↔ lists** — a list converts to a tuple only if it has *exactly* the required element count.
* **Sets ↔ lists/tuples** — converting a list/tuple to a set drops duplicates and loses order; converting a set back to a list/tuple gives elements in an arbitrary order (lexicographical for strings).
* Terraform also recursively converts each **element's** type, using the primitive conversion rules above where applicable.

If no valid conversion path exists (e.g. converting a tuple containing a nested list into a `map(string)`), Terraform raises a **type mismatch error**.

## The `any` Constraint

`any` is a **placeholder**, not a real type — Terraform tries to find one concrete type that satisfies every use of the value.

⚠️ **Use `any` only when you pass the value through untouched** (e.g. straight into `jsonencode()`) without ever accessing its internal structure. If your module inspects, indexes, or type-checks the value in any way, write the exact type instead.

With a collection like `list(any)`, Terraform infers a single element type for the *whole* list — e.g. `["a", "b", "c"]` → `list(string)`; mixed `["a", 1, "b"]` still resolves to `list(string)` via primitive conversion; but `["a", [], "b"]` fails because a string and an empty tuple share no common type.

## Optional Object Attributes

Mark an `object` attribute optional with the `optional(...)` modifier inside the type schema:

```hcl
variable "with_optional_attribute" {
  type = object({
    a = string                # required
    b = optional(string)      # optional, defaults to null if omitted
    c = optional(number, 127) # optional, defaults to 127 if omitted
  })
}
```

* `optional(TYPE)` — no default → missing/`null` becomes `null`.
* `optional(TYPE, DEFAULT)` — Terraform substitutes `DEFAULT` both when the attribute is **omitted** and when it's **explicitly set to `null`** — so downstream code never has to null-check it.
* Defaults apply **top-down** through nested optional structures — the outer default is applied first, then any nested defaults within it.
* To dynamically decide *not* to set an optional attribute, use a conditional expression with `null` in one arm: `error_document = var.legacy_filenames ? "ERROR.HTM" : null` — this leaves the attribute unset so the module's own default takes over.

## Exam Quick Facts

* Primitive types: `string`, `number`, `bool` — auto-convert between each other when the string content is valid.
* Collection types (`list`, `map`, `set`) hold **one** element type; structural types (`object`, `tuple`) hold **mixed** types under a fixed schema.
* Object → map conversion (and vice versa) can be **lossy** — extra keys get dropped.
* `any` is a placeholder resolved to one concrete type — never use it just to avoid writing a real type constraint.
* `optional(TYPE, DEFAULT)` substitutes the default for both omitted **and** explicit `null` — guaranteeing a non-null value inside the module.
