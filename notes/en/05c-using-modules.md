# 5c. Use modules in configuration

## The `module` Block's Meta-Arguments

Beyond `source` (5a) and `version` (5d), every `module` block supports these Terraform-native meta-arguments:

| Meta-argument | Purpose |
| :--- | :--- |
| `count` | Provision N nearly-identical instances of the module. Mutually exclusive with `for_each`. |
| `for_each` | Provision one instance per entry in a map or set of strings — best when instances need **varying** configuration keyed by something meaningful, rather than plain identical copies. |
| `providers` | Map an alternate/aliased provider configuration into the child module (see 5b). |
| `depends_on` | Force the module to wait on an upstream resource that isn't otherwise referenced in its arguments (see 4f for the general mechanics). |

## Referencing Module Outputs

```hcl
module.<LABEL>.<OUTPUT_NAME>
```

Only values the module explicitly exposes via `output` blocks are reachable this way (5b).

## Multiple Module Instances

**`count`** — good for identical/near-identical copies, indexed numerically:

```hcl
locals {
  instance_names = ["example-instance-1", "example-instance-2", "example-instance-3"]
}

module "ec2_instance" {
  source = "terraform-aws-modules/ec2-instance/aws"
  count  = length(local.instance_names)
  name   = local.instance_names[count.index]
}
```

**`for_each`** — good when each instance needs distinct configuration keyed by a map or set:

```hcl
module "ec2_instance" {
  source   = "terraform-aws-modules/ec2-instance/aws"
  for_each = var.instance_configs   # a map keyed by name
  name     = each.key
  ami      = each.value.ami
}
```

## Explicit Module Dependencies

```hcl
resource "aws_s3_bucket" "example" {
  bucket = "my-example-bucket-12345"
}

module "ec2_instance" {
  source     = "terraform-aws-modules/ec2-instance/aws"
  depends_on = [aws_s3_bucket.example]
}
```

Same rules as any other `depends_on` usage (4f): prefer implicit dependencies (referencing an attribute) whenever possible; reach for `depends_on` only when a real dependency exists that isn't visible through any argument reference.

## Passing Alternate Provider Configurations

Same `providers` meta-argument and `aws.usw1` alias setup as 5b's example —
shown here extended with a second alias to illustrate the general shape:

```hcl
module "tunnel" {
  source = "./tunnel"
  providers = {
    aws.src = aws.usw1   # aws.usw1 / aws.usw2 declared as in 5b
    aws.dst = aws.usw2
  }
}
```

(Full mechanics — including the child module's `configuration_aliases` requirement — are in 5b/2b.)

## Recommended Module File Structure

None of these files are special to Terraform — a module can be a single `.tf` file — but this layout is the community convention:

```text
.
├── LICENSE
├── README.md
├── main.tf
├── variables.tf
└── outputs.tf
```

* `main.tf` — the module's primary resources.
* `variables.tf` — its input interface.
* `outputs.tf` — its exported values.
* `README.md` / `LICENSE` — documentation and licensing; Terraform itself ignores these, but registries (public or private) and code hosts display them.

**Never distribute these with a module** (and `.gitignore` them):

* `terraform.tfstate` / `terraform.tfstate.backup` — state is specific to a particular deployment, not the module's code.
* `.terraform/` — the local plugin/module cache (see 3b).
* `*.tfvars` — module inputs come from the caller's `module` block arguments, not `.tfvars`, and these files often contain secrets.

## Module Composition: Keep the Tree Flat

Once you start calling modules, configuration becomes hierarchical rather than flat — but the strong recommendation is to keep the module tree to **one level of child modules** and use ordinary expressions to wire modules together, rather than nesting deeply:

```hcl
module "network" {
  source          = "./modules/aws-network"
  base_cidr_block = "10.0.0.0/8"
}

module "consul_cluster" {
  source     = "./modules/aws-consul-cluster"
  vpc_id     = module.network.vpc_id
  subnet_ids = module.network.subnet_ids
}
```

This "module composition" style assembles small, single-purpose modules into a larger system from the **root** module, rather than having one module embed and manage another module's resources internally — keeping each piece independently understandable and reusable in different combinations.

## Exam Quick Facts

* `count` and `for_each` on a `module` block are mutually exclusive — same rule as on `resource`/`data` blocks.
* `providers` + a `depends_on` list are both **meta-arguments**, not module-specific inputs — every module block supports them regardless of what the module itself declares.
* Never ship `terraform.tfstate`, `.terraform/`, or `*.tfvars` alongside module source code.
* Community convention: `main.tf` / `variables.tf` / `outputs.tf` / `README.md` / `LICENSE` — none of it required by Terraform itself.
* Prefer a **flat** module tree (one level of children) wired together with expressions in the root module, over deep nesting.
