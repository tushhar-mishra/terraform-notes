# 16 · Locals

> **Level:** ★ foundation
>
> **You will learn:** what **local values** (`locals`) are, how they differ from variables, the standard uses (a naming prefix, common tags, derived values, one place for a long expression), how locals are evaluated, and when locals make code **harder** to read.
>
> **You need:** topics 14 (variables), 15 (outputs).
>
> **Part 2 · Making configuration flexible** · Previous: [15 · Outputs](15-outputs.md) · Next: [17 · Types and expressions](17-types-and-expressions.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **local** is a named value computed **inside** the configuration. Write an expression once, give it a name, use it everywhere as `local.<name>`.

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"     # e.g. "shop-dev"
  common_tags = {
    project     = var.project
    environment = var.environment
    managed_by  = "terraform"
  }
}

resource "local_file" "app" {
  filename = "${path.module}/${local.name_prefix}-app.conf"
  content  = jsonencode(local.common_tags)
}
```

**Picture it.** Variables are the **ingredients** someone hands you. Locals are your **prep bowls**: chopped onions you made once and use in three dishes.

**The one thing to know.** Variables = set from **outside**. Locals = calculated **inside**, can't be overridden from outside. Note the spelling: the block is `locals { }` (plural), the reference is `local.name` (singular).

**Try it now:** add the block above to your topic 14 lab (with a `project` variable) and print `local.name_prefix` in `terraform console`.

**You've got it if you can:** say when to use a variable and when a local, and give two common uses for locals.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

```hcl
locals {
  environment_short = substr(var.environment, 0, 1)            # "d", "s", "p"
  is_prod           = var.environment == "prod"                 # true/false
  instance_count    = local.is_prod ? 3 : 1                     # a local can use another local
  bucket_name       = "${var.project}-${var.environment}-logs"
}
```

- A `locals` block holds **any number** of `name = expression` lines.
- You may have **several** `locals` blocks, in any file. All names must be unique across them.
- A local can use variables, resource attributes, data sources, functions and **other locals** (no loops between them).
- Read them as `local.<name>`.

### Variables vs locals vs outputs

| | Variable | Local | Output |
|---|---|---|---|
| Direction | **In** (from the caller / tfvars / CLI) | **Inside** only | **Out** |
| Can the user override it? | Yes | **No** | n/a |
| Block / reference | `variable "x" {}` / `var.x` | `locals { x = ... }` / `local.x` | `output "x" {}` / `terraform output x` |
| Can use resource attributes? | No (only defaults that are constants) | Yes | Yes |
| Typical use | Things that differ per environment | Names, tags, derived values, flags | Results for people and scripts |

---

## Why it exists

Without locals you repeat the same expression: `"${var.project}-${var.environment}"` in twenty resource names. Change the naming convention and you edit twenty lines, and miss one. Locals give you:

- **DRY** (Don't Repeat Yourself): one definition.
- **Names for ideas**: `local.is_prod` reads better than `var.environment == "prod"` repeated everywhere.
- **Fewer inputs**: instead of asking the user for `instance_count`, `instance_size` and `backup_days` separately, derive them from `environment`, so they can't be set inconsistently.

---

## Analogy

**Cooking prep bowls.** The ingredients delivered to the kitchen (variables) are raw. Before cooking, you chop and mix once into labelled bowls (locals): "onion mix", "spice blend". Each dish (resource) takes from the bowls. Customers can't change what's in your bowls; they can only change what they order (variables).

---

## How it works inside

- Each local becomes a **node in the graph** (topic 13), with edges to whatever it references.
- A local is evaluated when Terraform needs it during the walk. If it references an unknown attribute, it's unknown too (`(known after apply)`).
- Locals aren't stored in state by themselves. Only resources and outputs are. You see a local's value through what uses it, or in `terraform console`.
- Two locals that reference each other → `Error: Cycle: local.a, local.b`.

---

## Code / commands

`~/tf-lab/16-locals/` (with the usual `versions.tf` for `local`):

`variables.tf`:

```hcl
variable "project" {
  type    = string
  default = "shop"
}

variable "environment" {
  type    = string
  default = "dev"
}

variable "extra_tags" {
  type    = map(string)
  default = {}
}
```

`locals.tf`:

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"
  is_prod     = var.environment == "prod"

  # Settings derived from the environment, so they can't be set inconsistently
  replicas       = local.is_prod ? 3 : 1
  log_level      = local.is_prod ? "warn" : "debug"
  backup_enabled = local.is_prod

  # Standard tags merged with anything extra the caller passes
  common_tags = merge(
    {
      project     = var.project
      environment = var.environment
      managed_by  = "terraform"
    },
    var.extra_tags,
  )
}
```

- `cond ? a : b`: the **conditional expression**: `a` if `cond` is true, else `b` (topic 17).
- `merge(map1, map2)`: combines maps; later keys win (topic 18).

`main.tf`:

```hcl
resource "local_file" "app" {
  filename = "${path.module}/${local.name_prefix}-app.conf"
  content  = <<-EOT
    replicas=${local.replicas}
    log_level=${local.log_level}
    backups=${local.backup_enabled}
  EOT
}

resource "local_file" "tags" {
  filename = "${path.module}/${local.name_prefix}-tags.json"
  content  = jsonencode(local.common_tags)
}

output "name_prefix" {
  value = local.name_prefix
}
```

`jsonencode()` turns an HCL value into a JSON string.

### Check locals in the console

```bash
terraform init
terraform console -var="environment=prod"
```

```
> local.name_prefix
"shop-prod"
> local.replicas
3
> local.common_tags
{
  "environment" = "prod"
  "managed_by" = "terraform"
  "project" = "shop"
}
> exit
```

`terraform console` evaluates expressions using your variables and state, without changing anything (topic 18).

### Plan with prod

```
$ terraform plan -var="environment=prod" -var='extra_tags={owner="team-a"}'

  # local_file.app will be created
  + resource "local_file" "app" {
      + content  = <<-EOT
            replicas=3
            log_level=warn
            backups=true
        EOT
      + filename = "./shop-prod-app.conf"
      ...
  # local_file.tags will be created
  + resource "local_file" "tags" {
      + content  = jsonencode(
            {
              + environment = "prod"
              + managed_by  = "terraform"
              + owner       = "team-a"
              + project     = "shop"
            }
        )
      + filename = "./shop-prod-tags.json"
```

- `content = jsonencode({...})`: Terraform recognises JSON strings and shows them structured, which makes diffs readable.
- `owner = "team-a"`: merged in from `extra_tags`.

### When locals hurt readability

```hcl
# ❌ a "local" for every single value: the reader jumps around for nothing
locals {
  file_name    = "app.conf"
  file_content = "x"
}
resource "local_file" "a" {
  filename = local.file_name
  content  = local.file_content
}

# ❌ deep chains: local.c uses local.b uses local.a uses ... five hops to understand one value
```

Rule: use a local when a value is **reused** or **deserves a name**. Otherwise, write it inline.

---

## Hands-on lab

1. Build the project above. `terraform apply` (dev), check the files, then `terraform plan -var="environment=prod"` and explain each change.
2. In `terraform console`, evaluate `local.is_prod`, `local.replicas`, `local.common_tags` for dev and prod.
3. Try to override a local: `terraform plan -var="replicas=5"`. Read the error (`Value for undeclared variable`). Locals can't be set from outside.
4. Create a cycle on purpose: `a = local.b` and `b = local.a`. Run `terraform validate`. Read the `Cycle` error, then remove it.
5. Add `extra_tags = { cost_center = "1234" }` in a `terraform.tfvars` file and see it merged.
6. `terraform destroy`.

---

## Where you'll use it

- **Naming conventions**: `local.name_prefix` used in every resource name.
- **Common tags**: `local.common_tags` merged into every AWS resource's `tags` (topic 30 also shows the provider's `default_tags`).
- **Environment-derived settings**: `is_prod ? "db.r6g.large" : "db.t4g.micro"`.
- **Cleaning input**: `lower(trimspace(var.name))` once, used everywhere.
- **Shaping data for `for_each`** (topics 22–23): build the map once in a local.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `Reference to undeclared local value` | Typo, or wrote `locals.x` / `local.x` for a name that doesn't exist. | The reference is `local.x` (singular). Check the name. |
| `Duplicate local value definition` | The same name in two `locals` blocks. | Rename one. |
| `Cycle: local.a, local.b` | Locals referencing each other. | Break the loop. |
| `Value for undeclared variable` when trying `-var` on a local | Locals can't be set from outside. | Make it a variable if users must control it. |
| Code is hard to follow | Too many locals, or long chains. | Inline single-use values; keep locals near where they're used. |

---

## Quick recap

- `locals { name = expr }` defines named values; read them as `local.name`.
- Variables come from outside; locals are computed inside and can't be overridden.
- Use them for names, tags, derived settings and cleaned-up inputs.
- Don't over-use them: a local should be reused or clarify something.

### Interview questions

1. **Q: What's the difference between a variable and a local?**
   A: A variable is an input set from outside (tfvars, CLI, env, module caller). A local is an internal named expression that users can't override, often computed from variables and resource attributes.
2. **Q: Give a typical use of locals.**
   A: A common tag map merged into every resource, or a name prefix like `"${var.project}-${var.environment}"` reused in every resource name.
3. **Q: Are locals stored in state?**
   A: No. Only resources, data sources and outputs are recorded. Locals are evaluated during each run.
