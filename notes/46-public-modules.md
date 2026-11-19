# 46 · Using Public Modules

> **Level:** ★★ intermediate
>
> **You will learn:** how to use community modules from the **Terraform Registry**, using the most popular one, **`terraform-aws-modules/vpc/aws`**, to rebuild your topic 32 network in ~20 lines; how to **read** a module's documentation (inputs, outputs, examples); how to **judge** a module's quality and risk before trusting it with production; and when to **use directly, wrap, or fork** it.
>
> **You need:** topics 32 (VPC concepts), 43–45.
>
> **Part 6 · Modules** · Previous: [45 · Module sources and versioning](45-module-sources-and-versioning.md) · Next: [47 · Module design patterns](47-module-patterns.md) · [Back to INDEX](../INDEX.md)
>
> 💰 **Cost:** free with `enable_nat_gateway = false` (the default in this lab). Turning NAT on costs money.

---

## ⚡ Core: the 10-minute version

**The idea.** The community has already written well-tested modules for common infrastructure. The **`terraform-aws-modules`** organisation maintains the most-used AWS ones (VPC, EKS, RDS, S3, IAM, ALB...).

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 6.0"

  name = "tf-course"
  cidr = "10.0.0.0/16"

  azs             = ["ap-south-1a", "ap-south-1b"]
  public_subnets  = ["10.0.0.0/24", "10.0.1.0/24"]
  private_subnets = ["10.0.10.0/24", "10.0.11.0/24"]

  enable_nat_gateway   = false          # 💰 true costs ≈ USD 33+/month per gateway
  enable_dns_hostnames = true
}
```

That's topic 32's VPC, subnets, internet gateway, route tables and associations, in one block.

**Picture it.** Buying flat-pack furniture from a well-known brand instead of building it from raw wood. Faster, tested by thousands, but you still check the reviews, the instructions and that it fits your room.

**The one thing to know.** A public module is **someone else's code running with your credentials**. Pin the version, read what it creates (the plan), and check who maintains it.

**Try it now:** apply the block above (free) and count how many resources the plan creates.

**You've got it if you can:** list five things you check before adopting a public module.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Reading a module's page

On `registry.terraform.io/modules/terraform-aws-modules/vpc/aws`:

| Tab / section | What you learn |
|---|---|
| **Readme** | What it does, usage examples, notes |
| **Inputs** | Every variable: name, type, default, description (often 200+ for big modules: you set only what you need) |
| **Outputs** | What you can reference: `module.vpc.vpc_id`, `public_subnets`, `private_subnets`... |
| **Dependencies** | Required Terraform and provider versions |
| **Resources** | Every resource type it can create |
| **Examples** | Full working configurations in the repo's `examples/` folder |
| **Versions** | Release history, with links to the changelog |

### Judging quality and risk

| Check | Good sign |
|---|---|
| **Maintainer** | A verified partner, a well-known organisation (`terraform-aws-modules`, cloud vendors), or a team you can contact |
| **Activity** | Recent releases; issues answered; compatible with the current provider major |
| **Downloads/stars** | Widely used = widely tested (not a guarantee) |
| **Versioning** | SemVer tags, a changelog, upgrade guides for majors |
| **Tests and examples** | CI, `examples/`, `tests/` |
| **Scope** | Does it do what you need without dozens of resources you don't want? |
| **Security** | Secure defaults; no `local-exec` provisioners calling scripts; no surprising external data sources; scan it (topic 53) |
| **Licence** | Apache-2.0/MIT/MPL fine for most companies; check yours |

### Use, wrap or fork?

| Option | When | Cost |
|---|---|---|
| **Use directly** | The module fits; you pin and upgrade it | You follow upstream changes |
| **Wrap** (your own module that calls it with company defaults) | You want standards (naming, tags, flow logs, no NAT in dev) enforced for every team | A thin layer to maintain |
| **Fork** | You need changes upstream won't accept, or must control every line (regulated environments) | You own all future maintenance and security fixes |
| **Write your own** | Small scope; you'd use 10% of a huge module | Your time |

A common enterprise pattern is **wrap**: `acme/network/aws` calls `terraform-aws-modules/vpc/aws` with opinions baked in, so product teams see 5 inputs instead of 200.

---

## Why it exists

A production-grade VPC module handles subnets of four tiers, NAT per AZ, IPv6, flow logs, VPC endpoints, DHCP options, NACLs and dozens of edge cases. Writing and maintaining that yourself costs weeks. Popular public modules are battle-tested by thousands of users. The trade-off is **trust and control**, which is what the checks above manage.

---

## Analogy

**Flat-pack furniture from a known brand.** You get a tested design and clear instructions, quickly. You still read the reviews (activity, maintainers), check the dimensions fit your room (scope, inputs), keep the receipt and model number (pinned version), and look at what's in the box before assembling (read the plan). If you need something special, you add your own handles (wrap) or, rarely, rebuild it yourself (fork).

---

## How it works inside

A public module is just a Git repository published to the registry:

```
github.com/terraform-aws-modules/terraform-aws-vpc
   tag v6.5.1 ──▶ registry.terraform.io lists version 6.5.1
                     │ terraform init (source + version)
                     ▼
            .terraform/modules/vpc/   ← the code at that tag
                     │
            plan: its resources appear as module.vpc.aws_vpc.this[0], ...
```

Big modules use `count`/`for_each` heavily (to make features optional), so addresses look like `module.vpc.aws_subnet.public[0]`.

---

## Code / commands

`~/tf-lab/46-public-vpc/` (with `versions.tf` and the AWS provider as in topic 31):

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  azs  = slice(data.aws_availability_zones.available.names, 0, 2)
  cidr = "10.10.0.0/16"
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 6.0"

  name = "tf-course-public-module"
  cidr = local.cidr
  azs  = local.azs

  public_subnets  = [for i, _ in local.azs : cidrsubnet(local.cidr, 8, i)]
  private_subnets = [for i, _ in local.azs : cidrsubnet(local.cidr, 8, i + 10)]

  enable_nat_gateway   = false
  enable_dns_hostnames = true

  # S3 gateway endpoint (free) via the module's sub-module would go here; see its examples.

  tags = {
    stack = "lab-46"
  }
}

output "vpc_id" {
  value = module.vpc.vpc_id
}

output "public_subnets" {
  value = module.vpc.public_subnets
}

output "private_subnets" {
  value = module.vpc.private_subnets
}
```

- `[for i, _ in local.azs : ...]`: `_` = "I don't need this value", a common naming convention.

```
$ terraform init
Initializing modules...
Downloading registry.terraform.io/terraform-aws-modules/vpc/aws 6.5.1 for vpc...
- vpc in .terraform/modules/vpc

$ terraform plan
  # module.vpc.aws_default_network_acl.this[0] will be created
  # module.vpc.aws_default_route_table.default[0] will be created
  # module.vpc.aws_default_security_group.this[0] will be created
  # module.vpc.aws_internet_gateway.this[0] will be created
  # module.vpc.aws_route.public_internet_gateway[0] will be created
  # module.vpc.aws_route_table.private[0] will be created
  # module.vpc.aws_route_table.private[1] will be created
  # module.vpc.aws_route_table.public[0] will be created
  # module.vpc.aws_route_table_association.private[0] will be created
  ...
  # module.vpc.aws_subnet.public[1] will be created
  # module.vpc.aws_vpc.this[0] will be created

Plan: 17 to add, 0 to change, 0 to destroy.
```

(Exact count depends on the module version.) Note what it does that topic 32 didn't:

- **`aws_default_security_group`, `aws_default_network_acl`, `aws_default_route_table`**: the module takes over the VPC's **default** objects and, for the security group, strips their rules, a secure default.
- **One private route table per AZ**: ready for one NAT gateway per AZ later.

This is exactly why you **read the plan** of a public module: it shows you its opinions.

### Wrapping it

```hcl
# modules/acme-network/main.tf: the company's opinionated wrapper
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "6.5.1"                         # wrappers often pin exactly; callers pin the wrapper

  name = var.name
  cidr = var.cidr
  azs  = var.azs

  public_subnets  = [for i, _ in var.azs : cidrsubnet(var.cidr, 8, i)]
  private_subnets = [for i, _ in var.azs : cidrsubnet(var.cidr, 8, i + 10)]

  enable_nat_gateway     = var.environment == "prod"
  one_nat_gateway_per_az = var.environment == "prod"
  single_nat_gateway     = false

  enable_flow_log                      = var.environment == "prod"
  create_flow_log_cloudwatch_log_group = var.environment == "prod"
  create_flow_log_cloudwatch_iam_role  = var.environment == "prod"

  tags = merge(var.tags, { network_module = "acme-network" })
}
```

Teams now call `acme-network` with `name`, `cidr`, `azs`, `environment`, `tags`, and get the company's rules for free.

### Migrating your own code to a public module

If topic 32's hand-written network were in production, switching to the module means **`moved` blocks** for every resource (`aws_vpc.main` → `module.vpc.aws_vpc.this[0]`, and so on) so nothing is destroyed. Plan until `0 to add, 0 to destroy`; expect some in-place updates where the module's settings differ (tags, defaults).

---

## Hands-on lab

1. Apply the public VPC module (free). Compare its resources with your topic 32 code.
2. Open `.terraform/modules/vpc/main.tf` and find the `aws_vpc` resource and its `count`.
3. Read the module's **Inputs** on the registry; add `enable_ipv6 = true` and plan to see what changes.
4. Check the module's GitHub: last release date, open issues, `examples/`, the `CHANGELOG.md`, and the required provider version.
5. Write the `acme-network` wrapper and call it for `dev`.
6. Run a scanner on the downloaded module (`trivy config .terraform/modules/vpc`, topic 53) and read the findings.
7. `terraform destroy`.

---

## Where you'll use it

- **VPC, EKS, RDS, ALB, IAM, S3** modules from `terraform-aws-modules` appear in a very large share of AWS Terraform codebases.
- **Cloud vendors' modules** (AWS's own "AWS Integration & Automation", Azure Verified Modules, Google's Cloud Foundation Toolkit).
- **Platform teams** wrap them into internal modules.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `Unsupported argument` after a module upgrade | Input renamed/removed in a new major. | Read the upgrade guide (`UPGRADE-x.0.md`), update the call. |
| Plan replaces many resources after an upgrade | Internal addresses or defaults changed. | Check if the module ships `moved` blocks; read the changelog; pin back if needed. |
| Provider version conflict on `init` | Module requires a newer/older provider major. | Align versions; upgrade module and provider together (topic 62). |
| Your NAT bill appears | A module default or input turned NAT on. | Read the plan; set `enable_nat_gateway` explicitly. |
| Module does far more than needed | Huge feature surface. | Write a small module or use a more focused one. |
| Supply-chain concern (unknown maintainer) | Untrusted code with your credentials. | Fork into your org, review, pin SHAs; or write your own. |

---

## Quick recap

- Public modules (especially `terraform-aws-modules`) save weeks of work; pin the version and read the plan.
- Read **Inputs, Outputs, Examples, Dependencies** on the registry page.
- Judge maintainers, activity, versioning, tests, scope, security and licence before adopting.
- Use directly, **wrap** with company defaults, or fork if you must own every line.
- Migrating hand-written code to a module needs `moved` blocks.

### Interview questions

1. **Q: How do you decide whether to use a public module?**
   A: Check who maintains it, how active and widely used it is, its versioning and changelog, tests and examples, provider compatibility, scope, security defaults and licence; then pin a version and review its plan before using it in production.
2. **Q: What's a wrapper module and why use one?**
   A: An internal module that calls a public module with your organisation's defaults and a smaller interface. It enforces standards (tags, logging, naming, environment rules) while reusing the community module's tested implementation.
3. **Q: How would you replace hand-written VPC code with `terraform-aws-modules/vpc` in production?**
   A: Add the module with matching settings and `moved` blocks mapping each existing resource to its address inside the module, then iterate on the plan until nothing is destroyed or created, only moved (and harmless in-place updates).
