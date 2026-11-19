# 47 · Module Design Patterns

> **Level:** ★★★ advanced
>
> **You will learn:** how to structure many modules into a maintainable system: **composition** (thin root modules wiring small modules together), **dependency inversion** (pass objects in rather than creating or looking them up inside), the module **hierarchy** (resource modules → service modules → root/environment configurations), **data-only modules**, sizing modules for **blast radius**, the classic **anti-patterns** (god modules, wrapper chains, nested provider configs, flag soup), and how modules relate to state boundaries.
>
> **You need:** topics 38 (sharing state), 43–46.
>
> **Part 6 · Modules** · Previous: [46 · Using public modules](46-public-modules.md) · Next: [48 · Workspaces](48-workspaces.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Good Terraform systems are built like LEGO: **small modules that do one thing**, combined (**composed**) in a **thin root module** per environment. Modules receive what they need as **inputs** instead of creating or looking it up themselves.

```hcl
# environments/prod/main.tf: a thin root that only wires modules together
module "network" {
  source = "../../modules/network"
  cidr   = "10.20.0.0/16"
}

module "database" {
  source     = "../../modules/postgres"
  vpc_id     = module.network.vpc_id               # passed in, not looked up
  subnet_ids = module.network.private_subnet_ids
}

module "app" {
  source        = "../../modules/web-service"
  vpc_id        = module.network.vpc_id
  subnet_ids    = module.network.public_subnet_ids
  database_host = module.database.endpoint
}
```

**Picture it.** A Lego model: standard bricks (resource modules), a few sub-assemblies (service modules), and one instruction page per model (the root) that just says which pieces snap together.

**The one thing to know.** If you can't describe a module in one sentence without "and", it's probably two modules.

**Try it now:** take your topic 27 or topic 33 project and sketch which modules you'd split it into, and what each would take as inputs and return as outputs.

**You've got it if you can:** explain composition and dependency inversion, and name three anti-patterns.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### The module hierarchy

| Layer | What | Examples | Who writes it |
|---|---|---|---|
| **Resource modules** | A small, secure building block: a few closely related resources | `s3-secure-bucket` (topic 44), `kms-key`, `iam-role-for-service`, `security-group` | Platform team, or public modules |
| **Service / infrastructure modules** | A meaningful unit built from resource modules | `network` (VPC + endpoints + flow logs), `postgres` (RDS + subnet group + SG + secret + alarms), `web-service` (ALB + ASG + roles) | Platform team |
| **Root modules (compositions)** | One per environment × component; thin wiring, backends, providers | `environments/prod/network`, `environments/prod/app` | Each team |

### Composition

The root's job is to **connect** modules: outputs of one become inputs of another. Benefits:

- Each module stays small, testable and reusable.
- The root reads like an architecture diagram.
- Swapping one piece (e.g. `postgres` → `aurora`) touches one block.

### Dependency inversion

"Don't create or find your dependencies; **receive** them."

```hcl
# ❌ the module creates its own VPC: can't share a VPC between apps
module "app" { source = "./modules/app-with-its-own-vpc" }

# ❌ the module looks up the VPC by name: hidden dependency on naming
#    (inside the module: data "aws_vpc" "x" { tags = { Name = "main" } })

# ✅ the module receives the VPC: works with any VPC, in any test
module "app" {
  source     = "./modules/app"
  vpc_id     = module.network.vpc_id
  subnet_ids = module.network.private_subnet_ids
}
```

Pass **IDs or small objects**, not whole resources. A module can accept an object input like:

```hcl
variable "network" {
  type = object({
    vpc_id             = string
    private_subnet_ids = list(string)
  })
}
```

and callers pass `network = module.network` if the network module has matching outputs: a clean "interface".

### Data-only modules

A module with **no resources**, only data sources and outputs, that encapsulates "how we find things":

```hcl
# modules/shared-network-lookup/main.tf
data "aws_vpc" "shared" {
  tags = { Name = "shared-${var.environment}" }
}
data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.shared.id]
  }
  tags = { tier = "private" }
}
output "vpc_id"             { value = data.aws_vpc.shared.id }
output "private_subnet_ids" { value = data.aws_subnets.private.ids }
```

Consumers in other states (topic 38) call this instead of copying lookup logic everywhere. If the lookup rules change, one module changes.

### Module boundaries vs state boundaries

| | Module boundary | State boundary (separate root/configuration) |
|---|---|---|
| Purpose | **Code reuse** and encapsulation | **Blast radius**, permissions, speed, ownership |
| Shared plan/apply? | Yes: same graph | No: separate runs |
| Values passed by | Inputs/outputs (same run) | Remote state, SSM, lookups (topic 38) |

Split **state** by lifecycle and ownership (network vs app); split **modules** by reuse. They often line up, but not always: one state can call many modules; one module can be called by many states.

### Sizing for blast radius

- A root module (state) that contains the VPC, the database and twenty services means one bad PR can affect all of them, and every plan touches everything.
- Typical split: `network` (rarely changes) / `data` (databases, careful changes) / `platform` (clusters, shared services) / `apps` (frequent changes), per environment.
- Inside a state, **`prevent_destroy`** on stateful resources (topic 24).

### Anti-patterns

| Anti-pattern | Symptom | Fix |
|---|---|---|
| **God module** | One module creates a whole environment; 100+ inputs; every change risks everything | Split into service modules composed in roots |
| **Wrapper of a wrapper of a wrapper** | Five layers to find where a value is set; each passes 50 variables through | Max ~2–3 levels; flatten |
| **Flag soup** | `create_vpc`, `create_db`, `enable_x`, `enable_y`... combinations nobody tests | Separate modules; composition |
| **Provider blocks in child modules** | No `for_each`; broken removals | Providers only in roots (topic 44) |
| **Hidden lookups** | Modules find resources by name | Dependency inversion |
| **Copy-paste environments** | `dev/` and `prod/` hold full copies of all resources, drifting apart | Same modules, different inputs (topic 49) |
| **Module per resource** | `module "bucket_policy"` wrapping one resource with no added value | Use the resource directly |
| **Huge outputs** | `output "all" { value = aws_db_instance.this }` | Specific outputs |

---

## Why it exists

Modules are easy to start and hard to get right. Without structure, codebases drift into a few giant modules that nobody dares change, or hundreds of tiny ones nobody can follow. These patterns, borrowed from software design (single responsibility, composition over inheritance, dependency injection), keep Terraform **changeable** over years and many teams.

---

## Analogy

**Lego.**

- **Bricks** (resource modules): standard, tested, used everywhere.
- **Sub-assemblies** (service modules): "a wheel with axle", built from bricks.
- **Instruction page** (root module): "attach two wheels to the chassis". It doesn't make bricks; it connects them.
- **Dependency inversion**: a wheel doesn't come with its own car glued on; you **give** it the axle.
- **God module**: a model glued together in one piece. You can't change the wheels without breaking the roof.

---

## How it works inside

All modules in one root share one graph (topic 43), so composition costs nothing at run time: Terraform sees `module.app.aws_lb.this` depending on `module.network.aws_subnet.public[0]` through the input wiring, and runs everything else in parallel. The **design** choices above are for humans: readability, reuse, safe change.

---

## Code / commands

A realistic repository layout combining everything:

```
infra/
├── modules/                          # reusable modules (versioned, or used by path in a monorepo)
│   ├── s3-secure-bucket/             # resource module (topic 44)
│   ├── network/                      # service module: wraps terraform-aws-modules/vpc (topic 46)
│   ├── postgres/                     # service module: RDS + SG + subnet group + secret + alarms
│   └── web-service/                  # service module: ALB + ASG + IAM + SG
└── environments/
    ├── dev/
    │   ├── network/                  # root: backend key dev/network, calls modules/network
    │   ├── data/                     # root: reads network outputs (SSM), calls modules/postgres
    │   └── app/                      # root: reads network + data outputs, calls modules/web-service
    └── prod/
        ├── network/
        ├── data/
        └── app/
```

A service module composing resource modules (`modules/postgres/main.tf`, sketch):

```hcl
module "sg" {
  source = "../security-group"
  name   = "${var.name}-db"
  vpc_id = var.network.vpc_id
  ingress_from_security_groups = { postgres = { port = 5432, sg_ids = var.allowed_security_group_ids } }
}

resource "aws_db_subnet_group" "this" {
  name       = var.name
  subnet_ids = var.network.private_subnet_ids
}

resource "aws_db_instance" "this" {
  identifier                  = var.name
  engine                      = "postgres"
  engine_version              = var.engine_version
  instance_class              = var.instance_class
  allocated_storage           = var.storage_gb
  db_subnet_group_name        = aws_db_subnet_group.this.name
  vpc_security_group_ids      = [module.sg.id]
  manage_master_user_password = true        # AWS stores the password in Secrets Manager (topic 51)
  username                    = "app_admin"
  storage_encrypted           = true
  deletion_protection         = var.deletion_protection
  skip_final_snapshot         = !var.deletion_protection
}

output "endpoint" { value = aws_db_instance.this.address }
output "security_group_id" { value = module.sg.id }
output "master_secret_arn" { value = aws_db_instance.this.master_user_secret[0].secret_arn }
```

Note the trade-off: `lifecycle { prevent_destroy }` can't take variables (topic 24), so a module can't switch it on only in prod. Modules usually rely on the provider's `deletion_protection` argument, driven by a variable, for per-environment protection. (`module "sg"` assumes a small security-group resource module of your own; its inputs are illustrative.)

---

## Hands-on lab

1. Refactor topic 33's single configuration into `modules/network` (from topic 32 code), `modules/web-server` (security group, IAM, instance) and a thin root that composes them. Use `moved` blocks so the existing deployment isn't replaced (if it's still running), or start fresh.
2. Make `web-server` take a `network` object input (dependency inversion); pass `module.network`.
3. Write a data-only module that finds your VPC by tag, and use it from a second root module (topic 38's consumer).
4. Review one of your earlier labs and list its anti-patterns.
5. `terraform destroy` everything you created.

---

## Where you'll use it

- **Designing a new Terraform codebase** (an interview favourite: "How would you structure Terraform for 10 teams and 3 environments?").
- **Refactoring** legacy god modules into composable pieces with `moved` blocks.
- **Platform engineering**: a catalogue of resource and service modules, documented and versioned.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| Every plan touches hundreds of resources and takes 10 minutes | God state / god module | Split states by lifecycle; compose modules |
| Nobody knows where a value is set | Deep wrapper chains, pass-through variables | Flatten; ≤ 3 levels |
| A module can't be reused in another account/VPC | Hidden lookups or hard-coded values | Dependency inversion |
| dev and prod behave differently "for no reason" | Copy-pasted code drifting | Same modules, different inputs |
| Small change → `Cycle` errors across modules | Modules referencing each other both ways | Re-cut boundaries; pass values one way |

---

## Quick recap

- Hierarchy: **resource modules** → **service modules** → **thin root modules** per environment × component.
- **Composition**: roots wire outputs to inputs. **Dependency inversion**: modules receive IDs/objects; they don't create or look up their dependencies.
- **Data-only modules** centralise lookups.
- Split **state** by blast radius/ownership; split **modules** by reuse.
- Avoid god modules, wrapper chains, flag soup, provider blocks in children, hidden lookups and copy-paste environments.

### Interview questions

1. **Q: How would you structure Terraform modules for a medium-sized company?**
   A: Small resource modules (secure bucket, KMS key, IAM role), service modules composed from them (network, database, web service), and thin root modules per environment and component with their own state, wiring modules together through inputs and outputs, versioned and tested.
2. **Q: What is dependency inversion in Terraform modules?**
   A: Modules receive their dependencies, such as VPC and subnet IDs, as inputs instead of creating them or looking them up by name inside. That makes them reusable in any context and testable.
3. **Q: What's a "god module" and why is it bad?**
   A: A single module that builds an entire environment with huge numbers of inputs and flags. Changes are risky, plans are slow, it's hard to test and every team is coupled to it. Split it into focused modules composed in root configurations.
