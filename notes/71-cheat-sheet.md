# 71 · Cheat Sheet

> **Level:** ★ reference
>
> **You will learn:** nothing new. This is a **one-page-style reference** of the commands, flags, blocks, meta-arguments, expressions, functions, environment variables, files and "symptom → first move" pairs used in the course, each with the topic that explains it. Keep it open while you work.
>
> **You need:** the topics referenced.
>
> **Part 11 · Interviews and reference** · Previous: [70 · Senior scenario interviews](70-senior-scenario-interviews.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: how to use it

Not for reading top to bottom. Search it (Ctrl-F) when you need syntax. Test: close it, write your own from memory, then compare. What you forgot is what to practise next.

---

## Commands

### Everyday (09)

```bash
terraform init [-upgrade] [-backend-config=FILE] [-reconfigure|-migrate-state] [-backend=false] [-lockfile=readonly]
terraform fmt [-check] [-diff] [-recursive]
terraform validate [-json]
terraform plan [-out=tfplan] [-var 'k=v'] [-var-file=f.tfvars] [-detailed-exitcode] [-input=false]
               [-lock-timeout=5m] [-parallelism=N] [-refresh=false] [-refresh-only] [-replace=ADDR]
               [-destroy] [-target=ADDR ⚠️] [-generate-config-out=gen.tf] [-no-color] [-json]
               [-minimal-refresh (1.17+)] [-policies=PATH (1.17+)]
terraform apply [tfplan] [-auto-approve ⚠️] [-replace=ADDR] [-refresh-only] [-invoke=action.TYPE.NAME (1.14+)]
terraform destroy                       # = apply -destroy
terraform show [tfplan] [-json]
terraform output [NAME] [-raw|-json]
terraform console                       # evaluate expressions (holds the lock)
terraform graph [-format=mermaid] [-type=plan]
terraform providers [lock -platform=OS_ARCH ...] [mirror DIR] [schema -json]
terraform version
terraform test [-filter=FILE] [-verbose]
terraform login                         # HCP Terraform
terraform query [-generate-config-out=gen.tf]   # 1.14+, runs *.tfquery.hcl list blocks
```

### State (12, 34, 42)

```bash
terraform state list [ADDR]
terraform state show 'ADDR'
terraform state pull > backup.tfstate
terraform state push [-force ⚠️] FILE
terraform state mv SRC DST              # prefer moved blocks
terraform state rm ADDR                 # prefer removed blocks
terraform state replace-provider FROM TO
terraform force-unlock LOCK_ID          # only if the holder is dead
terraform untaint ADDR
terraform workspace list|show|new|select [-or-create]|delete NAME
```

### Exit codes (09)

| Command | 0 | 1 | 2 |
|---|---|---|---|
| `plan -detailed-exitcode` | no changes | error | changes present |

---

## Plan symbols (09)

| Symbol | Meaning |
|---|---|
| `+` | create |
| `-` | destroy |
| `~` | update in place |
| `-/+` | replace (destroy first) |
| `+/-` | replace (create first: `create_before_destroy`) |
| `<=` | read data source during apply |
| `.` | forget (removed, `destroy = false`) |
| `# forces replacement` | the argument causing a replacement |
| `(known after apply)` | value unknown until apply |
| `has moved to` | address change via `moved` |
| `will be imported` | `import` block |

---

## Blocks (08)

Every snippet below is valid HCL you can copy. (HCL allows **one argument per line**; a one-line block `{ ... }` may hold at most one argument.) `...` marks where your own values go.

```hcl
terraform {
  required_version = ">= 1.14"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.0" }   # object: commas OK
  }
  backend "s3" {
    bucket       = "my-state-bucket"
    key          = "env/comp/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true                     # 1.10+ (GA 1.11)
  }
  # or instead of backend:
  # cloud {
  #   organization = "acme"
  #   workspaces { name = "x" }
  # }
}

provider "aws" {
  region              = "ap-south-1"
  allowed_account_ids = ["123456789012"]
  default_tags {
    tags = { owner = "team", managed_by = "terraform" }
  }
  # alias = "us"
  # assume_role { role_arn = "arn:aws:iam::123456789012:role/x" }
}

variable "x" {
  type        = string
  default     = "a"
  description = "..."
  sensitive   = false
  nullable    = false
  # ephemeral  = true      # 1.10+
  # deprecated = "msg"     # 1.15+
  validation {
    condition     = length(var.x) > 0
    error_message = "x must not be empty."
  }
}

locals {
  name = "${var.project}-${var.env}"
}

resource "TYPE" "NAME" { }                # arguments go inside
data "TYPE" "NAME" { }
ephemeral "TYPE" "NAME" { }               # 1.10+

module "NAME" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 6.0"                      # registry sources only
}

output "NAME" {
  value       = aws_s3_bucket.x.id
  description = "..."
  sensitive   = false
}

import {                                  # 1.5+ (for_each 1.7+, identity 1.12+, in modules 1.16+)
  to = aws_s3_bucket.x
  id = "name"
}

moved {                                   # 1.1+
  from = aws_x.old
  to   = aws_x.new
}

removed {                                 # 1.7+
  from = aws_x.y
  lifecycle {
    destroy = false
  }
}

check "NAME" {                            # 1.5+
  data "http" "h" {
    url = "https://example.com/health"
  }
  assert {
    condition     = data.http.h.status_code == 200
    error_message = "Health check failed."
  }
}

action "TYPE" "NAME" {                    # 1.14+; TYPE comes from the provider
  config { }
}
# triggered from a resource:
#   lifecycle {
#     action_trigger {
#       events  = [after_create, after_update]
#       actions = [action.TYPE.NAME]
#     }
#   }
```

---

## Meta-arguments (20–24)

| Meta-argument | Use |
|---|---|
| `count = N` / `count.index` | N copies; `cond ? 1 : 0` toggles |
| `for_each = map/set` / `each.key`, `each.value` | Named instances |
| `depends_on = [res.x]` | Hidden dependencies |
| `provider = aws.alias` | Non-default provider config |
| `providers = { aws = aws.x }` (modules) | Pass providers to modules |
| `lifecycle { create_before_destroy, prevent_destroy, ignore_changes = [...], replace_triggered_by = [...], destroy = false, precondition {}, postcondition {} }` | Change behaviour |

---

## Expressions (17, 23)

```hcl
"${var.a}-${local.b}"                       # interpolation
cond ? a : b                                # conditional
var.list[0]   var.map["k"]   var.obj.attr   # access
res.x[*].id                                 # splat
[for s in var.l : upper(s) if s != ""]      # list for
{for k, v in var.m : k => v.port}           # map for
{for u in var.users : u.team => u.name...}  # grouping
flatten([for e in var.envs : [for s in var.svcs : {env = e, svc = s}]])
dynamic "ingress" {                         # generates nested blocks
  for_each = var.ports
  content {
    from_port = ingress.value
  }
}
<<-EOT ... EOT                              # heredoc
%{ for x in list ~}...%{ endfor ~}          # template loop; ~ strips whitespace
null                                        # "not set"
```

Types: `string number bool list(T) set(T) map(T) object({...}) tuple([...]) any`, `optional(T, default)`.

---

## Functions you'll use (18)

| Group | Functions |
|---|---|
| Strings | `lower upper format join split replace trimspace substr startswith endswith strcontains regex` |
| Collections | `length concat contains distinct flatten element index slice sort range one toset tolist tomap setsubtract keys values merge lookup zipmap` |
| Errors/types | `try can coalesce nonsensitive sensitive type(console)` |
| Encoding | `jsonencode jsondecode yamlencode yamldecode base64encode` |
| Files | `file fileexists templatefile fileset filesha1 abspath` |
| Network | `cidrsubnet cidrsubnets cidrhost cidrnetmask` |
| Time/hash | `plantimestamp formatdate timeadd sha256 md5` (⚠️ `timestamp()`, `uuid()` change every run) |
| Provider-defined | `provider::aws::arn_parse(...)` |

Paths: `path.module`, `path.root`, `path.cwd`. Workspace: `terraform.workspace`.

---

## Environment variables (05)

| Variable | Purpose |
|---|---|
| `TF_LOG` / `TF_LOG_CORE` / `TF_LOG_PROVIDER` / `TF_LOG_PATH` | Debug logging (64) |
| `TF_VAR_name` | Set a variable (14) |
| `TF_INPUT=0`, `TF_IN_AUTOMATION=1` | CI behaviour (09) |
| `TF_CLI_ARGS`, `TF_CLI_ARGS_plan` | Extra flags |
| `TF_PLUGIN_CACHE_DIR` | Provider cache (06, 63) |
| `TF_WORKSPACE` | Select workspace (48) |
| `TF_TOKEN_app_terraform_io` | HCP Terraform token (50) |
| `TF_CLI_CONFIG_FILE` | Alternative `.terraformrc` (62) |
| `AWS_PROFILE`, `AWS_REGION`, `AWS_ACCESS_KEY_ID`... | AWS credentials (30) |

---

## Files (05, 10, 35)

| File | Commit? |
|---|---|
| `*.tf`, `.terraform.lock.hcl`, non-secret `*.tfvars`, `*.tfbackend`, `.tflint.hcl`, `tests/*.tftest.hcl` | ✅ |
| `.terraform/`, `*.tfstate*`, `tfplan`/`*.tfplan`, `crash.log`, `errored.tfstate`, secret tfvars, `.terraformrc` | ❌ |
| `.terraform/terraform.tfstate` | Backend settings only (not your state) |
| `terraform.tfstate.d/` | Local workspace states |

---

## Version constraints (10)

`= 1.2.3` exact · `>= 1.2` minimum · `~> 1.2` ≥1.2,<2.0 · `~> 1.2.0` ≥1.2.0,<1.3.0 · `!= 1.2.5` exclude. Roots: `~> MAJOR.MINOR` + lock file. Modules: minimums.

---

## Symptom → first move

| Symptom | First move | Topic |
|---|---|---|
| Plan creates everything | Wrong folder/key/workspace? Don't apply | 34, 42 |
| Plan destroys unexpected things | Look for renames → `moved`; check `git log` | 40 |
| `forces replacement` | Which argument? Revert, CBD, or migration | 24 |
| Lock error | Read lock info; wait; checklist; `force-unlock` | 36 |
| `Saved plan is stale` | Re-plan, re-review | 09 |
| `errored.tfstate` | Don't re-apply; `state push errored.tfstate` | 42 |
| `AccessDenied` | `sts get-caller-identity`; the named action | 29, 64 |
| `No valid credential sources` | `AWS_PROFILE`, `aws sso login`, OIDC permissions | 30, 57 |
| `already exists` | Import it (or unique names) | 39 |
| `DependencyViolation` | Destroy dependents first; find unmanaged ENIs/SGs | 20, 32 |
| Drift notes in the plan | Revert / adopt / accept / ignore | 37 |
| Perpetual diff | Impure functions, normalisation, `ignore_changes` | 18, 24 |
| `Invalid for_each/count argument` | Plan-time-known keys/counts | 21, 22 |
| `Cycle` | Remove a reference; separate link resources | 20 |
| Version/lock errors | `init` / `init -upgrade` / `providers lock` | 10, 62 |
| Provider crash / inconsistent result | Upgrade; report with `TF_LOG` | 64 |

---

## What it is / Why it exists / Analogy / How it works inside / Hands-on lab / Where you'll use it / When it breaks / Quick recap

A laminated card next to the machine: reference, not explanation. Each line points to the topic that teaches it. Lab: rewrite this page from memory once a week while you're learning; compare; study the gaps.

### Interview questions

Use [topic 69](69-interview-question-bank.md) and [topic 70](70-senior-scenario-interviews.md).
