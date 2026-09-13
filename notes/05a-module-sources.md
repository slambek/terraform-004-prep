# 5a. Explain how Terraform sources modules

## The `source` Argument

Every `module` block requires a `source` argument telling Terraform **where** to get the module's code. It must be a **literal string** — no expressions or interpolation allowed. Whenever you change `source`, you must re-run `terraform init` (or `terraform get`) so Terraform re-fetches the code.

```hcl
module "consul" {
  source  = "hashicorp/consul/aws"
  version = "0.1.0"
}
```

## Source Types

| Source type | Syntax | Notes |
| :--- | :--- | :--- |
| **Local path** | `./PATH` or `../PATH` | Files already on disk. Terraform treats this as a direct reference (no copy) for relative paths. Absolute paths (`/...` or a drive letter) get copied into the local module cache — **avoid absolute paths**, they couple config to one machine's filesystem layout. |
| **Public Terraform Registry** | `<NAMESPACE>/<NAME>/<PROVIDER>` | e.g. `hashicorp/consul/aws`. This is the **primary way to share modules** across configs/teams. |
| **Private registry (HCP Terraform/Enterprise)** | `<HOSTNAME>/<NAMESPACE>/<NAME>/<PROVIDER>` | Same shape as public, with a hostname prefix, e.g. `app.terraform.io/example_corp/vpc/aws`. `localterraform.com` is a generic hostname that always resolves to whichever platform instance is running the config. |
| **GitHub** | `github.com/ORG/REPO` (HTTPS) or `git::github.com/ORG/REPO` (SSH) | Runs `git clone` under the hood — uses your local Git credential configuration. |
| **Generic Git repo** | `git::ssh://...` or `git::https://...` | Any Git URL. Supports `?ref=BRANCH_OR_TAG_OR_SHA` and `?depth=N` (shallow clone) query params. |
| **Bitbucket** | `bitbucket.org/PATH` | Git-hosted — same clone mechanics/credentials as generic Git. |
| **Mercurial repo** | `hg::PROTOCOL://...` | Runs `hg clone`; supports `#revision` fragment. |
| **HTTP/HTTPS URL** | `https://...` | Terraform sends a GET (`?terraform-get=1`) and expects either an `X-Terraform-Get` response header or an HTML `<meta name="terraform-get" ...>` tag pointing at the real source — a "vanity URL" indirection. If the URL itself ends in a recognized archive extension (`.zip`, `.tar.gz`, etc.), Terraform skips the redirect and treats the URL as the archive directly. |
| **S3 bucket object** | `s3::https://BUCKET-URL/module.zip` | Object must be an archive; credentials resolved via the standard AWS SDK credential chain (env vars → shared credentials file → shared config → instance profile), or via inline query params (`aws_profile`, etc. — never commit these). |
| **GCS bucket object** | `gcs::https://www.googleapis.com/storage/v1/BUCKET/PATH` | Authenticates via Google Cloud SDK conventions (`GOOGLE_OAUTH_ACCESS_TOKEN`, `GOOGLE_APPLICATION_CREDENTIALS`, GCE default credentials, or `gcloud auth application-default login`). |

## Subdirectories Within a Source Package

Add `//` after the package root to point at a subdirectory within it — place any query parameters (like `ref=`) **after** the subdirectory segment:

```hcl
module "consul" {
  source = "hashicorp/consul/aws//modules/consul-cluster"
}

module "vpc" {
  source = "git::https://example.com/network.git//modules/vpc?ref=v1.2.0"
}
```

Terraform downloads/extracts the **entire package** to local disk but only reads the module from the specified subdirectory — which means modules inside the same package's subdirectories can reference each other with ordinary local paths.

## Duplicate Sources, Unique Labels

You can use the **same** `source` in two or more separate `module` blocks — you just need a **unique label** for each block. This lets you provision multiple, differently-configured copies of the same module without `count`/`for_each` (see 5c for those).

## Exam Quick Facts

* `source` must be a literal string — never an expression or variable reference.
* Changing `source` (or `version` — see 5d) always requires re-running `terraform init`.
* The Terraform Registry format is `<NAMESPACE>/<NAME>/<PROVIDER>`; a private registry just prepends a hostname.
* `//` marks the start of a subdirectory path within a downloaded package; query params (like `ref=`) go **after** it.
* Absolute local filesystem paths are discouraged — they tie configuration to one machine.
* Reusing the same `source` across multiple `module` blocks is fine as long as each block's label is unique.
