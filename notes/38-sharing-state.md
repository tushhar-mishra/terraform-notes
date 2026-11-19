# 38 · Sharing State Between Configurations

> **Level:** ★★ intermediate
>
> **You will learn:** why real systems are split into **several configurations** (several states), and the four ways one configuration can use values from another: the **`terraform_remote_state`** data source, **provider data sources** looking things up by tag, a **parameter store** (AWS SSM Parameter Store) as a published contract, and platform features (HCP Terraform, Terragrunt). You'll learn the trade-offs (coupling, security, ordering), and build a **producer** (your network) and a **consumer** (an app) that reads its outputs two ways.
>
> **You need:** topics 15 (outputs), 19 (data sources), 32 (network), 35 (S3 backend), 36.
>
> **Part 5 · State in depth** · Previous: [37 · Drift](37-drift.md) · Next: [39 · Importing existing infrastructure](39-import.md) · [Back to INDEX](../INDEX.md)
>
> 💰 **Cost:** free (VPC pieces, security groups and SSM standard parameters are free).

---

## ⚡ Core: the 10-minute version

**The idea.** Big estates are split: a **network** configuration, a **database** configuration, an **app** configuration, each with its own state. The app needs the network's VPC ID. Options:

```hcl
# 1. Read the other state's outputs directly
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "tf-course-state-123456789012"
    key    = "lab/network/terraform.tfstate"
    region = "ap-south-1"
  }
}
vpc_id = data.terraform_remote_state.network.outputs.vpc_id

# 2. Look the thing up in AWS by a tag (no state access needed)
data "aws_vpc" "main" {
  tags = { Name = "tf-course-vpc" }
}

# 3. Read a value the producer published in SSM Parameter Store
data "aws_ssm_parameter" "vpc_id" {
  name = "/tf-course/network/vpc_id"
}
```

**Picture it.** Two departments in one company. The app team needs the network team's room numbers. They can read the network team's **whole private notebook** (remote state), **walk around and read the door signs** (lookup by tag), or check the **company directory** the network team publishes (parameter store).

**The one thing to know.** `terraform_remote_state` is easy but gives the consumer **read access to the entire producer state, secrets included**, and ties it to the producer's backend layout. Prefer lookups or a published parameter for anything crossing team boundaries.

**Try it now:** put your topic 32 network into the S3 backend, then build the small consumer below that creates a security group in that VPC.

**You've got it if you can:** list the options and say which you'd use between two teams, and why.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Why split into several configurations

| Reason | Example |
|---|---|
| **Blast radius** | A bad app change can't destroy the VPC if the VPC is in another state. |
| **Speed** | A 2,000-resource state takes minutes to plan; ten 200-resource states plan in seconds each. |
| **Ownership and permissions** | The network team owns the network state; app teams can't change it. |
| **Different change rates** | The VPC changes twice a year; the app changes daily. |
| **Avoiding provider chicken-and-egg** | The EKS cluster in one configuration, the Kubernetes resources in another (topic 28). |

A **producer** configuration exposes values; **consumers** use them. Values flow one way, and the producer must be applied first.

### The four options

| Option | How the consumer gets the value | Pros | Cons |
|---|---|---|---|
| **`terraform_remote_state`** | Reads the producer's state file and takes its **root outputs** | Simple; exact values; no extra resources | Consumer needs **read access to the whole state** (incl. secrets); coupled to backend/bucket/key; outputs become an implicit API |
| **Provider data sources** | Queries the cloud API: `aws_vpc` by tag, `aws_subnets` by filter | No state access; works with anything (even non-Terraform resources) | Depends on naming/tagging conventions; lookup may match 0 or 2 objects |
| **Parameter store** (AWS SSM Parameter Store; or HashiCorp Consul/Vault key-value stores) | Producer writes `aws_ssm_parameter`; consumer reads `data.aws_ssm_parameter` | Explicit, versioned **contract**; fine-grained IAM per parameter; readable by non-Terraform tools too | One more resource per value; strings only (encode lists as JSON) |
| **Platform features** | HCP Terraform `tfe_outputs` data source (topic 50), Terragrunt `dependency` blocks (topic 49) | Integrated, handles ordering | Tied to that platform |

### `terraform_remote_state` details

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"                          # any backend type
  config = {                              # the same settings as that backend block
    bucket = "tf-course-state-123456789012"
    key    = "lab/network/terraform.tfstate"
    region = "ap-south-1"
  }
  defaults = {                            # optional: used if the producer hasn't defined an output yet
    extra_tag = "none"
  }
}

# Only ROOT-module OUTPUTS are available:
data.terraform_remote_state.network.outputs.vpc_id
```

- It comes from the built-in `terraform` provider; nothing to install.
- It reads **outputs only**, not resources. A value must be an `output` in the producer's root module.
- It's read-only: it never locks or changes the other state.
- IAM: `s3:GetObject` on the producer's key. That object contains **everything**, which is the security concern.

### Outputs are an API

Whatever you share, the producer's outputs/parameters become a **contract**:

- Renaming or removing an output breaks consumers on their next plan.
- Changing a value's **meaning** (e.g. subnet IDs as a list vs a map) breaks them too.
- Treat them like an API: add new ones freely, deprecate old ones slowly, and document them.

---

## Why it exists

Splitting state solves blast-radius, speed and ownership problems, but creates a new one: configurations need each other's values. Hard-coding IDs (`vpc_id = "vpc-0a1b..."`) breaks every time something is rebuilt, and differs per environment. These mechanisms keep values **flowing automatically** while keeping states separate.

---

## Analogy

**Two departments in one company.**

- **`terraform_remote_state`** = the app team borrows the network team's **entire notebook** to copy two room numbers. Easy, but they can also read the salary page (secrets), and if the network team moves the notebook to another shelf (backend key), the app team can't find it.
- **Lookup by tag** = walk the corridor and read the **door signs** ("Network Room: Main VPC"). No notebook needed, but if two doors have the same sign, or someone changes a sign, you're confused.
- **Parameter store** = the **company directory** the network team publishes on purpose: "Main VPC: room 4.12". Only those entries are visible, and everyone (even non-Terraform tools) can read it.

---

## How it works inside

```
PRODUCER (network config, state key lab/network/...)             apply FIRST
  outputs: vpc_id, public_subnet_ids, web_sg_id  ──► written into its state
  aws_ssm_parameter "/tf-course/network/vpc_id"   ──► written into SSM

CONSUMER (app config, state key lab/app/...)                     apply SECOND
  plan:
   data.terraform_remote_state.network → GET s3://.../lab/network/terraform.tfstate
                                         → parse JSON → .outputs
   data.aws_ssm_parameter.vpc_id       → ssm:GetParameter
   data.aws_vpc.main (tags)            → ec2:DescribeVpcs with a tag filter
   resources use those values like any data source (read at plan time)
```

No locks are shared and no apply order is enforced by Terraform: **you** (or your pipeline / Terragrunt) must apply the producer first and re-apply consumers when the producer's outputs change.

---

## Code / commands

### Step 1: put the producer in S3 and publish values

In `~/tf-lab/32-network` (the topic 32 network), add `backend.tf`:

```hcl
terraform {
  backend "s3" {
    bucket       = "tf-course-state-123456789012"
    key          = "lab/network/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

Add `publish.tf`:

```hcl
# Published contract for other configurations
resource "aws_ssm_parameter" "vpc_id" {
  name  = "/${var.name}/network/vpc_id"
  type  = "String"
  value = aws_vpc.main.id
}

resource "aws_ssm_parameter" "public_subnet_ids" {
  name  = "/${var.name}/network/public_subnet_ids"
  type  = "String"
  value = jsonencode([for s in aws_subnet.public : s.id])    # lists as JSON
}
```

```bash
cd ~/tf-lab/32-network
terraform init -migrate-state        # local → S3 (topic 35)
terraform apply                      # creates the two parameters
```

Its existing `outputs.tf` (topic 32) already exposes `vpc_id`, `public_subnet_ids`, `web_sg_id`.

### Step 2: the consumer

`~/tf-lab/38-app/`:

```hcl
# versions.tf
terraform {
  required_version = ">= 1.14"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.0" }
  }
  backend "s3" {
    bucket       = "tf-course-state-123456789012"
    key          = "lab/app/terraform.tfstate"          # its OWN state
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}

provider "aws" {
  region = "ap-south-1"
  default_tags {
    tags = { project = "tf-course", stack = "app", managed_by = "terraform" }
  }
}
```

```hcl
# inputs.tf: three ways to get the same VPC

# A: the producer's state outputs
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "tf-course-state-123456789012"
    key    = "lab/network/terraform.tfstate"
    region = "ap-south-1"
  }
}

# B: the published parameters
data "aws_ssm_parameter" "vpc_id" {
  name = "/tf-course/network/vpc_id"
}

data "aws_ssm_parameter" "public_subnet_ids" {
  name = "/tf-course/network/public_subnet_ids"
}

# C: a lookup by tag
data "aws_vpc" "by_tag" {
  tags = { Name = "tf-course-vpc" }
}

locals {
  vpc_id_from_state = data.terraform_remote_state.network.outputs.vpc_id
  vpc_id_from_ssm   = nonsensitive(data.aws_ssm_parameter.vpc_id.value)
  public_subnets    = jsondecode(nonsensitive(data.aws_ssm_parameter.public_subnet_ids.value))
}
```

- `aws_ssm_parameter.value` is marked **sensitive** by the provider (parameters can hold secrets). A VPC ID isn't secret, so `nonsensitive()` is fine here (topic 18).

```hcl
# main.tf
resource "aws_security_group" "app" {
  name        = "tf-course-app"
  description = "App tier"
  vpc_id      = local.vpc_id_from_ssm
}

check "all_sources_agree" {
  assert {
    condition = (
      local.vpc_id_from_state == local.vpc_id_from_ssm &&
      local.vpc_id_from_ssm == data.aws_vpc.by_tag.id
    )
    error_message = "The three ways of finding the VPC disagree."
  }
}

output "vpc_id" {
  value = local.vpc_id_from_ssm
}

output "public_subnets" {
  value = local.public_subnets
}
```

```bash
cd ~/tf-lab/38-app
terraform init && terraform apply
```

```
data.terraform_remote_state.network: Reading...
data.aws_ssm_parameter.vpc_id: Reading...
data.aws_vpc.by_tag: Reading...
data.aws_ssm_parameter.public_subnet_ids: Reading...
data.terraform_remote_state.network: Read complete after 1s
data.aws_ssm_parameter.vpc_id: Read complete after 0s [id=/tf-course/network/vpc_id]
...
  # aws_security_group.app will be created
  + resource "aws_security_group" "app" {
      + vpc_id = "vpc-0a1b2c3d4e5f6a7b8"
      ...
Plan: 1 to add, 0 to change, 0 to destroy.
...
Outputs:

public_subnets = [
  "subnet-0a3b...",
  "subnet-0b4c...",
]
vpc_id = "vpc-0a1b2c3d4e5f6a7b8"
```

### See the security problem

```bash
terraform console
> keys(data.terraform_remote_state.network.outputs)
[
  "private_subnet_ids",
  "public_subnet_ids",
  "vpc_id",
  "web_sg_id",
]
> exit
aws s3 cp s3://tf-course-state-123456789012/lab/network/terraform.tfstate - | jq '.resources | length'
```

Terraform only exposes outputs, but the IAM permission that allows it lets the consumer download the **whole** file (every attribute of every resource). In the network state that's harmless; in a database state it includes the master password.

### Breaking the contract

In the producer, rename output `vpc_id` → `main_vpc_id`, apply. In the consumer:

```
╷
│ Error: Unsupported attribute
│
│   on inputs.tf line 25, in locals:
│   25:   vpc_id_from_state = data.terraform_remote_state.network.outputs.vpc_id
│     ├────────────────
│     │ data.terraform_remote_state.network.outputs is object with 4 attributes
│
│ This object does not have an attribute named "vpc_id".
╵
```

A producer change broke a consumer that nobody touched. Rename it back.

---

## Hands-on lab

1. Migrate the topic 32 network to S3 and publish the SSM parameters.
2. Build the consumer; apply; confirm the `check` passes.
3. List the producer state's outputs from the consumer's `terraform console`, then download the whole state with the CLI, and think about who should have that permission.
4. Break the contract (rename an output) and read the consumer's error. Revert.
5. Change the consumer to use **only** the SSM parameters (delete the remote-state data source) and apply: no state access needed any more.
6. Clean up the consumer: `terraform destroy` in `38-app`. Keep the network for topic 39.

---

## Where you'll use it

- **Layered architectures**: network → shared services (DNS, KMS) → data (RDS) → compute (EKS/ECS) → apps, each a separate state.
- **Platform teams** publishing VPC IDs, subnet IDs, cluster names and KMS key ARNs in SSM for product teams.
- **Cross-account**: SSM parameters can be shared via AWS RAM (Resource Access Manager, for sharing resources between accounts), or the consumer assumes a read-only role in the producer account.
- **Terragrunt** stacks wiring `dependency` outputs between units (topic 49).

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `Unsupported attribute ... does not have an attribute named "x"` on `.outputs` | The producer has no such root output (renamed, never added, or not applied yet). | Add/restore the output in the producer and apply it first. Use `defaults` during transitions. |
| `Unable to find remote state ... No stored state was found for the given workspace` | Wrong bucket/key/region, or the producer was never applied. | Check `config`; apply the producer. |
| `AccessDenied` reading the remote state | The consumer's identity can't read the producer's key. | Grant `s3:GetObject` on that key (and KMS decrypt), or switch to SSM/lookups. |
| `ParameterNotFound` | Parameter name/region/account wrong, or not published yet. | Check the name; apply the producer. |
| `multiple VPCs matched` / `no matching VPC found` | Tag lookup ambiguous or wrong. | Make tags unique and conventional; add more filters. |
| Consumer still uses old values after the producer changed | Consumers only see changes when **they** are planned/applied. | Re-apply consumers after producer changes (pipelines can trigger downstream runs). |
| Producer's `destroy` fails with `DependencyViolation` | A consumer's resources (e.g. a security group) still live in the producer's VPC. | Destroy consumers first: reverse order of creation. |

---

## Quick recap

- Split estates into several states for blast radius, speed and ownership; values flow **producer → consumer**.
- Options: `terraform_remote_state` (simple; exposes the whole state; tight coupling), **provider lookups** (by tag), **parameter store** (explicit contract, fine-grained access), platform features.
- Only **root outputs** are visible through remote state; outputs/parameters are an **API**: don't break them casually.
- Apply producers before consumers; destroy in reverse.

### Interview questions

1. **Q: How do you share outputs between Terraform configurations?**
   A: With the `terraform_remote_state` data source reading the producer's root outputs, provider data sources looking resources up by tags, or a published parameter store like SSM, which the producer writes and consumers read. Platforms like HCP Terraform and Terragrunt add their own mechanisms.
2. **Q: What's the downside of `terraform_remote_state`?**
   A: The consumer needs read access to the producer's entire state file, including any secrets, and is coupled to its backend location and output names. Across team boundaries, a published parameter or a lookup is safer.
3. **Q: Why split infrastructure into multiple states at all?**
   A: Smaller blast radius, faster plans, separate ownership and permissions, different change rates, and avoiding provider bootstrap problems like configuring the Kubernetes provider from a cluster in the same state.
