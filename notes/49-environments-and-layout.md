# 49 · Environments and Repository Layout

> **Level:** ★★★ advanced
>
> **You will learn:** the standard way to run **dev, staging and prod** from one codebase: a **folder per environment × component**, the **same modules** with different **`.tfvars`** inputs, a **separate state** (ideally a separate **AWS account**) per environment, and **promotion** of changes from dev to prod. You'll compare **monorepo vs multi-repo**, see how `CODEOWNERS` and pipelines fit, and learn what **Terragrunt** adds (DRY backends, dependencies between units, `run --all`) and when it's worth it.
>
> **You need:** topics 30 (`allowed_account_ids`), 35 (partial backend config), 38, 44–48.
>
> **Part 7 · Environments and team structure** · Previous: [48 · Workspaces](48-workspaces.md) · Next: [50 · HCP Terraform and other platforms](50-hcp-terraform-and-platforms.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Each environment is a **separate root module** (folder) with its own **state** and **credentials**, calling the **same modules** with **different inputs**:

```
infra/
├── modules/                    # shared code
│   ├── network/
│   └── web-service/
└── envs/
    ├── dev/
    │   ├── network/  main.tf  backend.tf  terraform.tfvars   ← state: dev account
    │   └── app/      ...
    ├── staging/ ...
    └── prod/
        ├── network/  main.tf  backend.tf  terraform.tfvars   ← state: prod account
        └── app/      ...
```

```hcl
# envs/prod/app/main.tf
module "app" {
  source        = "../../../modules/web-service"
  instance_type = var.instance_type        # "t3.large" in prod/terraform.tfvars, "t3.micro" in dev
  min_size      = var.min_size
}
```

**Picture it.** A franchise restaurant chain: one **recipe book** (modules), but each branch (environment) has its own **kitchen, keys and accounts** (state, credentials), and adjusts quantities to its size (tfvars).

**The one thing to know.** The folder you're in **is** the environment. No hidden selection (unlike workspaces), and prod can live in a different AWS account with different credentials.

**Try it now:** turn topic 33's web server into `modules/web` + `envs/dev` + `envs/prod` roots with different tfvars and state keys.

**You've got it if you can:** explain how a change is promoted from dev to prod in this layout.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### The building blocks

| Piece | Per environment | Shared |
|---|---|---|
| **Root module folder** | `envs/prod/app/` | — |
| **Backend / state** | Separate key (and ideally bucket in the env's account) | — |
| **Credentials** | Separate AWS account/role; `allowed_account_ids` per env | — |
| **Inputs** | `terraform.tfvars` (sizes, counts, CIDRs, domains, flags) | — |
| **Modules** | — | `modules/` (by path, or versioned) |
| **Provider/Terraform versions** | Can differ temporarily during upgrades | Usually aligned |

### A root module per environment (thin)

```hcl
# envs/prod/app/backend.tf
terraform {
  backend "s3" {
    bucket       = "acme-tfstate-prod-222222222222"
    key          = "app/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

```hcl
# envs/prod/app/providers.tf
provider "aws" {
  region              = "ap-south-1"
  allowed_account_ids = ["222222222222"]          # prod account only
  default_tags {
    tags = { environment = "prod", repo = "acme/infra", stack = "app" }
  }
}
```

```hcl
# envs/prod/app/terraform.tfvars
instance_type = "t3.large"
min_size      = 3
max_size      = 10
domain        = "shop.acme.com"
```

The `main.tf` and `variables.tf` are often **identical** across environments; only `backend.tf`, `providers.tf` and `terraform.tfvars` differ. That repetition is small, explicit, and easy to review.

### Alternative: one root, per-env backend + tfvars files

```
app/
├── main.tf  variables.tf  outputs.tf
├── backends/  dev.s3.tfbackend   prod.s3.tfbackend
└── vars/      dev.tfvars         prod.tfvars
```

```bash
terraform init  -backend-config=backends/prod.s3.tfbackend -reconfigure
terraform plan  -var-file=vars/prod.tfvars
```

Less duplication, but the environment is chosen by **command-line flags**, so wrappers/pipelines must get them right every time. Many teams prefer folders for prod clarity.

### Promotion

How a change reaches prod safely:

| Strategy | How | Pros / cons |
|---|---|---|
| **Same commit, environment by environment** | One PR changes a module; the pipeline applies `dev` → (tests, approval) → `staging` → (approval) → `prod` | Simple; but a merged change is "in flight" for prod until applied |
| **Versioned modules** | `dev` points at module `v1.5.0`; after it's proven, a PR bumps `prod` from `v1.4.0` to `v1.5.0` | Explicit promotion per env; more PRs; requires versioned modules (topic 45) |

### Monorepo vs multi-repo

| | Monorepo (`infra/` holds all modules and envs) | Multi-repo (modules in their own repos; one repo per team/stack) |
|---|---|---|
| Changes across module + envs | One PR | Several PRs, in order |
| Module versioning | Optional (path references) | Required (tags) |
| Permissions | Via `CODEOWNERS` + pipeline rules per folder | Natural per repo |
| CI | Needs "which folders changed?" logic | Each repo's CI is simple |
| Typical for | Small/medium orgs, platform teams | Large orgs, many independent teams |

`CODEOWNERS` (a GitHub/GitLab file) assigns reviewers by path: `envs/prod/  @acme/platform-leads` makes prod changes need their approval.

### Terragrunt

**Terragrunt** (by Gruntwork) is a thin wrapper around Terraform/OpenTofu for many-unit layouts. It adds:

| Feature | Problem it solves |
|---|---|
| **`remote_state` generation** | Writes the `backend` block for every unit from one place (no copy-paste; can auto-create the bucket) |
| **`include`** of a root config (`root.hcl`) | Shared provider/backend settings defined once |
| **`dependency` blocks** | Read another unit's outputs (like `terraform_remote_state`, but with ordering) |
| **`run --all`** | Plan/apply many units in dependency order (`terragrunt run --all plan`; older versions: `run-all`) |
| **Stacks** (`terragrunt.stack.hcl`) | Generate many units from a template |

A unit:

```hcl
# envs/prod/app/terragrunt.hcl
include "root" {
  path = find_in_parent_folders("root.hcl")   # backend + provider settings for all units
}

terraform {
  source = "../../../modules//web-service"
}

dependency "network" {
  config_path = "../network"
}

inputs = {
  vpc_id        = dependency.network.outputs.vpc_id
  subnet_ids    = dependency.network.outputs.public_subnet_ids
  instance_type = "t3.large"
}
```

When it's worth it: dozens of units across many accounts/regions, where backend/provider repetition and cross-unit ordering become painful. For a handful of roots, plain Terraform folders are simpler. (Terramate is a similar tool focused on orchestration and change detection.)

---

## Why it exists

Every company needs at least a non-prod and a prod environment, and they must be **the same shape** (so tests mean something) but **different in size, access and risk**. The layout decides whether a mistake in dev can touch prod, whether reviewers can see what's about to change, and how quickly changes flow safely.

---

## Analogy

**A restaurant franchise.**

- **One recipe book** (modules), improved centrally.
- Each **branch** (environment) has its own **kitchen, safe, keys and bank account** (state, credentials, AWS account).
- Each branch adjusts **quantities** (tfvars): a small test kitchen cooks 10 portions, the flagship 500.
- A new dish is tried in the **test kitchen first** (dev), then rolled out to branches (promotion).
- **Terragrunt** = a regional manager's clipboard that sets up every new branch's safe and keys the same way, and knows which branch must open first.

---

## How it works inside

Nothing new happens inside Terraform: each environment folder is an independent root module with its own init, state, lock and plan (topics 13, 35). The layout is about **people and pipelines**:

```
PR changes modules/web-service + envs/*/app
  CI: detect changed roots → plan dev/app, staging/app, prod/app (each with its own role)
  reviewers read three plans; CODEOWNERS for envs/prod requires platform-leads
merge → apply dev (auto) → smoke tests → apply staging (approval) → apply prod (approval)
```

---

## Code / commands

A tiny but complete version using your local Docker provider (free) so you can run it:

```
49-envs/
├── modules/web/        variables.tf  main.tf  outputs.tf
└── envs/
    ├── dev/   main.tf  terraform.tfvars
    └── prod/  main.tf  terraform.tfvars
```

`modules/web/main.tf`:

```hcl
terraform {
  required_providers {
    docker = { source = "kreuzwerker/docker", version = ">= 4.0" }
  }
}

variable "env"      { type = string }
variable "replicas" { type = number }
variable "port"     { type = number }

resource "docker_image" "nginx" {
  name         = "nginx:1.29"
  keep_locally = true
}

resource "docker_container" "web" {
  count = var.replicas                          # identical replicas: count is fine (topic 21)
  name  = "web-${var.env}-${count.index}"
  image = docker_image.nginx.image_id
  ports {
    internal = 80
    external = var.port + count.index
  }
}

output "urls" {
  value = [for c in docker_container.web : "http://localhost:${c.ports[0].external}"]
}
```

`envs/dev/main.tf` (and the same in `envs/prod/`, only the backend key differs):

```hcl
terraform {
  required_version = ">= 1.14"
  required_providers {
    docker = { source = "kreuzwerker/docker", version = "~> 4.7" }
  }
  backend "s3" {
    bucket       = "tf-course-state-123456789012"
    key          = "lab/49/dev/web/terraform.tfstate"     # prod: lab/49/prod/web/...
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}

provider "docker" {}

variable "env"      { type = string }
variable "replicas" { type = number }
variable "port"     { type = number }

module "web" {
  source   = "../../modules/web"
  env      = var.env
  replicas = var.replicas
  port     = var.port
}

output "urls" { value = module.web.urls }
```

`envs/dev/terraform.tfvars`: `env = "dev"`, `replicas = 1`, `port = 8100`.
`envs/prod/terraform.tfvars`: `env = "prod"`, `replicas = 3`, `port = 8200`.

```bash
cd envs/dev  && terraform init && terraform apply -auto-approve
cd ../prod   && terraform init && terraform apply -auto-approve
```

```
# dev
urls = ["http://localhost:8100"]
# prod
urls = ["http://localhost:8200", "http://localhost:8201", "http://localhost:8202"]
```

Two environments, one module, separate states. A change to `modules/web` shows up in **both** plans; you apply dev first.

---

## Hands-on lab

1. Build the Docker version above. Apply dev, then prod.
2. Change the module (e.g. add a `restart = "unless-stopped"` argument) and plan **both** environments. Apply dev, check it, then prod: your first promotion.
3. Convert to the "one root + `-backend-config` + `-var-file`" style and compare which is easier to get wrong.
4. Write a `CODEOWNERS` file that requires a specific reviewer for `envs/prod/`.
5. (Optional) Install Terragrunt and express the two envs with a `root.hcl` that generates the backend block; run `terragrunt run --all plan` from `envs/`.
6. `terraform destroy` in both environments.

---

## Where you'll use it

- **Every real Terraform repo**: deciding and explaining the layout is a senior skill and a common interview design question.
- **Multi-account AWS**: one state bucket and pipeline role per account; prod roles only usable from the main branch's pipeline (topic 57).
- **Terragrunt** shops: common in consultancies and large AWS estates.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| dev and prod drift apart over months | Copy-pasted resources instead of shared modules | Same modules; only tfvars differ |
| A dev change accidentally affected prod | Shared state or credentials (workspaces, one account) | Separate state keys/buckets and accounts; `allowed_account_ids` |
| Prod change applied before anyone tested it | No promotion order | Pipeline: dev → staging → prod with approvals |
| CI plans every environment on every PR (slow) | No change detection in a monorepo | Plan only roots whose files (or used modules) changed |
| Backend blocks copy-pasted with mistakes | Many units | Partial config files, a generator, or Terragrunt |
| `dependency` outputs missing in Terragrunt | Dependency not applied yet | Apply in order (`run --all` handles it), or `mock_outputs` for plans |

---

## Quick recap

- Environment = **separate root folder + separate state + separate credentials** (ideally separate account), calling **shared modules** with **different tfvars**.
- Promote changes dev → staging → prod (same commit in order, or versioned module bumps per env).
- Monorepo (one PR, CODEOWNERS) vs multi-repo (natural ownership, versioned modules).
- **Terragrunt** removes backend/provider repetition and orders cross-unit dependencies; use it when you have many units.

### Interview questions

1. **Q: How do you manage multiple environments in Terraform?**
   A: A separate root configuration per environment (and component) with its own backend state and credentials, ideally in separate AWS accounts, all calling the same modules with environment-specific tfvars. Changes are promoted through dev and staging before prod via the pipeline.
2. **Q: Why not workspaces for environments?**
   A: They share one backend and credential set, the selected environment is invisible in the code, and environments need different settings and access controls. Separate roots make each environment explicit and isolated.
3. **Q: What does Terragrunt add on top of Terraform?**
   A: DRY backend and provider configuration generated for every unit, dependencies between units with output passing, running commands across many units in dependency order, and templated stacks. It helps most with many units across accounts and regions.
