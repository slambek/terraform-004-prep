# 4f. Define resource dependencies in configuration

> **Scope note:** the internal graph mechanics (node types, build/walk order, `-parallelism`) are covered in **3d**. This note focuses on the two ways *you* declare a dependency in configuration, and their practical effects.

## Implicit Dependencies (the Default, Preferred Way)

Terraform automatically infers a dependency whenever one resource's configuration **references another resource's attribute**:

```hcl
resource "aws_instance" "example_a" { }

resource "aws_eip" "ip" {
  instance = aws_instance.example_a.id   # implicit dependency
}
```

Because `aws_eip.ip` references `aws_instance.example_a.id`, Terraform knows it must fully create `example_a` before creating `ip`. A resource with **no** such reference (e.g. `aws_instance.example_b` alongside the above) has no ordering constraint and can be created **in parallel**.

This is the primary and preferred way Terraform understands relationships — it requires no extra syntax, just writing the reference naturally.

## Explicit Dependencies (`depends_on`)

Sometimes a real dependency exists but isn't visible through any attribute reference — e.g. an application on an EC2 instance that expects a specific S3 bucket to already exist, where nothing in the Terraform config for the instance actually references the bucket. Use `depends_on` on the resource or module block to force the ordering:

```hcl
resource "aws_s3_bucket" "example" {}

resource "aws_instance" "example_c" {
  depends_on = [aws_s3_bucket.example]
}

module "example_sqs_queue" {
  source     = "terraform-aws-modules/sqs/aws"
  depends_on = [aws_s3_bucket.example, aws_instance.example_c]
}
```

`depends_on` accepts a **list**, so a single resource/module can wait on several others.

`depends_on` is also supported on `data` blocks (see 4a) and `output` blocks (see 4c) — in both cases it forces evaluation to wait on the listed dependency, and on a data block it defers the read as covered in 4a.

⚠️ **Cost:** since Terraform waits for the listed resource to fully finish before starting the dependent one, adding explicit dependencies can **increase total apply time** by removing parallelism that would otherwise be possible.

## Dependencies Affect Both Create *and* Destroy Order

The same graph edges used for creation are used — in reverse — for destruction. Given the S3-bucket example above: on `terraform destroy`, the SQS queue and the EC2 instance (both of which depend on the bucket) are destroyed **before** the bucket itself, even though the bucket was created first.

## What Does *Not* Affect Order

**The order resources are declared in the configuration file has no effect** on the order Terraform creates or destroys them — only the dependency graph (implicit references + `depends_on`) determines execution order.

## Exam Quick Facts

* Implicit dependency = referencing another resource's attribute in an expression — the default, preferred mechanism.
* Explicit dependency = `depends_on = [...]`, used only when a real dependency exists but isn't visible via any attribute reference.
* `depends_on` works on `resource`, `module`, `data`, and `output` blocks.
* Explicit dependencies **cost apply time** — they remove otherwise-available parallelism.
* Dependency order governs **both** create and destroy — destroy runs in reverse.
* Declaration order in the `.tf` file is irrelevant to execution order.
