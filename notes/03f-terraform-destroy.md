# 3f. Destroy Terraform-managed infrastructure

## `terraform destroy` = a Convenience Alias

```bash
terraform destroy
```

is exactly equivalent to:

```bash
terraform apply -destroy
```

It accepts most of the same options as `apply`, except it does **not** take a plan-file argument, and it forces the "destroy" planning mode (see 3d). Like a normal `apply`, it prompts for confirmation (`yes`) before proceeding, since destruction is irreversible.

## Previewing a Destroy Without Executing It

```bash
terraform plan -destroy
```

Generates a **speculative destroy plan** — shows exactly what would be removed, without doing it. Useful before committing to a destroy, especially in shared environments.

## Destroying Specific Resources

```bash
terraform destroy -target aws_instance.example
```

`-target` scopes destruction to a specific resource (and anything depending on it) rather than the entire workspace. Same exceptional-use caveat as with `plan`/`apply` (3d/3e) — prefer splitting configurations over routine `-target` use.

## Removing a Resource Without Destroying Everything

You don't need `terraform destroy` to remove just *one* resource. The normal workflow:

1. **Comment out (or delete) the resource block** in configuration (and anything that references it, e.g. an output value pointing at its attributes — that must also be removed/commented, or `validate` will fail).
2. Run `terraform apply`.
3. Terraform detects the resource is no longer in configuration and proposes to **destroy** just that resource — leaving everything else untouched.

This is the standard way to shrink a workspace incrementally, as opposed to tearing down everything with `destroy`.

## When to Use Full `destroy`

* Ephemeral / short-lived environments (dev sandboxes, CI test infra) once the task is done.
* Retiring an entire workspace or application environment from service.

Not typically used for long-lived production infrastructure.

## Exam Quick Facts

* `terraform destroy` is literally `terraform apply -destroy` under the hood.
* `terraform plan -destroy` previews a destroy without executing it (speculative destroy plan).
* Removing one resource from a live workspace = **delete/comment its block + `terraform apply`** (Terraform proposes to destroy just that resource) — you do **not** need `terraform destroy` for that.
* `-target` scopes a destroy to specific resources — exceptional use, not routine.
