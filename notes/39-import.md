# 39 · Importing Existing Infrastructure

> **Level:** ★★ intermediate
>
> **You will learn:** how to bring resources that were created **outside** Terraform (by hand, by scripts, by another tool, or by a Terraform run whose state was lost) **under management**, using **`import` blocks** (Terraform 1.5+): import IDs, the `will be imported` plan, **`-generate-config-out`** to draft the configuration for you, the loop "plan → adjust config → plan" until **No changes**, bulk imports with **`for_each`** in import blocks (1.7+), **identity-based** import (1.12+), import blocks inside **modules** (1.16+), and the ⚠️ legacy `terraform import` command.
>
> **You need:** topics 26 (Docker provider), 31 (S3), 34 (config/state/reality), 35.
>
> **Part 5 · State in depth** · Previous: [38 · Sharing state between configurations](38-sharing-state.md) · Next: [40 · Refactoring: `moved` and `removed` blocks](40-moved-and-removed.md) · [Back to INDEX](../INDEX.md)
>
> 💰 **Cost:** Docker part free; AWS part: one empty S3 bucket (free).

---

## ⚡ Core: the 10-minute version

**The idea.** **Import** = "this real object already exists; record it in state at this address". You write an `import` block and a matching `resource` block, then plan and apply like any change:

```hcl
import {
  to = aws_s3_bucket.legacy           # the address in your code
  id = "acme-legacy-reports"          # the real object's import ID (from the provider docs)
}

resource "aws_s3_bucket" "legacy" {
  bucket = "acme-legacy-reports"
}
```

```bash
terraform plan        # "aws_s3_bucket.legacy will be imported"
terraform apply       # writes it into state; changes nothing in AWS
```

Don't want to write the resource block yourself? Let Terraform draft it:

```bash
terraform plan -generate-config-out=generated.tf
```

**Picture it.** Adopting a stray cat: it already exists and has a life; import is the **adoption paperwork** that makes it officially yours. The cat doesn't change; your records do.

**The one thing to know.** Import only updates **state**. Your goal is a plan that says **`1 to import, 0 to add, 0 to change, 0 to destroy`**: if it also says "change", your config doesn't match reality yet, and applying would modify the real object.

**Try it now:** `docker run -d --name legacy nginx:1.29`, then import it into a Terraform project with `-generate-config-out`.

**You've got it if you can:** find a resource's import ID format, and explain why `import` blocks are better than the old `terraform import` command.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### The `import` block

```hcl
import {
  to       = <resource address>          # where it goes in state (must match a resource block,
                                         #   or be generated with -generate-config-out)
  id       = "<import ID>"               # how the provider finds the real object
  provider = aws.prod                    # optional: which provider configuration (aliases)
}
```

- Lives in any `.tf` file of the **root module** (since 1.16 also inside child modules).
- Is processed during **plan**: the plan shows `will be imported`. Nothing is written until **apply**.
- After a successful apply, the block has done its job. You can delete it (leaving it is harmless: once the address is in state, it's a no-op) or keep it as documentation for a while.

### Import IDs

Each resource type defines what its `id` looks like. Find it at the bottom of the resource's Registry page, under **"Import"**:

| Resource | Import ID | Example |
|---|---|---|
| `aws_s3_bucket` | Bucket name | `acme-legacy-reports` |
| `aws_instance` | Instance ID | `i-0abc123def4567890` |
| `aws_vpc` | VPC ID | `vpc-0a1b2c3d` |
| `aws_iam_role` | Role **name** | `app-role` |
| `aws_iam_role_policy_attachment` | `role-name/policy-arn` | `app-role/arn:aws:iam::aws:policy/ReadOnlyAccess` |
| `aws_route53_record` | `ZONEID_name_type` | `Z0123456789_www.example.com_A` |
| `docker_container` | Full container ID | `4f1e...` (64 hex chars) |
| `kubernetes_deployment_v1` | `namespace/name` | `shop/web` |

The import ID is **not always** the resource's `id` attribute. Always check the docs.

### Identity-based import (1.12+)

Some providers now define a structured **resource identity**. Instead of a string ID you can write:

```hcl
import {
  to = aws_s3_bucket.legacy
  identity = {
    bucket = "acme-legacy-reports"
    region = "ap-south-1"
  }
}
```

Clearer than packing values into one string. Use it when the resource's docs list an identity schema.

### Bulk import with `for_each` (1.7+)

```hcl
locals {
  legacy_buckets = {
    reports = "acme-legacy-reports"
    logs    = "acme-legacy-logs"
    exports = "acme-legacy-exports"
  }
}

import {
  for_each = local.legacy_buckets
  to       = aws_s3_bucket.legacy[each.key]
  id       = each.value
}

resource "aws_s3_bucket" "legacy" {
  for_each = local.legacy_buckets
  bucket   = each.value
}
```

### `-generate-config-out`

```bash
terraform plan -generate-config-out=generated.tf
```

For every `import` block whose `to` address has **no** resource block, Terraform asks the provider for the object's current attributes and **writes a resource block** into `generated.tf` (the file must not exist yet). It's a **draft**:

- It contains **every** argument, including defaults and sometimes conflicting ones (e.g. both `X` and `X_prefix`). You delete what you don't need.
- It uses literal values. Replace them with references and variables (`vpc_id = aws_vpc.main.id`).
- Secrets aren't readable from APIs, so the generator may leave them `null`; you fill them in.

### Discovering what to import: `terraform query` (1.14+)

Terraform 1.14 added **list resources** and the `terraform query` command: you write `list` blocks in `*.tfquery.hcl` files ("list all S3 buckets with tag X"), and Terraform lists matching real objects and can **generate configuration for importing them** (`terraform query -generate-config-out=generated.tf`), which you review like any `-generate-config-out` draft. Provider support is growing; topic 67 shows it. Until your provider supports it, use the CLI/console to find IDs.

### ⚠️ The legacy `terraform import` command

```bash
terraform import aws_s3_bucket.legacy acme-legacy-reports
```

Still exists, but:

| `terraform import` (command) | `import` block |
|---|---|
| Writes to state **immediately**: no plan, no review | Shows in the **plan**, applied with everything else |
| One resource per command | Many, with `for_each` |
| Can't generate config | `-generate-config-out` |
| Hard to do in CI / code review | It's just code in a PR |
| You must write the resource block first | Can be generated |

Use import blocks. You'll still see the command in old docs and runbooks.

---

## Why it exists

Real estates are never 100% Terraform from day one:

- **Brownfield**: years of console-created resources you now want under code.
- **Lost state**: a state was deleted; the resources still run (topic 42).
- **Migrations**: moving resources from CloudFormation, another Terraform repo or another tool.
- **Drift of the "unmanaged" kind** (topic 37): something created by hand that should be managed.

Without import, the only option would be to delete and recreate, which is unacceptable for databases, DNS zones, buckets with data, or anything in production.

---

## Analogy

**Adopting a stray cat.**

- The cat (real object) already exists, with its own habits (attributes).
- The **adoption form** (import block) says: "the cat at this address (ID) is now my cat, recorded under the name `legacy`".
- The **vet's record** (state) is updated. The cat doesn't change.
- Your **description of the cat** (resource block) must match the real cat. If you write "white cat" and it's black, your next visit (apply) would try to dye it white (change the real object). So you fix the description first, until there are **no differences**.
- `-generate-config-out` = the vet writes the description for you by looking at the cat. You tidy it up.

---

## How it works inside

```
terraform plan
  for each import block:
    1. provider.ImportResourceState(type, id)   → a minimal object (often just the ID)
    2. provider.ReadResource(that object)       → full current attributes from the API
       (object not found → "Cannot import non-existent remote object")
    3. if no resource block exists at `to`:
         with -generate-config-out → provider builds a config from the attributes → write file
         without it               → error: resource block required
    4. the imported object becomes the "prior state" for that address
    5. normal planning: config vs imported state → no-op, update, or even replace!
  plan output: "will be imported" (+ any update/replace on top)
terraform apply
  writes the imported objects (and any other changes) into state
```

Step 5 is why the goal is **0 to change**: after import, Terraform treats the object exactly like any managed resource, so any difference between your config and reality becomes a change.

---

## Code / commands

### Part A: Docker (free)

Create the "legacy" container by hand:

```bash
docker run -d --name legacy-web -p 8090:80 --restart unless-stopped nginx:1.29
docker inspect -f '{{.Id}}' legacy-web
```

```
4f1e2d3c...(64 hex characters)
```

`~/tf-lab/39-import-docker/main.tf`:

```hcl
terraform {
  required_providers {
    docker = { source = "kreuzwerker/docker", version = "~> 4.7" }
  }
}

provider "docker" {}

import {
  to = docker_container.legacy
  id = "4f1e2d3c..."                 # paste the full ID from docker inspect
}
```

No resource block yet. Generate it:

```bash
terraform init
terraform plan -generate-config-out=generated.tf
```

```
docker_container.legacy: Preparing import... [id=4f1e2d3c...]
docker_container.legacy: Refreshing state... [id=4f1e2d3c...]

Terraform will perform the following actions:

  # docker_container.legacy will be imported
  # (config will be generated)
    resource "docker_container" "legacy" {
        id       = "4f1e2d3c..."
        image    = "sha256:4e1b..."
        name     = "legacy-web"
        restart  = "unless-stopped"
        ...
        ports {
            external = 8090
            internal = 80
            ip       = "0.0.0.0"
            protocol = "tcp"
        }
    }

Plan: 1 to import, 0 to add, 0 to change, 0 to destroy.

─────────────────────────────────────────────────────────────────────────────

Terraform has generated configuration and written it to generated.tf. Please
review the configuration and edit it as necessary before adding it to version
control.
```

- `Preparing import... / Refreshing state...`: steps 1–2.
- `(config will be generated)`: step 3.
- No `+`/`~` symbols on the resource: just the imported object.

Look at `generated.tf`: it's long (dozens of attributes such as `cpu_shares`, `log_driver`, `shm_size`...). Tidy it to what you care about:

```hcl
resource "docker_container" "legacy" {
  name    = "legacy-web"
  image   = "sha256:4e1b..."      # better: docker_image.nginx.image_id (then plan again)
  restart = "unless-stopped"

  ports {
    internal = 80
    external = 8090
  }
}
```

Rename the file to `main.tf` content (or keep it as `legacy.tf`), run `terraform plan` again, and iterate until:

```
Plan: 1 to import, 0 to add, 0 to change, 0 to destroy.
```

Then `terraform apply`:

```
docker_container.legacy: Importing... [id=4f1e2d3c...]
docker_container.legacy: Import complete [id=4f1e2d3c...]

Apply complete! Resources: 1 imported, 0 added, 0 changed, 0 destroyed.
```

The container never restarted (`docker ps` shows the same uptime). Now delete the `import` block and run `terraform plan`: `No changes`. It's managed.

### When the plan says "change" or "replace" after import

If your tidied config differs from reality, the plan says so. For example, writing `restart = "always"`:

```
  # docker_container.legacy will be updated in-place
  # (imported from "4f1e2d3c...")
  ~ resource "docker_container" "legacy" {
      ~ restart = "unless-stopped" -> "always"
        ...
    }

Plan: 1 to import, 0 to add, 1 to change, 0 to destroy.
```

Decide: make the code match (`unless-stopped`), or accept that apply will change the real container. For arguments that force replacement, the plan would show `must be replaced`, which would **recreate** the thing you meant to adopt. Always reach `0 to change` (or a change you consciously want) before applying.

### Part B: AWS bucket with a for_each import

Create two buckets "by hand":

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)
for b in reports logs; do aws s3 mb "s3://tf-course-legacy-$b-$ACCT" --region ap-south-1; done
```

`~/tf-lab/39-import-aws/main.tf` (with the usual `versions.tf`/provider):

```hcl
data "aws_caller_identity" "current" {}

locals {
  legacy = toset(["reports", "logs"])
}

import {
  for_each = local.legacy
  to       = aws_s3_bucket.legacy[each.key]
  id       = "tf-course-legacy-${each.key}-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket" "legacy" {
  for_each = local.legacy
  bucket   = "tf-course-legacy-${each.key}-${data.aws_caller_identity.current.account_id}"
}

# Now that they're managed, add what was missing: versioning
resource "aws_s3_bucket_versioning" "legacy" {
  for_each = aws_s3_bucket.legacy
  bucket   = each.value.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

```
  # aws_s3_bucket.legacy["logs"] will be imported
    resource "aws_s3_bucket" "legacy" { ... }

  # aws_s3_bucket.legacy["reports"] will be imported
    resource "aws_s3_bucket" "legacy" { ... }

  # aws_s3_bucket_versioning.legacy["logs"] will be created
  + resource "aws_s3_bucket_versioning" "legacy" { ... }

  # aws_s3_bucket_versioning.legacy["reports"] will be created
  + resource "aws_s3_bucket_versioning" "legacy" { ... }

Plan: 2 to import, 2 to add, 0 to change, 0 to destroy.
```

A realistic adoption: import what exists, and in the same reviewed plan, bring it up to standard. Apply, then clean up with `terraform destroy` (the buckets are empty, so no `force_destroy` needed).

### Errors you'll meet

```
│ Error: Cannot import non-existent remote object
│
│ While attempting to import an existing object to "aws_s3_bucket.legacy["logs"]",
│ the provider detected that no object exists with the given id. Only
│ pre-existing objects can be imported; check that the id is correct and that it
│ is associated with the provider's configured region or endpoint, or use
│ "terraform apply" to create a new remote object for this resource.
```

Wrong ID, wrong region, or wrong account.

```
│ Error: Resource already managed by Terraform
│
│ Terraform is already managing a remote object for docker_container.legacy.
│ To import to this address you must first remove the existing object from the state.
```

That address already has an object in state (only with the legacy command; import blocks treat an already-imported address as done).

---

## Hands-on lab

1. Do Part A: create a container with `docker run`, import it with generated config, tidy it, reach `0 to change`, apply. Check the container didn't restart.
2. Introduce a difference on purpose (`restart = "always"`, then a port change) and read the plans: update vs replace.
3. Do Part B with `for_each` and add versioning in the same apply.
4. Try a wrong ID and read `Cannot import non-existent remote object`.
5. Use the legacy command once on a throwaway resource to feel the difference (`terraform import ...`), then never again.
6. Clean up: `terraform destroy` in both labs.

---

## Where you'll use it

- **Brownfield adoption** projects: bring a console-built account under Terraform, resource type by resource type.
- **State recovery**: rebuild a lost state by importing every resource (topic 42).
- **Moving between configurations**: `removed` (with `destroy = false`) in the old configuration + `import` in the new one (topic 40).
- **Migrating from CloudFormation**: retain the resources in CloudFormation (`DeletionPolicy: Retain`), delete the stack, import into Terraform.

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `Cannot import non-existent remote object` | ID/region/account wrong. | Check the import ID format in the docs; check provider region and credentials. |
| Plan shows `1 to import, 1 to change` | Config doesn't match reality. | Adjust config until `0 to change` (unless you want the change). |
| Plan shows `must be replaced` on the imported resource | A force-new argument differs. | **Stop.** Fix the config; applying would destroy the object you're adopting. |
| `Configuration for import target does not exist` | `import` without a resource block and without `-generate-config-out`. | Write the block or generate it. |
| `-generate-config-out`: `Target generated file already exists` | The file must be new. | Delete/rename it first. |
| Generated config fails validation (`conflicts with`) | The generator wrote mutually exclusive arguments. | Delete the redundant ones. |
| Secrets missing after import (`password = null` diffs) | APIs don't return secrets. | Provide them (ideally via write-only arguments, topic 51) or `ignore_changes`. |
| Imported a resource into **two** configurations | Two owners fight forever. | Remove it from one (`removed` block, topic 40). |

---

## Quick recap

- Import records an **existing** object in state; it never changes the object by itself.
- Use `import { to = ADDR  id = "..." }` blocks: planned, reviewed, bulk with `for_each`, identity-based (1.12+), in modules (1.16+).
- `-generate-config-out=file.tf` drafts the resource block; tidy it.
- Iterate until the plan says **`N to import, 0 to change`**. Watch for replacements.
- ⚠️ `terraform import` (command) writes state immediately with no plan: legacy.

### Interview questions

1. **Q: How do you bring an existing AWS resource under Terraform management?**
   A: Add an `import` block with the target address and the provider's import ID, write (or generate with `-generate-config-out`) a matching resource block, run plan until it shows only the import with no changes, then apply.
2. **Q: Why are import blocks better than `terraform import`?**
   A: They go through plan and review, can be applied in CI with other changes, support `for_each` for bulk imports, and can generate configuration. The command changes state immediately with no preview.
3. **Q: After importing, the plan wants to replace the resource. What do you do?**
   A: Don't apply. A force-new argument in the config differs from the real object. Fix the config to match reality (or deliberately accept the replacement if it's safe), and re-plan until it's a pure import.
