# 21 · `count`

> **Level:** ★★ intermediate
>
> **You will learn:** the `count` meta-argument: making **N copies** of a resource, `count.index`, addresses like `local_file.web[0]`, referencing instances and using splat (`[*]`) to collect them, **conditional resources** (`count = var.enabled ? 1 : 0`) with `one()`, `count` on modules, the rule that `count` must be **known at plan time**, and the big trap: **removing an item from the middle of a list shifts every index and replaces the wrong things**, which is why topic 22's `for_each` exists.
>
> **You need:** topics 11 (addresses), 17 (lists, splat, conditionals), 18 (`length`, `one`), 20 (the graph).
>
> **Part 2 · Making configuration flexible** · Previous: [20 · Dependencies and the graph](20-dependencies-and-the-graph.md) · Next: [22 · `for_each`](22-for-each.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** `count = N` turns one resource block into **N instances**, numbered from 0. Inside the block, `count.index` is the current number.

```hcl
resource "local_file" "web" {
  count    = 3
  filename = "${path.module}/web-${count.index}.txt"    # web-0, web-1, web-2
  content  = "server ${count.index}\n"
}
# addresses: local_file.web[0], local_file.web[1], local_file.web[2]
# all of them: local_file.web[*].filename
```

The other big use: **on/off switches**:

```hcl
resource "local_file" "debug" {
  count    = var.debug ? 1 : 0          # 1 = create it, 0 = don't
  filename = "${path.module}/debug.txt"
  content  = "debug on\n"
}
```

**Picture it.** Numbered lockers: locker 0, 1, 2. Fine for identical things. But if you empty locker 1 and everyone shifts down a number, people get the wrong lockers.

**The one thing to know.** `count` identifies instances by **position**. Use it for **identical** copies and **on/off** switches. For a list of **named, different** things, use `for_each` (topic 22).

**Try it now:** make `count = length(var.names)` with `names = ["a","b","c"]`, apply, then remove `"a"` and plan. Count how many files get replaced.

**You've got it if you can:** explain why removing the first element replaces everything after it.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

`count` is a **meta-argument** (works on any resource, data source or module):

```hcl
resource "TYPE" "NAME" {
  count = <whole number known at plan time>
  # count.index is 0, 1, ..., count-1
}
```

| You write | It means |
|---|---|
| `count = 3` | Instances `[0]`, `[1]`, `[2]` |
| `count = 0` | **No** instances (the resource exists only in code) |
| `count = length(var.list)` | One instance per list element |
| `count = var.enabled ? 1 : 0` | Create it or not |

### Addresses and references

| Expression | Value |
|---|---|
| `local_file.web` | A **list** of all instances (objects) |
| `local_file.web[1]` | One instance |
| `local_file.web[1].filename` | One attribute |
| `local_file.web[*].filename` | List of that attribute from **all** instances (splat) |
| `length(local_file.web)` | How many exist |
| `one(local_file.debug[*].filename)` | For a 0-or-1 resource: the value, or `null` if `count = 0` |

On the command line, quote addresses with brackets: `terraform state show 'local_file.web[0]'` (the shell would otherwise treat `[0]` as a pattern).

---

## Why it exists

You often need several **identical** things (3 worker VMs, 2 replicas) or something that exists **only in some environments** (a bastion host in dev, a read replica in prod). Copy-pasting blocks doesn't scale and can't depend on a variable. `count` makes the number a value.

---

## Analogy

**Numbered lockers at a gym.**

- `count = 3` gives lockers #0, #1, #2. Perfect when every locker holds the same kind of thing.
- Assign lockers from a list of members: Asha → #0, Ben → #1, Chen → #2.
- Asha leaves. If you rebuild the assignments from the new list, Ben → #0 and Chen → #1. **Both** Ben and Chen must move their stuff, and locker #2 is emptied. Nobody actually wanted to move.
- That's the `count` index-shift problem. Name tags on lockers (topic 22's `for_each`) fix it.

---

## How it works inside

- During **plan**, Terraform must know `count` **before** it can create instance nodes in the graph. So `count` can use variables, locals, data sources read at plan time, and attributes of **existing** resources, but not values that are `(known after apply)`.
- Each instance is a separate object in state, with an `index_key`:

```json
{
  "type": "local_file",
  "name": "web",
  "instances": [
    { "index_key": 0, "attributes": { "filename": "./web-0.txt", ... } },
    { "index_key": 1, "attributes": { "filename": "./web-1.txt", ... } }
  ]
}
```

- When `count` changes, Terraform compares **by index**: index 0 in config ↔ index 0 in state. New indexes are created; missing ones destroyed; same index with different arguments → updated or replaced.

---

## Code / commands

`~/tf-lab/21-count/` (versions.tf with `local`).

```hcl
variable "names" {
  type    = list(string)
  default = ["asha", "ben", "chen"]
}

variable "debug" {
  type    = bool
  default = false
}

resource "local_file" "member" {
  count    = length(var.names)
  filename = "${path.module}/member-${count.index}.txt"
  content  = "locker ${count.index}: ${var.names[count.index]}\n"
}

resource "local_file" "debug" {
  count    = var.debug ? 1 : 0
  filename = "${path.module}/debug.txt"
  content  = "debug enabled\n"
}

output "member_files" {
  value = local_file.member[*].filename
}

output "debug_file" {
  value = one(local_file.debug[*].filename)       # null when debug = false
}
```

### Apply

```
$ terraform apply -auto-approve
...
local_file.member[0]: Creating...
local_file.member[2]: Creating...
local_file.member[1]: Creating...
...
Outputs:

debug_file = null
member_files = [
  "./member-0.txt",
  "./member-1.txt",
  "./member-2.txt",
]
```

- The three instances are created in parallel (no dependencies between them).
- `debug_file = null`: `count = 0`, so `one([])` = `null`.

### The index-shift problem

Remove `"asha"` from the **start** of the list: `names = ["ben", "chen"]`.

```
$ terraform plan
  # local_file.member[0] must be replaced
-/+ resource "local_file" "member" {
      ~ content = <<-EOT # forces replacement
          - locker 0: asha
          + locker 0: ben
        EOT
      ...
    }

  # local_file.member[1] must be replaced
-/+ resource "local_file" "member" {
      ~ content = <<-EOT # forces replacement
          - locker 1: ben
          + locker 1: chen
        EOT
      ...
    }

  # local_file.member[2] will be destroyed
  # (because index [2] is out of range for count)
  - resource "local_file" "member" {
      - content  = <<-EOT
            locker 2: chen
        EOT -> null
      ...
    }

Plan: 2 to add, 0 to change, 3 to destroy.
```

- You removed **one** member. Terraform wants to touch **all three**.
- `(because index [2] is out of range for count)`: Terraform tells you why.
- With files, that's harmless. With **VMs or databases**, Ben's and Chen's servers would be destroyed and rebuilt. That's how outages happen.

Remove from the **end** instead (`["asha", "ben"]`): only `[2]` is destroyed. `count` is only safe when you add/remove at the end, or when all instances are truly identical.

### `count` must be known at plan time

```hcl
resource "random_integer" "n" {
  min = 1
  max = 3
}

resource "local_file" "bad" {
  count    = random_integer.n.result        # not known until apply
  filename = "${path.module}/bad-${count.index}.txt"
  content  = "x"
}
```

```
╷
│ Error: Invalid count argument
│
│   on main.tf line 34, in resource "local_file" "bad":
│   34:   count    = random_integer.n.result
│
│ The "count" value depends on resource attributes that cannot be determined
│ until apply, so Terraform cannot predict how many instances will be created.
│ To work around this, use the -target argument to first apply only the
│ resources that the count depends on.
╵
```

The suggested `-target` workaround (topic 63) is a ⚠️ one-off fix. The real fix is to base `count` on values you know up front (variables, locals).

### `count` on a module

```hcl
module "bastion" {
  source = "./modules/bastion"     # modules: topic 43
  count  = var.environment == "dev" ? 1 : 0
}
# reference: module.bastion[0].public_ip, or one(module.bastion[*].public_ip)
```

---

## Hands-on lab

1. Build the project and apply. Check the files and `terraform state list`.
2. Turn debug on: `terraform apply -var="debug=true"`. Check the `debug_file` output. Turn it off again: the file is destroyed.
3. Remove `"asha"` from the start of the list and plan. Count the replacements. **Don't apply.** Then remove `"chen"` from the end instead and plan: only one destroy.
4. Insert `"dev"` in the **middle**. What happens to everything after it?
5. Try `terraform state show local_file.member[0]` without quotes, then with quotes. (In bash, an unquoted `[0]` can glob-match a filename `local_file.member0`; quoting is the safe habit.)
6. Create the `Invalid count argument` error and read it.
7. `terraform destroy`.

---

## Where you'll use it

- **On/off features**: `count = var.create_bastion ? 1 : 0`, `count = var.enable_monitoring ? 1 : 0`. Still the most common use in real modules.
- **Identical copies**: N worker instances behind a load balancer where none is special.
- **Legacy code**: lots of older Terraform uses `count` for lists. Migrating it to `for_each` uses `moved` blocks (topic 40).

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| Removing one list item replaces many resources | Index shift. | Use `for_each` with stable keys (topic 22). Migrate existing ones with `moved` blocks (topic 40). |
| `Invalid count argument ... cannot be determined until apply` | `count` uses an unknown value. | Base it on variables/locals known at plan time. |
| `Missing resource instance key` / `Because local_file.x has "count" set, its attributes must be accessed on specific instances` | Referenced `local_file.x.filename` without an index. | Use `local_file.x[0].filename`, `[*]`, or `one(...)`. |
| `Invalid index` when `count = 0` | Referenced `local_file.debug[0]` while it doesn't exist. | Use `one(local_file.debug[*].attr)` or `try(local_file.debug[0].attr, null)`. |
| `count` and `for_each` in the same block | Not allowed. | Choose one. |

---

## Quick recap

- `count = N` → instances `[0]..[N-1]`; `count.index` inside; `[*]` collects attributes.
- `count = cond ? 1 : 0` is the standard on/off switch; read it with `one(...[*].x)`.
- `count` must be known at plan time.
- Instances are matched **by position**: removing from the middle shifts indexes and replaces everything after. Use `for_each` for named things.

### Interview questions

1. **Q: How do you create a resource only in some environments?**
   A: `count = var.enabled ? 1 : 0` (or a condition on the environment), and reference it with `one(resource.x[*].attr)` so it's `null` when absent.
2. **Q: What's the main problem with `count` for a list of items?**
   A: Instances are tracked by index. Removing or inserting an element shifts later indexes, so Terraform updates or replaces resources that didn't actually change. `for_each` uses stable keys instead.
3. **Q: Why might you get "Invalid count argument"?**
   A: `count` depends on a value only known after apply, e.g. a resource attribute. Terraform needs the number of instances at plan time.
