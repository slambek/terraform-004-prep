# 4g. Validate configuration using custom conditions

## Four Ways to Validate, and When Each Runs

| Mechanism | Runs when | Blocks the operation? |
| :--- | :--- | :--- |
| **Input variable `validation`** | Immediately, before Terraform generates a plan. | Yes — errors and stops. |
| **`precondition`** | After a plan exists, but before Terraform creates/reads the enclosing resource/data source/output. | Yes. |
| **`postcondition`** | After planning **and** applying changes to the resource (or reading the data source). | Yes — halts further downstream actions, but does **not** undo what already happened. |
| **`check` block** | As the very last step of `plan` or `apply`, after everything else. | **No** — reports a warning and continues. |

All four require an `error_message` (a string expression — literals, heredocs, `format()`, etc. all work).

## Input Variable Validation

```hcl
variable "image_id" {
  type = string
  validation {
    condition     = length(var.image_id) > 4 && substr(var.image_id, 0, 4) == "ami-"
    error_message = "The image_id value must be a valid AMI ID, starting with \"ami-\"."
  }
}
```

Runs as Terraform builds the plan — catches misconfiguration before the provider API ever sees it, with a clearer message than a generic provider error.

## Preconditions and Postconditions

Both live inside a `lifecycle` block on a `resource`, `data`, or `output` block.

* **Precondition** — verify an *assumption* before Terraform acts. Preconditions take precedence over provider argument errors — they run first.

```hcl
resource "aws_instance" "example" {
  ami = data.aws_ami.example.id
  lifecycle {
    precondition {
      condition     = data.aws_ami.example.architecture == "x86_64"
      error_message = "The selected AMI must be for the x86_64 architecture."
    }
  }
}
```

* **Postcondition** — verify a *guarantee* after the resource is created or the data source read. Inside a postcondition, `self` refers to the enclosing block's own result:

```hcl
data "aws_ami" "example" {
  id = var.aws_ami_id
  lifecycle {
    postcondition {
      condition     = self.tags["Component"] == "nomad-server"
      error_message = "tags[\"Component\"] must be \"nomad-server\"."
    }
  }
}
```

`output` blocks support `precondition` only (no `self`, since an output has no separate identity to check afterward):

```hcl
output "instance_public_ip" {
  value = aws_instance.web.public_ip
  precondition {
    condition     = length([for r in aws_security_group.web.ingress : r if r.to_port == 80]) > 0
    error_message = "Security group must allow HTTP ingress traffic."
  }
}
```

**Choosing between them:** use a precondition for an assumption you want checked *before* creating the target block (helps future maintainers understand expected inputs); use a postcondition for a guarantee you need *after* creation (helps maintainers understand what must be preserved). If a resource has many dependencies, one postcondition on the resource itself is often more useful than duplicating the same precondition on every dependency.

## Check Blocks

Decoupled from any single resource's lifecycle — validates behavior of your infrastructure as a whole, without blocking the run:

```hcl
check "health_check" {
  data "http" "terraform_io" {
    url = "https://www.terraform.io"
  }
  assert {
    condition     = data.http.terraform_io.status_code == 200
    error_message = "${data.http.terraform_io.url} returned an unhealthy status code"
  }
}
```

* Runs last, after plan/apply completes.
* A failed `assert` produces a **warning only** — Terraform still finishes the operation.
* A `data` block declared inside a `check` is scoped to that check — you **cannot** reference it elsewhere in your configuration.
* Terraform records check results (`pass`/`fail`) in the **state file** under `check_results`.
* In HCP Terraform, enabling **health assessments** re-runs check blocks (plus preconditions/postconditions) periodically as **continuous validation**, independent of any plan/apply — useful for catching drift or external failures (e.g. a certificate expiring) between runs.

## Order of Evaluation (Exam-Relevant Sequence)

```text
1. Input variable validations   — before plan generation
2. Preconditions                — after plan, before create/read
3. Postconditions               — after apply (or data read)
4. Check blocks                 — last step of plan or apply
```

The *precise* timing of preconditions/postconditions can shift based on whether the referenced value is known at plan time or only after apply — if a condition depends on a value only known post-apply (e.g. an AWS-assigned volume ID), Terraform defers checking it until then.

## Exam Quick Facts

* Only `check` blocks are non-blocking (warning + continue); variable validation, preconditions, and postconditions all **halt** the operation on failure.
* Precondition = verify **before** acting; postcondition = verify **after** acting; `self` is only valid inside postconditions (and preconditions) on the enclosing resource/data block, not in variable validation.
* `output` blocks support `precondition` only — never `postcondition`.
* A `data` block nested inside a `check` block cannot be referenced anywhere outside that check.
* Evaluation order: variable validation → preconditions → postconditions → checks (last, non-blocking).
* Every validation mechanism requires an `error_message`.
