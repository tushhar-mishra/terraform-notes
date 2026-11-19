# 30 · The AWS Provider and Authentication

> **Level:** ★★ intermediate
>
> **You will learn:** how to configure **`hashicorp/aws`** (v6): `region`, `profile`, the **credential chain** (where the provider looks for credentials, in order), **`assume_role`** for other accounts, **`allowed_account_ids`** as a safety catch, **`default_tags`** on every resource, **aliases** for several regions or accounts, and v6's per-resource **`region`** argument. Your first AWS Terraform code: account/region data sources and a **budget** resource (free).
>
> **You need:** topics 10 (providers, aliases), 14 (variables), 19 (data sources), 29 (AWS identities, profiles).
>
> **Part 4 · AWS** · Previous: [29 · AWS basics for Terraform](29-aws-basics.md) · Next: [31 · First AWS resources: S3 and EC2](31-first-aws-resources.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** The AWS provider needs a **region** and **credentials**. Put the region in code; let credentials come from **outside** the code (a profile, SSO, environment variables or a CI role), never written in `.tf` files.

```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.0" }
  }
}

provider "aws" {
  region              = "ap-south-1"
  allowed_account_ids = ["123456789012"]   # refuse to run against any other account

  default_tags {
    tags = {
      project    = "tf-course"
      managed_by = "terraform"
    }
  }
}
```

```bash
export AWS_PROFILE=tf-lab && aws sso login
terraform init && terraform plan
```

**Picture it.** The provider is a courier: the **region** is the city on the parcel label (written in the code); the **credentials** are the courier's ID badge, handed over at the door each day (from your profile), never printed on the parcel.

**The one thing to know.** No access keys in `.tf` files, ever. Use `AWS_PROFILE` locally and an assumed role (OIDC) in CI.

**Try it now:** apply the topic's lab code, which reads your account ID and creates a free USD 5 budget.

**You've got it if you can:** list the credential chain in order, and explain what `allowed_account_ids` and `default_tags` protect you from.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Provider arguments you'll actually use

| Argument | Purpose |
|---|---|
| `region` | Default region for resources. Required (or from `AWS_REGION`/profile). |
| `profile` | Named profile from `~/.aws/config`. Better left out of code and set via `AWS_PROFILE`, so CI and laptops can differ. |
| `allowed_account_ids` / `forbidden_account_ids` | Refuse to run unless the credentials belong to an allowed account. Cheap insurance against "applied dev code to prod". |
| `default_tags { tags = {...} }` | Tags added to **every** taggable resource from this provider. Resource-level `tags` merge on top. |
| `assume_role { role_arn = ... }` | After authenticating, assume this role and use its temporary credentials. |
| `assume_role_with_web_identity { ... }` | Assume a role using an OIDC token (how CI often works; topic 57). |
| `ignore_tags { keys = [...] }` | Ignore tags that other systems add (avoids endless diffs). |
| `access_key` / `secret_key` | ⚠️ Exist, but **never** put them in code. |

### The credential chain

The provider (like the CLI) tries these **in order** and uses the first that works:

| # | Source | Typical where |
|---|---|---|
| 1 | `access_key`/`secret_key`/`token` in the provider block | ⚠️ never |
| 2 | **Environment variables**: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN` | CI after an OIDC step writes them |
| 3 | **Shared config/credentials files** with a profile (`profile` argument or `AWS_PROFILE`, else `default`): SSO, keys, `credential_process`, `role_arn` + `source_profile` | Your laptop |
| 4 | **Web identity**: `AWS_WEB_IDENTITY_TOKEN_FILE` + `AWS_ROLE_ARN` | Kubernetes pods on EKS (IRSA = IAM Roles for Service Accounts), some CI |
| 5 | **Container credentials** (ECS task role) | Terraform running in ECS or CodeBuild (AWS's managed build service) |
| 6 | **Instance metadata (IMDS)**: the EC2 instance's role | Terraform running on an EC2 instance (e.g. a Jenkins agent, [Jenkins topic 61](../../Jenkins/notes/61-aws-and-k8s-primer.md)) |

Then, if `assume_role` is set, the provider uses those credentials to assume the role.

**Debugging tip:** if Terraform uses unexpected credentials, an old `AWS_ACCESS_KEY_ID` in your environment (step 2) is usually overriding your profile (step 3). `env | grep ^AWS_`.

### Multi-region and multi-account

Two tools:

**1. Provider aliases** (topic 10): a second configuration with its own region and/or role:

```hcl
provider "aws" {
  alias  = "virginia"
  region = "us-east-1"            # e.g. certificates for CloudFront (AWS's CDN) must live here
}

provider "aws" {
  alias  = "prod"
  region = "ap-south-1"
  assume_role {
    role_arn     = "arn:aws:iam::222222222222:role/terraform"
    session_name = "tf-course"
  }
}

resource "aws_acm_certificate" "cdn" {
  provider          = aws.virginia
  domain_name       = "www.example.com"
  validation_method = "DNS"
}
```

**2. Per-resource `region` (AWS provider 6.0+)**: same account, different region, without an extra provider block:

```hcl
resource "aws_s3_bucket" "replica" {
  region = "eu-west-1"            # overrides the provider's region for this resource only
  bucket = "acme-logs-replica"
}
```

Use aliases for **different accounts/roles**; the `region` argument for "same credentials, another region".

---

## Why it exists

- **Credentials outside code** means the same code runs on your laptop (SSO profile), in GitHub Actions (OIDC role) and in Jenkins (instance role) without changes, and secrets never reach Git.
- **`allowed_account_ids`** turns "oops, wrong profile" from an incident into an error message.
- **`default_tags`** gives consistent tagging for cost reports and ownership with zero repetition.
- **Aliases/`assume_role`** let one configuration manage resources across accounts, e.g. DNS in a shared account and the app in a workload account.

---

## Analogy

**A courier company.**

- **Region** = the city on the delivery label. Part of the order (code).
- **Credentials** = the courier's ID badge. Issued fresh each day at the depot (SSO, roles), never printed on the parcel (code).
- **Credential chain** = where the courier looks for the badge: in hand → in the jacket pocket (env vars) → in the locker (profile files) → from the depot's machine (instance/container role).
- **`assume_role`** = showing your badge at a partner company's reception to get **their** visitor badge for the day.
- **`allowed_account_ids`** = "this parcel may only be delivered to customer #123". Wrong address → refused.
- **`default_tags`** = the company logo stamped on every parcel automatically.

---

## How it works inside

```
terraform plan
 └─ ConfigureProvider(aws): region=ap-south-1, assume_role=?, default_tags=...
      1. walk the credential chain → base credentials
      2. assume_role set? → sts:AssumeRole → temporary credentials
      3. sts:GetCallerIdentity → account ID
         allowed_account_ids set and account not in it? → ERROR, stop
 └─ for each resource: sign API requests (SigV4) with those credentials
      tags sent = default_tags merged with resource tags
      (state records both: tags = resource tags, tags_all = merged result)
```

`tags` vs `tags_all`: on every taggable resource, `tags` is what **you** wrote; `tags_all` (computed) is `default_tags` + `tags`. Plans for tag changes often show `tags_all`.

---

## Code / commands

`~/tf-lab/30-aws/`:

`versions.tf`:

```hcl
terraform {
  required_version = ">= 1.14"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

`variables.tf`:

```hcl
variable "region" {
  type    = string
  default = "ap-south-1"
}

variable "account_id" {
  type        = string
  description = "The only AWS account this configuration may touch."
  validation {
    condition     = can(regex("^[0-9]{12}$", var.account_id))
    error_message = "account_id must be 12 digits."
  }
}

variable "alert_email" {
  type        = string
  description = "Where budget alerts are sent."
}
```

`providers.tf`:

```hcl
provider "aws" {
  region              = var.region
  allowed_account_ids = [var.account_id]

  default_tags {
    tags = {
      project    = "tf-course"
      managed_by = "terraform"
      owner      = "you"
    }
  }
}
```

`main.tf`:

```hcl
data "aws_caller_identity" "current" {}      # who am I?
data "aws_region" "current" {}               # which region is the provider using?
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_budgets_budget" "lab" {
  name         = "tf-course-monthly"
  budget_type  = "COST"
  limit_amount = "5"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"     # warn early, on the forecast
    subscriber_email_addresses = [var.alert_email]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = [var.alert_email]
  }
}
```

`outputs.tf`:

```hcl
output "whoami" {
  value = {
    account = data.aws_caller_identity.current.account_id
    arn     = data.aws_caller_identity.current.arn
    region  = data.aws_region.current.region   # v6: "region" (the old "name" is deprecated)
    zones   = data.aws_availability_zones.available.names
  }
}
```

`terraform.tfvars` (your values; your email is personal data, so keep it out of public repos):

```hcl
account_id  = "123456789012"
alert_email = "you@example.com"
```

AWS Budgets: the first two budgets in an account are free.

### Plan and apply

```bash
export AWS_PROFILE=tf-lab
aws sso login             # if you use SSO
terraform init
terraform apply
```

```
data.aws_region.current: Reading...
data.aws_caller_identity.current: Reading...
data.aws_availability_zones.available: Reading...
data.aws_region.current: Read complete after 0s [id=ap-south-1]
data.aws_caller_identity.current: Read complete after 0s [id=123456789012]
data.aws_availability_zones.available: Read complete after 1s [id=ap-south-1]

Terraform will perform the following actions:

  # aws_budgets_budget.lab will be created
  + resource "aws_budgets_budget" "lab" {
      + account_id   = (known after apply)
      + arn          = (known after apply)
      + budget_type  = "COST"
      + limit_amount = "5"
      + limit_unit   = "USD"
      + name         = "tf-course-monthly"
      + tags_all     = {
          + "managed_by" = "terraform"
          + "owner"      = "you"
          + "project"    = "tf-course"
        }
      + time_unit    = "MONTHLY"
      ...
      + notification { ... }
      + notification { ... }
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + whoami = {
      + account = "123456789012"
      + arn     = "arn:aws:sts::123456789012:assumed-role/AWSReservedSSO_AdministratorAccess_abc123/you"
      + region  = "ap-south-1"
      + zones   = [
          + "ap-south-1a",
          + "ap-south-1b",
          + "ap-south-1c",
        ]
    }
```

- The data sources were read at plan time (topic 19), so the output is already known.
- `tags_all`: `default_tags` were applied automatically. We wrote no `tags` on the budget.

### The safety catch in action

```bash
terraform plan -var="account_id=999999999999"
```

```
╷
│ Error: Invalid provider configuration
│
│ AWS account ID not allowed: 123456789012
╵
```

Terraform stopped before reading or changing anything. In real projects, `account_id` is fixed per environment folder (topic 49), so using the wrong profile in the wrong folder fails like this.

### Credentials debugging

```bash
env | grep ^AWS_                         # anything overriding your profile?
aws sts get-caller-identity              # what the CLI sees
TF_LOG=DEBUG terraform plan 2>&1 | grep -iE 'credential|caller identity|assume' | head
```

```
[DEBUG] provider.terraform-provider-aws_v6.73.0_x5: Retrieved credentials: tf_aws.credentials_source=SharedConfigCredentials: /home/you/.aws/config ...
```

The provider logs **which source** in the chain it used (the exact wording varies between versions).

---

## Hands-on lab

1. Build the project, fill in `terraform.tfvars`, apply. Check **Billing → Budgets** in the console: the budget is there, with tags.
2. Trigger the `allowed_account_ids` error.
3. Break auth on purpose: `AWS_PROFILE=nope terraform plan` → read the error. Then `export AWS_ACCESS_KEY_ID=AKIAFAKE AWS_SECRET_ACCESS_KEY=fake` and plan: the env vars **win** over your profile (step 2 of the chain) and fail. `unset` them.
4. Add a `provider "aws" { alias = "virginia" region = "us-east-1" }` and a `data "aws_region" "virginia" { provider = aws.virginia }`, then output its region.
5. Add `tags = { cost_center = "lab" }` to the budget and plan: see `tags` and `tags_all`.
6. Keep the budget (it's free and protects you) **or** `terraform destroy`. If you keep it, keep this folder and its state.

---

## Where you'll use it

- **Every AWS project**: a `providers.tf` with region, `default_tags` and `allowed_account_ids`.
- **Multi-account landing zones**: one role per account assumed by the pipeline; aliases for shared DNS or logging accounts.
- **CI/CD**: no keys; GitHub Actions OIDC sets env credentials (topic 57); Jenkins agents on EC2 use instance roles (topic 58).
- **Cost allocation**: `default_tags` with `project`, `env`, `owner`, `cost_center` → Cost Explorer reports by tag.

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `No valid credential sources found` | Nothing in the chain worked. | `export AWS_PROFILE=...`; `aws sso login`; check `aws sts get-caller-identity`. |
| `failed to refresh cached credentials, the SSO session has expired` | SSO token expired. | `aws sso login`. |
| `AWS account ID not allowed: ...` | `allowed_account_ids` protection: wrong account/profile. | Use the right profile. That's the point. |
| `AccessDenied ... is not authorized to perform: sts:AssumeRole` | The base identity can't assume `role_arn`, or the role's **trust policy** doesn't trust it. | Fix the trust policy / permissions. |
| `InvalidClientTokenId` with a profile that works in the CLI | Stale `AWS_ACCESS_KEY_ID` env vars override it. | `unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN`. |
| Endless diffs on `tags_all` | Tags added by another system, or the same key in `default_tags` and `tags` with different values. | `ignore_tags`; don't duplicate keys. |
| `Warning: Deprecated attribute ... name` on `aws_region` | v6 renamed it. | Use `.region`. |
| `Missing required argument: region` | No region anywhere. | Set `region` in the provider (or `AWS_REGION`). |

---

## Quick recap

- `provider "aws"` = region + optional `assume_role`, `default_tags`, `allowed_account_ids`. **Never keys in code.**
- Credential chain: provider args → env vars → profile files (SSO/keys/roles) → web identity → container role → instance role.
- `default_tags` tag everything; `tags_all` shows the merged result.
- Aliases for other accounts/regions; v6 adds a per-resource `region` argument.
- Debug auth with `env | grep AWS_`, `aws sts get-caller-identity`, `TF_LOG=DEBUG`.

### Interview questions

1. **Q: How does the AWS provider find credentials?**
   A: In order: explicit provider arguments, environment variables, the shared config/credentials files (profile, SSO, role chaining), web identity tokens, container credentials, then the EC2 instance role. If `assume_role` is set, it then assumes that role.
2. **Q: How do you stop someone applying dev code to the production account?**
   A: `allowed_account_ids` in the provider block (fails if the credentials are for another account), separate state and credentials per environment, and pipeline roles that can only reach one account.
3. **Q: How would you manage resources in two AWS accounts from one configuration?**
   A: Two `provider "aws"` blocks, one aliased, each with its own `assume_role`, and set `provider = aws.<alias>` on the resources (or pass providers into modules).
