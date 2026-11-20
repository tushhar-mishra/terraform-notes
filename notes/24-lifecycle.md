# 24 · The `lifecycle` Meta-Argument

> **Level:** ★★ intermediate
>
> **You will learn:** **why** Terraform replaces resources (the five usual causes), and how to control replacements and changes with the `lifecycle` block: **`create_before_destroy`** (no-downtime replacement, and its name-collision trap), **`prevent_destroy`** (a safety catch), **`ignore_changes`** (let something outside Terraform own an attribute), **`replace_triggered_by`** (replace when something else changes), and **`destroy = false`** (1.16+). You'll also meet the built-in **`terraform_data`** resource.
>
> **You need:** topics 07 (`-/+`), 09 (plan symbols), 11 (unknown values), 13 (apply walk), 20 (destroy order), 22 (`for_each`).
>
> **Part 2 · Making configuration flexible** · Previous: [23 · `for` expressions and `dynamic` blocks](23-for-expressions-and-dynamic-blocks.md) · Next: [25 · Custom conditions and `check` blocks](25-conditions-and-checks.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Every resource can have a `lifecycle` block that changes **how** Terraform creates, updates and destroys it:

```hcl
resource "aws_db_instance" "main" {
  # ...
  lifecycle {
    prevent_destroy       = true                 # refuse any plan that destroys it
    create_before_destroy = false                # (default) destroy old, then create new
    ignore_changes        = [tags["LastBackup"]] # a backup tool updates this tag: ignore it
    replace_triggered_by  = [terraform_data.db_version]  # replace when this changes
  }
}
```

**Picture it.** House rules for a contractor: "never knock down the main wall" (`prevent_destroy`), "build the new kitchen before ripping out the old one" (`create_before_destroy`), "don't repaint the door, the tenant chooses its colour" (`ignore_changes`), "if the boiler is replaced, redo the plumbing" (`replace_triggered_by`).

**The one thing to know.** Before every apply, search the plan for **`must be replaced`** and **`forces replacement`**. For anything stateful (databases, disks, buckets), put `prevent_destroy = true` on it.

**Try it now:** add `prevent_destroy = true` to a lab resource, then run `terraform destroy`. Read the error.

**You've got it if you can:** name four lifecycle settings, give a real use for each, and explain the `create_before_destroy` name-collision trap.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Why resources get replaced (the five causes)

| # | Cause | Plan shows | Example |
|---|---|---|---|
| 1 | You changed an argument the API can't update in place | `# forces replacement` on that argument | AWS VM's `ami`, a bucket's `bucket` name, any `local_file` argument |
| 2 | A value feeding a force-new argument became **unknown** | `-> (known after apply) # forces replacement` | An upstream resource is being replaced, so its new ID is unknown (the cascade, topic 11) |
| 3 | The resource is **tainted** | `is tainted, so must be replaced` | A previous create failed halfway (topic 13) |
| 4 | You asked for it | `will be replaced, as requested` | `terraform apply -replace=ADDR` (topic 41) |
| 5 | A `replace_triggered_by` reference changed | `will be replaced due to changes in replace_triggered_by` | Below |

Also common: renaming a resource or a `for_each` key in code. That's not a replacement but a **destroy + create** of different addresses (fix with `moved`, topic 40).

### The `lifecycle` settings

| Setting | Type | Effect |
|---|---|---|
| `create_before_destroy` | bool | On replacement, **create the new object first**, then destroy the old one (`+/-` instead of `-/+`). |
| `prevent_destroy` | bool | Any plan that would destroy this object **fails**. |
| `ignore_changes` | list of attributes, or `all` | Differences in these attributes between config and reality are **ignored** in plans. |
| `replace_triggered_by` | list of references | Replace this resource when any referenced resource (or attribute) **changes**. (1.2+) |
| `destroy` | bool | (1.16+) `false` = when Terraform would destroy it, **forget** it instead: remove it from state, leave the real object. |
| `precondition` / `postcondition` | blocks | Checks before/after (topic 25). |

`lifecycle` settings must be **literal values**: no variables, because Terraform needs them before evaluating anything. (OpenTofu 1.12 allows a variable in `prevent_destroy`; Terraform doesn't.)

### `create_before_destroy`

Default replacement (`-/+`): destroy old → create new. In between, **the thing doesn't exist**: downtime. With `create_before_destroy = true` (`+/-`): create new → switch references → destroy old.

**The trap:** the new and old objects exist **at the same time**. If the resource has a **unique name** (an S3 bucket name, an IAM role name, a load balancer name, a DNS record, a file path), creating the new one fails with "already exists". Fix: don't hard-code the name; use a **prefix** (`name_prefix = "web-"` in many AWS resources) or add a random suffix.

It also **spreads, upstream**: if A has `create_before_destroy` and A **depends on** B, Terraform forces create-before-destroy on B's replacement too (otherwise B would have to be destroyed before A's new copy exists, which is a cycle). ⚠️ *Corrected:* it propagates to A's **dependencies** (what A references), not to A's dependents. Resources that depend on A keep their own setting, so set `create_before_destroy` explicitly where you need it downstream. You can see the forced setting in `terraform plan` as `+/-` on B.

### `prevent_destroy`

A safety catch for things you can't afford to lose: production databases, state buckets, KMS (Key Management Service) keys, DNS zones. Limits:

- It only protects while the block **is in the code**. Deleting the whole resource block also deletes the `lifecycle` setting, and the next plan happily destroys the object. (Use `removed` blocks for intentional removal, topic 40, and cloud-side protection like `deletion_protection = true` on an RDS (Relational Database Service, AWS's managed databases) instance for real safety.)
- It blocks **replacement** too, which is often exactly what you want to find out.

### `ignore_changes`

Tells Terraform: "this attribute is managed by someone else; don't fight over it."

```hcl
lifecycle {
  ignore_changes = [
    desired_count,                 # an autoscaler changes this
    tags["LastScanned"],           # a security scanner writes this tag
  ]
}
```

- Lists **attribute names** (not strings), optionally with keys/indexes.
- `ignore_changes = all`: ignore every attribute after creation. ⚠️ Very blunt; you'll forget why.
- The attribute is still **set at create time** from your config. Only later differences are ignored.
- Can't ignore `count`/`for_each` or meta-arguments.

### `replace_triggered_by`

Replace this resource whenever another one changes, even though nothing forces it:

```hcl
resource "aws_instance" "app" {
  # ...
  lifecycle {
    # a launch template = a saved "how to build a VM" recipe in AWS
    replace_triggered_by = [aws_launch_template.app.latest_version]
  }
}
```

- References a **resource** or a resource's **attribute** (not variables or locals directly).
- To trigger on a plain value, wrap it in **`terraform_data`**.

### `terraform_data`: the built-in "store a value" resource

`terraform_data` (Terraform 1.4+) is a resource that comes with Terraform itself (no provider to install). It:

- stores whatever you put in **`input`** and copies it to the **`output`** attribute,
- is **replaced** whenever **`triggers_replace`** changes.

It's mostly used as a trigger for `replace_triggered_by`, and it replaces the older ⚠️ `null_resource` from the `null` provider (topic 52).

---

## Why it exists

Terraform's default behaviour is correct but not always **safe** or **practical**:

- Default replacement causes downtime → `create_before_destroy`.
- One wrong change could destroy a production database → `prevent_destroy`.
- Other systems (autoscalers, scanners, Kubernetes controllers) legitimately change some attributes, and Terraform would keep "fixing" them back → `ignore_changes`.
- Some changes should cause a rebuild that the provider doesn't know about (new app version → new servers) → `replace_triggered_by`.

---

## Analogy

**House rules for a renovation contractor.**

- `create_before_destroy`: "Build the new bathroom **before** removing the old one, so we're never without one." The trap: if there's only one bathroom-shaped space (a unique name), you can't build the new one in the same spot first.
- `prevent_destroy`: a sign on the load-bearing wall: "**Do not demolish.**" If the plans say demolish it, the contractor stops and asks.
- `ignore_changes`: "The tenant picks the curtains. Don't swap them back to the ones in the blueprint."
- `replace_triggered_by`: "If the boiler is replaced, redo the radiators too," even though the radiators' blueprint didn't change.
- `destroy = false`: "If the blueprint drops the garden shed, just stop maintaining it. Don't knock it down."

---

## How it works inside

- **During planning**, after the provider says "replacement required", core checks `create_before_destroy` to choose `-/+` or `+/-`, and adjusts the apply graph: with CBD, the **create** node runs before the **destroy** node, and dependents are updated in between.
- **`prevent_destroy`** is checked when the plan is finalised: any `delete` (including the delete half of a replacement) on such a resource → error, **no plan at all**.
- **`ignore_changes`**: before asking the provider to plan, core copies the **prior state's** values into the config for the ignored attributes, so the provider sees no difference.
- **`replace_triggered_by`**: during planning, core checks whether any referenced resource has a planned change (create/update/replace). If so, it converts this resource's action to a replacement.

---

## Code / commands

`~/tf-lab/24-lifecycle/versions.tf` with `local` and `random`. (`terraform_data` needs no provider entry.)

### 1. `prevent_destroy`

```hcl
resource "local_file" "important" {
  filename = "${path.module}/important.txt"
  content  = "do not lose me\n"

  lifecycle {
    prevent_destroy = true
  }
}
```

```
$ terraform apply -auto-approve      # creates it
$ terraform destroy
╷
│ Error: Instance cannot be destroyed
│
│   on main.tf line 1:
│    1: resource "local_file" "important" {
│
│ Resource local_file.important has lifecycle.prevent_destroy set, but the plan
│ calls for this resource to be destroyed. To avoid this error and continue
│ with the plan, either disable lifecycle.prevent_destroy or reduce the scope
│ of the plan using the -target option.
╵
```

Change the `content` (which forces replacement) and plan: the same error. Replacement includes a destroy.

### 2. `create_before_destroy` and its trap

```hcl
variable "version_tag" {
  type    = string
  default = "v1"
}

resource "local_file" "release" {
  filename = "${path.module}/release-${var.version_tag}.txt"   # unique per version
  content  = "release ${var.version_tag}\n"

  lifecycle {
    create_before_destroy = true
  }
}
```

```
$ terraform apply -var="version_tag=v2"
  # local_file.release must be replaced
+/- resource "local_file" "release" {
      ~ content  = <<-EOT # forces replacement
          - release v1
          + release v2
        EOT
      ~ filename = "./release-v1.txt" -> "./release-v2.txt" # forces replacement
      ...
    }

Plan: 1 to add, 0 to change, 1 to destroy.
...
local_file.release: Creating...
local_file.release: Creation complete after 0s [id=...]
local_file.release (deposed object 3f9a2c1e): Destroying... [id=...]
local_file.release (deposed object 3f9a2c1e): Destruction complete after 0s
```

- `+/-`: create first.
- **Deposed object**: the old instance, kept aside after the new one is created and until it's destroyed. If the destroy fails, it stays in state as "deposed" and the next apply retries the destroy.

Now the trap. Remove `-${var.version_tag}` from `filename` (same file name for every version) and apply `v3`:

```
local_file.release: Creating...                       # writes release.txt (v3)
local_file.release: Creation complete after 0s
local_file.release (deposed object ...): Destroying... # deletes release.txt!
```

`ls`: `release.txt` is **gone**. The old and new objects had the same "name" (path), so destroying the old one destroyed the new one too. With cloud resources, it's even more visible: the **create** fails with `already exists` (e.g. `BucketAlreadyOwnedByYou`, `EntityAlreadyExists`). Unique or prefixed names are the fix.

### 3. `ignore_changes`

```hcl
resource "random_pet" "server" {
  keepers = {
    image = "nginx:1.29"
  }
  lifecycle {
    ignore_changes = [keepers]        # image updates are rolled out elsewhere; keep the name
  }
}
```

Change the image to `nginx:1.30` and plan:

```
No changes. Your infrastructure matches the configuration.
```

Without `ignore_changes`, it would be `-/+` (topic 11). Remove the setting and plan again to compare.

### 4. `replace_triggered_by` with `terraform_data`

```hcl
variable "app_version" {
  type    = string
  default = "1.0.0"
}

resource "terraform_data" "app_version" {
  input = var.app_version              # stores the value; changes → this resource updates
}

resource "random_pet" "worker" {
  lifecycle {
    replace_triggered_by = [terraform_data.app_version]   # new version → new worker name
  }
}
```

```
$ terraform apply -var="app_version=1.1.0"
  # random_pet.worker will be replaced due to changes in replace_triggered_by
-/+ resource "random_pet" "worker" {
      ~ id        = "lucky-wren" -> (known after apply)
        # (2 unchanged attributes hidden)
    }

  # terraform_data.app_version will be updated in-place
  ~ resource "terraform_data" "app_version" {
        id     = "8b1e..."
      ~ input  = "1.0.0" -> "1.1.0"
      ~ output = "1.0.0" -> (known after apply)
    }

Plan: 1 to add, 1 to change, 1 to destroy.
```

- `will be replaced due to changes in replace_triggered_by`: Terraform tells you exactly why.
- `terraform_data.app_version` is just **updated** (`input` changed); that update is the trigger.

### 5. Reading "why is this being replaced?"

A method for any surprising `-/+`:

1. Find `# forces replacement` in the resource's diff. That argument is the cause.
2. Its new value is `(known after apply)`? → the cause is **upstream**. Follow the reference to the first resource in the chain that's being replaced (the root of the cascade).
3. No `forces replacement` line? Look at the header: `tainted`, `as requested`, or `due to changes in replace_triggered_by`.
4. A `-` and a `+` for **different addresses**? That's a rename/move, not a replacement → `moved` block (topic 40).

---

## Hands-on lab

1. `prevent_destroy`: create `important.txt`, try `destroy`, try a content change. Then **delete the whole resource block** and plan: it now plans to destroy. (That's the limit.) Put it back.
2. `create_before_destroy`: do the version experiment with unique names, then reproduce the same-name trap and see the file vanish.
3. Watch for `deposed` in the apply output and in `terraform state list` if you interrupt a CBD apply with Ctrl-C at the right moment (tricky; optional).
4. `ignore_changes`: change `keepers` with and without it.
5. `replace_triggered_by`: bump `app_version` twice and read the plan header each time.
6. Make a three-resource cascade (a `random_pet` whose ID feeds two `local_file`s), change the pet's `keepers`, and use the method above to find the root cause in the plan.
7. Remove `prevent_destroy` and run `terraform destroy`.

---

## Where you'll use it

- **`prevent_destroy`** on production databases, state buckets, KMS keys, Route 53 zones, and anything holding data.
- **`create_before_destroy`** on load-balancer target groups (the list of servers a load balancer sends traffic to), launch templates, TLS certificates, security groups, anything that would cause downtime if it disappeared first, together with `name_prefix`.
- **`ignore_changes`** on: autoscaling `desired_capacity`, ECS `desired_count`, tags written by other tools, `user_data`/AMI on long-lived instances you patch in place.
- **`replace_triggered_by`** to roll instances when a launch template, config file or app version changes.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `Instance cannot be destroyed` | `prevent_destroy` blocked a destroy or replacement. | Good: find out why the plan destroys it. If intended, remove the flag in a reviewed PR. |
| CBD create fails: `already exists` / `EntityAlreadyExists` / `BucketAlreadyOwnedByYou` | Old and new objects need the same unique name. | Use `name_prefix` or a random suffix; or accept downtime without CBD. |
| `Cycle` errors after adding `create_before_destroy` | CBD reverses ordering for dependents and can create loops with other CBD/non-CBD resources. | Apply CBD consistently along the dependency chain. |
| Changes you made in code "don't apply" | The attribute is in `ignore_changes` (maybe via `all`). | Remove it from the list for that change, apply, put it back. |
| `Variables not allowed` / `Invalid lifecycle` | Used a variable in `lifecycle`. | Lifecycle settings must be literals. |
| `Invalid replace_triggered_by expression` | Referenced a variable/local directly. | Wrap the value in `terraform_data` and reference that. |
| Resource deleted after you removed its block, despite `prevent_destroy` | The flag lived in the deleted block. | Use cloud-side deletion protection; use `removed` blocks to stop managing instead (topic 40). |

---

## Quick recap

- Replacements come from force-new arguments, unknown upstream values, taint, `-replace`, or `replace_triggered_by`. Read `# forces replacement`.
- `create_before_destroy`: `+/-`, no downtime, but beware unique names.
- `prevent_destroy`: blocks destroys and replacements while the block is in code.
- `ignore_changes`: let other systems own certain attributes.
- `replace_triggered_by`: replace when another resource changes; use `terraform_data` for plain values.
- `destroy = false` (1.16+): forget instead of destroying.

### Interview questions

1. **Q: How do you avoid downtime when a resource must be replaced?**
   A: `create_before_destroy = true`, so the new object is created and dependents are switched before the old one is destroyed. The resource must allow two copies at once, so use name prefixes rather than fixed names.
2. **Q: How do you protect a production database from `terraform destroy`?**
   A: `lifecycle { prevent_destroy = true }` in Terraform, plus provider-side protection like `deletion_protection = true`, because `prevent_destroy` disappears if someone deletes the resource block.
3. **Q: An autoscaler changes `desired_count`, and every plan tries to reset it. What do you do?**
   A: Add `desired_count` to `lifecycle.ignore_changes`, so Terraform sets it at creation and then leaves it to the autoscaler.
