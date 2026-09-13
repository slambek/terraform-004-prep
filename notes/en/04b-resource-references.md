# 4b. Refer to resource attributes and create cross-resource references

## Basic Reference Syntax

```text
<RESOURCE_TYPE>.<NAME>            → a managed resource
data.<TYPE>.<NAME>                → a data resource
<RESOURCE_TYPE>.<NAME>.<ATTR>     → one exported/configured attribute
```

Any name that doesn't match one of Terraform's other named-value patterns (`var.`, `local.`, `module.`, `data.`, `path.`, `terraform.`) is interpreted as a **resource reference**.

Referencing an attribute of another resource — anywhere in an expression — is exactly what creates an **implicit dependency** (see 4f) between the two.

## Shape of a Resource Reference Depends on `count`/`for_each`

| Resource has… | Reference value is… |
| :--- | :--- |
| Neither `count` nor `for_each` | A single **object** — access attributes with dot or bracket notation. |
| `count` | A **list of objects** — one per instance. |
| `for_each` | A **map of objects** — keyed by the `for_each` key. |

## Nested Blocks and Attribute Access

Given:

```hcl
resource "aws_instance" "example" {
  ami = "ami-abc123"
  ebs_block_device {
    device_name = "sda2"
    volume_size = 16
  }
  ebs_block_device {
    device_name = "sda3"
    volume_size = 20
  }
}
```

* Top-level argument: `aws_instance.example.ami`
* Exported attribute: `aws_instance.example.id`
* All values across repeated nested blocks: a **splat expression** — `aws_instance.example.ebs_block_device[*].device_name` returns a list of every `device_name`.
* Nested blocks identified by a logical key (a block type that takes a label) are accessed by index: `aws_instance.example.device["foo"].size`; to get a map across all keyed blocks, use a `for` expression: `{for k, device in aws_instance.example.device : k => device.size}`.

## Referencing Multi-Instance Resources

**`count`** (result is a list):

```hcl
aws_instance.example[*].id   # list of every instance's id
aws_instance.example[0].id   # just the first instance's id
```

**`for_each`** (result is a map — indexed by key, not position):

```hcl
aws_instance.example["a"].id            # id of the "a"-keyed instance
[for value in aws_instance.example : value.id]   # list of all ids
```

⚠️ Splat expressions (`[*]`) only work on **lists** — they do **not** apply directly to a `for_each` resource (which is a map). To splat a `for_each` resource, convert it to a list first: `values(aws_instance.example)[*].id`.

## Resource Address Reference (Full Syntax)

A **resource address** identifies zero or more resource instances anywhere in the configuration tree:

```text
[module path][resource spec]
```

**Module path** — `module.<module_name>[<module index>]`. Omitting the module path means the **root module**. Multiple `module.` segments indicate nesting: `module.foo[0].module.bar["a"]`. An address with no resource spec (`module.foo`) addresses every resource within that module (instance).

**Resource spec** — `<resource_type>.<resource_name>[<instance index>]`. Without a module path prefix, this matches only resources in the **root module** (Terraform 0.12+; earlier versions matched any descendant module — no longer the exam-relevant behavior).

**Index values:**

* `[N]` — 0-based numeric index into a `count`-based resource. Omitting the index when `count > 1` means *all* instances.
* `["INDEX"]` — string key into a `for_each`-based resource.

```hcl
resource "aws_instance" "web" {
  count = 4
}
```

* `aws_instance.web[3]` → the last instance only.
* `aws_instance.web` → all four instances.

## Values Not Yet Known (`(known after apply)`)

Some resource attribute values (e.g. a generated unique ID) can't be predicted until the remote system actually creates the object. Terraform represents these as **unknown value placeholders** during plan — shown in plan output as `(known after apply)`.

Notable effects of unknown values:

* `count` **cannot** be unknown — Terraform must know the instance count during planning.
* If a `data` block's arguments include an unknown value, that data source's read is **deferred to apply** (see 4a), and its results are also unknown until then.
* An unknown value assigned into a `module` block's input, or into an `output` block's `value`, propagates the "unknown-ness" to every reference of that variable/output.

## Sensitive Resource Attributes

A provider can mark specific resource attributes as **sensitive** in its schema. Terraform then:

* Shows `(sensitive value)` instead of the real value in plan/apply output.
* Treats any value **derived from** that attribute as sensitive too (Terraform v0.15+ — earlier versions only obscured the direct attribute, not derived values).
* **Requires** you to explicitly mark an `output` as `sensitive = true` if it exposes a sensitive resource attribute — otherwise Terraform errors.
* Still **records the sensitive value in cleartext in state** — sensitivity is a display/UI protection, not an encryption mechanism (see 4h for handling this properly).

## Exam Quick Facts

* No `count`/`for_each` → reference is an **object**. `count` → **list**. `for_each` → **map**.
* Splat (`[*]`) works on lists (including `count` resources and repeated nested blocks) — **not** directly on `for_each` maps (use `values(...)[*]` instead).
* A resource address is `[module path][resource spec]`; omitting the module path means the root module.
* `count` can never itself be an unknown value.
* Sensitive resource attributes force sensitive outputs, but are still stored in cleartext in state.
