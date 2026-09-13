# 4c. Use variables and outputs

## Why: The Module Interface

Variables, locals, and outputs are how modules communicate:

* **Variables** — the module's input interface (like function arguments).
* **Locals** — temporary, module-scoped named expressions (avoid retyping the same expression).
* **Outputs** — the module's return values / exported data.

## Variable Block

```hcl
variable "instance_type" {
  type        = string
  description = "EC2 instance type for the web server"
  default     = "t2.micro"
}
```

Key arguments (all optional except the label itself):

| Argument | Purpose |
| :--- | :--- |
| `type` | Type constraint (see 4d). No `type` → accepts any type. |
| `default` | Makes the variable optional. Must be a **literal value** — cannot reference other objects in configuration. Without a default, the caller **must** supply a value (Terraform never leaves a variable unassigned). |
| `description` | Documents purpose — write from the **consumer's** point of view. |
| `validation` | One or more `condition` + `error_message` blocks — evaluated **before generating a plan** (see 4g). |
| `sensitive` | Redacts the value from CLI output; any expression referencing it also becomes sensitive. Still stored in cleartext in state (see 4h). |
| `nullable` | Default `true` — allows explicitly passing `null`. Set `false` to require a non-null value. |
| `ephemeral` | Terraform 1.10+. Value is available at runtime but **omitted from state and plan files** entirely (see 4h). |

Reserved variable names you cannot use: `source`, `version`, `providers`, `count`, `for_each`, `lifecycle`, `depends_on`, `locals`.

## Locals Block

```hcl
locals {
  app_name = "${var.project_name}-${var.environment}"
}
```

* Reference with `local.<name>`.
* A local can reference other locals (even within the same block) as long as there's no circular dependency.
* Scoped to the module — not an input, not exported automatically (must go through an output to leave the module).

## Output Block

```hcl
output "instance_ip_addr" {
  value       = aws_instance.server.private_ip
  description = "The private IP address of the main server instance."
}
```

Serves four purposes: exposing a child module's resource attributes to its parent, displaying values in root-module CLI output, letting other configs read root outputs via `terraform_remote_state`, and passing data to automation tooling.

| Argument | Purpose |
| :--- | :--- |
| `value` | **Required.** Any valid expression; the result is stored in state. |
| `description` | Documents the output, again from the consumer's perspective. |
| `sensitive` | Redacts the value in CLI output. **Required** if the value derives from a sensitive resource attribute or sensitive variable. Still recorded in state in cleartext (`-json`/`-raw` on `terraform output` will reveal it in plain text regardless). |
| `ephemeral` | Terraform 1.10+, **child modules only** — cannot be set on root-module outputs. Omits the value from state/plan files (see 4h). |
| `depends_on` | Explicit dependency (rare — add a comment explaining why when you use it; see 4f). |
| `precondition` | Validates the output's value before exposing/storing it (see 4g). |

## Assigning Variable Values (Order of Precedence)

Terraform uses the **last value it finds**, roughly in this order (later overrides earlier):

1. Environment variables — `TF_VAR_<name>`.
2. `terraform.tfvars` (auto-loaded if present).
3. `*.auto.tfvars` files (auto-loaded, alphabetical).
4. `-var-file=FILENAME` on the command line (repeatable).
5. `-var 'NAME=VALUE'` on the command line (repeatable, highest precedence among these).

If Terraform Community Edition still has no value and no default, it **prompts interactively** — unless `-input=false`, in which case it errors.

`.tfvars` files use HCL-like syntax (or JSON) but **cannot contain resource definitions or other configuration constructs** — only variable value assignments.

## Querying Outputs

```bash
terraform output                 # all outputs, human-readable (redacts sensitive)
terraform output <name>          # a single output BY NAME — sensitive values are NOT redacted here
terraform output -raw <name>     # unquoted string, for piping into other commands
terraform output -json           # machine-readable — sensitive values NOT redacted here either
```

Terraform only redacts sensitive outputs during `plan`/`apply`/`destroy` operations and when querying **all** outputs together — querying a specific output by name, or using `-json`, always reveals the real value.

## Exam Quick Facts

* A variable with no `default` **requires** a caller-supplied value — Terraform never runs with an unassigned variable.
* `default` must be a literal — no references to other resources/variables allowed.
* Assignment precedence (low→high): env vars → `terraform.tfvars` → `*.auto.tfvars` → `-var-file` → `-var`.
* `sensitive` on outputs is **mandatory** when the value comes from a sensitive resource attribute or sensitive variable.
* `terraform output <name>` and `terraform output -json` do **not** redact sensitive values — only the "all outputs" human view does.
* `ephemeral` on outputs is child-module-only — never valid on a root module output.
