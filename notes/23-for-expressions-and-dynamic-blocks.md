# 23 · `for` Expressions and `dynamic` Blocks

> **Level:** ★★ intermediate
>
> **You will learn:** how to **reshape data** with `for` expressions (list form `[...]`, map form `{...}`, filtering with `if`, using the index or key, grouping with `...`), the **`flatten` pattern** for nested loops (e.g. every service × every environment), and how to **generate repeated nested blocks** with `dynamic` blocks, plus when `dynamic` makes code worse. The lab uses the free `cloudinit` provider, which you'll meet again for EC2 `user_data`.
>
> **You need:** topics 17 (types), 18 (functions), 22 (`for_each`, maps of objects).
>
> **Part 2 · Making configuration flexible** · Previous: [22 · `for_each`](22-for-each.md) · Next: [24 · The `lifecycle` meta-argument](24-lifecycle.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **`for` expression** builds a new list or map from an existing one. A **`dynamic` block** builds repeated **nested blocks** from a collection.

```hcl
# for expressions
[for n in var.names : upper(n)]                        # list → list
{for n in var.names : n => length(n)}                  # list → map
[for k, v in var.members : k if v.plan == "gold"]      # filter

# dynamic block: one "part" block per script
dynamic "part" {
  for_each = var.scripts                               # map: filename => content
  content {
    filename     = part.key
    content_type = "text/x-shellscript"
    content      = part.value
  }
}
```

**Picture it.** A `for` expression is a **sorting machine** on a conveyor belt: items go in, get changed or filtered, come out in a new box. A `dynamic` block is a **stamp** that prints one form section per item.

**The one thing to know.** `[ ... ]` makes a **list**, `{ ... => ... }` makes a **map**. Inside `dynamic "X"`, the loop variable is named `X` (the block's name), with `X.key` and `X.value`.

**Try it now:** in `terraform console`: `[for i in range(5) : i * i if i % 2 == 0]` → `[0, 4, 16]`.

**You've got it if you can:** turn a list of objects into a map keyed by name (for `for_each`), and write a `dynamic` block.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### `for` expressions

```
[for <ITEM> in <COLLECTION> : <RESULT> if <CONDITION>]                → list (tuple)
{for <ITEM> in <COLLECTION> : <KEY> => <VALUE> if <CONDITION>}         → map (object)
```

| Form | Example | Result |
|---|---|---|
| List from list | `[for s in ["a","b"] : upper(s)]` | `["A","B"]` |
| With index | `[for i, s in ["a","b"] : "${i}-${s}"]` | `["0-a","1-b"]` |
| From a map | `[for k, v in {a=1, b=2} : "${k}=${v}"]` | `["a=1","b=2"]` (sorted by key) |
| Map from list | `{for s in ["a","bb"] : s => length(s)}` | `{a=1, bb=2}` |
| Map from map | `{for k, v in var.m : k => v * 2}` | Same keys, new values |
| Filter | `[for s in var.list : s if s != ""]` | Drops empty strings |
| **Group** | `{for u in var.users : u.team => u.name...}` | `{team => [names...]}` |

- `if` filters: only items where the condition is true are kept.
- With `...` after the value, items with the **same key** are grouped into a list instead of causing a `Duplicate object key` error.
- A `for` over a **set** gives elements in sorted order; over a **map**, keys in sorted order.

### The flatten pattern: nested loops

`for_each` takes one flat map. When you need "every A × every B" (e.g. each service in each environment), build pairs with nested `for` expressions and `flatten`, then key them:

```hcl
locals {
  pairs = flatten([
    for env in var.environments : [
      for svc in var.services : {
        env = env
        svc = svc
      }
    ]
  ])
  # → [{env="dev",svc="api"}, {env="dev",svc="web"}, {env="prod",svc="api"}, ...]

  pairs_by_key = { for p in local.pairs : "${p.env}-${p.svc}" => p }
  # → { "dev-api" = {...}, "dev-web" = {...}, ... }
}

resource "local_file" "deploy" {
  for_each = local.pairs_by_key
  filename = "${path.module}/${each.key}.txt"
  content  = "deploy ${each.value.svc} to ${each.value.env}\n"
}
```

`flatten` turns the list-of-lists into one list. This is one of the most common patterns in real Terraform.

### `dynamic` blocks

Some arguments are **nested blocks** that may repeat (topic 08), e.g. `ingress { }` in an AWS security group, `ports { }` in a Docker container, `part { }` in cloud-init. You can't put `for_each` inside them directly, so you use `dynamic`:

```hcl
dynamic "<BLOCK NAME>" {
  for_each = <collection>          # list, set or map
  iterator = <optional new name>   # default: the block name
  content {
    # arguments of the nested block, using <BLOCK NAME>.key / .value
  }
}
```

- `for_each` here works like the resource-level one, but **lists are allowed** (key = index).
- The iterator is named after the block (`part.value`) unless you set `iterator = s` (`s.value`).
- `dynamic` blocks can be nested inside each other.
- `dynamic` only generates **nested blocks**. It can't generate arguments, resources or top-level blocks.

---

## Why it exists

Inputs rarely arrive in the exact shape a resource needs. Your variable might be a list of objects, but `for_each` wants a map; a YAML file lists users, but you need them grouped by team. `for` expressions do these transformations **inside** Terraform, declaratively. `dynamic` exists because a security group with 15 rules from a variable can't be written as 15 hard-coded blocks.

---

## Analogy

- **`for` expression** = a **sorting machine** at a post office: parcels go in, get relabelled, unwanted ones are dropped (`if`), parcels for the same street are bundled (`...`), and they come out in a new box (list or map).
- **`flatten`** = emptying several small boxes into one big one.
- **`dynamic` block** = a **rubber stamp** that prints one "dependant" section on a tax form for each child you list.

---

## How it works inside

- `for` expressions are evaluated like any expression: during plan, re-evaluated at apply if they contain unknowns. Unknown **collections** make the result unknown; unknown **elements** stay unknown inside a known collection.
- Keys built with `{ ... => ... }` must be unique (unless grouped with `...`).
- A `dynamic` block is **expanded before** the provider sees the configuration: the provider just receives N ordinary nested blocks. In plans, they appear as normal blocks, not as `dynamic`.
- If a `dynamic` block's `for_each` is unknown at plan time, the whole set of nested blocks is unknown, which can show as a large diff.

---

## Code / commands

### `for` expressions in the console

```bash
mkdir -p ~/tf-lab/23-for && cd ~/tf-lab/23-for && terraform console
```

```
> [for s in ["web", "api", ""] : upper(s) if s != ""]
[
  "WEB",
  "API",
]
> {for s in ["web", "api"] : s => length(s)}
{
  "api" = 3
  "web" = 3
}
> [for i, s in ["a", "b"] : "${i}:${s}"]
[
  "0:a",
  "1:b",
]
> {for u in [{name="asha", team="web"}, {name="ben", team="data"}, {name="chen", team="web"}] : u.team => u.name...}
{
  "data" = [
    "ben",
  ]
  "web" = [
    "asha",
    "chen",
  ]
}
> flatten([for e in ["dev", "prod"] : [for s in ["api", "web"] : "${e}-${s}"]])
[
  "dev-api",
  "dev-web",
  "prod-api",
  "prod-web",
]
> exit
```

### List of objects → map for `for_each`

A very common need: your input is a list (easy for users to write), but `for_each` needs a map:

```hcl
variable "services" {
  type = list(object({
    name = string
    port = number
  }))
  default = [
    { name = "api", port = 8080 },
    { name = "web", port = 3000 },
  ]
}

resource "local_file" "svc" {
  for_each = { for s in var.services : s.name => s }     # key by name
  filename = "${path.module}/svc-${each.key}.conf"
  content  = "port=${each.value.port}\n"
}
```

### A `dynamic` block you can run: cloud-init parts

**cloud-init** is the standard tool that runs setup scripts when a Linux VM boots for the first time. AWS EC2 passes it a **`user_data`** document (topic 33). The `hashicorp/cloudinit` provider builds a multi-part cloud-init document, and each script is a nested `part { }` block. That's a perfect `dynamic` use, and it runs locally.

`versions.tf`:

```hcl
terraform {
  required_version = ">= 1.14"
  required_providers {
    local     = { source = "hashicorp/local", version = "~> 2.5" }
    cloudinit = { source = "hashicorp/cloudinit", version = "~> 2.3" }
  }
}
```

`main.tf`:

```hcl
variable "boot_scripts" {
  description = "Scripts to run at first boot, in order. Key = filename."
  type        = map(string)
  default = {
    "01-update.sh" = "#!/bin/bash\ndnf -y update\n"
    "02-nginx.sh"  = "#!/bin/bash\ndnf -y install nginx && systemctl enable --now nginx\n"
  }
}

variable "extra_packages" {
  type    = list(string)
  default = ["htop", "jq"]
}

data "cloudinit_config" "app" {
  gzip          = false
  base64_encode = false

  # Static part: a cloud-config YAML document built with yamlencode
  part {
    content_type = "text/cloud-config"
    content = yamlencode({
      packages = var.extra_packages
    })
  }

  # One part per script
  dynamic "part" {
    for_each = var.boot_scripts
    content {
      filename     = part.key
      content_type = "text/x-shellscript"
      content      = part.value
    }
  }
}

resource "local_file" "user_data" {
  filename = "${path.module}/user-data.mime"
  content  = data.cloudinit_config.app.rendered
}
```

- The first `part` is written normally; the `dynamic "part"` adds one more per script. Static and dynamic blocks of the same type can be mixed.
- `part.key` / `part.value`: the iterator takes the block's name.
- `rendered`: the data source's output: the full multi-part document.

```bash
terraform init && terraform apply -auto-approve
grep -E 'Content-Type|filename|^#!' user-data.mime
```

```
Content-Type: multipart/mixed; boundary="MIMEBOUNDARY"
Content-Type: text/cloud-config
Content-Type: text/x-shellscript
Content-Disposition: attachment; filename="01-update.sh"
#!/bin/bash
Content-Type: text/x-shellscript
Content-Disposition: attachment; filename="02-nginx.sh"
#!/bin/bash
```

Three parts: one static, two generated. Add a third script to the map and plan: the document gains one part.

### The same pattern in AWS (read it now, run it in Part 4)

```hcl
variable "ingress_ports" {
  type    = list(number)
  default = [80, 443]
}

resource "aws_security_group" "web" {
  name = "web"

  dynamic "ingress" {
    for_each = var.ingress_ports
    iterator = port                        # rename: port.value instead of ingress.value
    content {
      from_port   = port.value
      to_port     = port.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

(Modern AWS code prefers separate `aws_vpc_security_group_ingress_rule` resources with `for_each`, topic 32. But you'll see this `dynamic` style in many existing codebases.)

### When `dynamic` makes code worse

```hcl
# ❌ dynamic for a block that always appears exactly once: just write the block
dynamic "logging" {
  for_each = [1]
  content { level = "info" }
}

# ✅ but this "optional block" trick IS a standard idiom:
dynamic "logging" {
  for_each = var.logging_enabled ? [1] : []     # 0 or 1 block
  content { level = var.log_level }
}
```

HashiCorp's advice: use `dynamic` sparingly. Hidden structure is harder to read than written-out blocks. Use it when the number of blocks really comes from data, or for the optional-block trick.

---

## Hands-on lab

1. Run all console examples. Then write, without looking: a map from name → port for `var.services`; a list of only the ports above 5000.
2. Build the flatten example with `environments = ["dev", "prod"]` and `services = ["api", "web", "db"]`. Apply, check 6 files exist, and check `terraform state list`.
3. Build the cloud-init project. Apply, inspect `user-data.mime`.
4. Add a `"03-app.sh"` script to the map; plan; apply.
5. Add an `iterator = script` to the `dynamic` block and update the references (`script.key`, `script.value`).
6. Use the optional-block trick: add a `dynamic "part"` that adds a `text/x-shellscript` debugging part only when a `debug` variable is `true`.
7. Break it: a `for` expression that produces duplicate keys (`{for u in users : u.team => u.name}` without `...`). Read the `Duplicate object key` error.
8. `terraform destroy`.

---

## Where you'll use it

- **Reshaping inputs** for `for_each`: list → map keyed by name.
- **Flatten pattern**: subnets × route tables, users × groups, services × environments.
- **Grouping**: IAM users by team, alarms by severity.
- **`dynamic`**: security group rules (older style), container ports and volumes (topic 27), cloud-init parts, optional blocks.

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `Duplicate object key` | Two items produce the same map key. | Make keys unique, or group with `...`. |
| `Invalid 'for' expression ... Key expression is required when building an object` | Used `{for x in y : x}` without `=>`. | `{for x in y : x => ...}`, or use `[...]` for a list. |
| `Unsupported attribute` on the iterator | Used `each.value` inside `dynamic` (that's for resource-level `for_each`), or the wrong iterator name. | Use `<block name>.value` or your `iterator` name. |
| `Blocks of type "X" are not expected here` in `dynamic "X"` | The block name doesn't exist in that resource's schema, or it's an argument, not a block. | Check the docs; arguments can't be generated with `dynamic`. |
| Plan shows every nested block changing | Order of a list-based `for_each` changed, and the provider treats the blocks as an ordered list. | Sort the input (`sort()`), or use a map; prefer separate resources for rules. |

---

## Quick recap

- `[for x in c : f(x) if cond]` → list; `{for k, v in c : key => val}` → map; `...` groups duplicates.
- `flatten` + a keyed map is the standard way to do nested loops for `for_each`.
- `dynamic "block" { for_each = ...; content { ... } }` generates repeated nested blocks; the iterator is `block.key`/`block.value`.
- Use `dynamic` when the count truly comes from data, or for optional blocks (`cond ? [1] : []`). Don't overuse it.

### Interview questions

1. **Q: How do you loop over two lists to create every combination?**
   A: Nested `for` expressions inside `flatten()` to produce a flat list of objects, then a `for` expression to turn it into a map with a unique composite key, and use that with `for_each`.
2. **Q: What is a `dynamic` block for?**
   A: Generating a variable number of nested blocks (like security group `ingress` blocks) from a collection. It works only for nested blocks, not arguments or resources.
3. **Q: How do you make an optional nested block?**
   A: `dynamic "x" { for_each = var.enabled ? [1] : [] content { ... } }`: zero or one block depending on the condition.
