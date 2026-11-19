# 22 · `for_each`

> **Level:** ★★ intermediate
>
> **You will learn:** the `for_each` meta-argument: one instance per element of a **map** or **set of strings**, `each.key` and `each.value`, addresses like `local_file.member["ben"]`, why **stable keys** fix `count`'s index-shift problem, using maps of objects for per-instance settings, collecting results with `for` expressions and `values()`, chaining `for_each` resources together, the rule that **keys must be known at plan time**, and a decision table for `count` vs `for_each`.
>
> **You need:** topics 17 (maps, sets, objects), 18 (`toset`), 21 (`count` and its trap).
>
> **Part 2 · Making configuration flexible** · Previous: [21 · `count`](21-count.md) · Next: [23 · `for` expressions and `dynamic` blocks](23-for-expressions-and-dynamic-blocks.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** `for_each` makes one instance **per key** of a map (or per string in a set). Instances are named by **key**, not by position, so removing one doesn't disturb the others.

```hcl
variable "members" {
  type = map(object({
    locker = number
    plan   = string
  }))
  default = {
    asha = { locker = 12, plan = "gold" }
    ben  = { locker = 7,  plan = "basic" }
  }
}

resource "local_file" "member" {
  for_each = var.members
  filename = "${path.module}/${each.key}.txt"                         # asha.txt, ben.txt
  content  = "locker ${each.value.locker}, plan ${each.value.plan}\n"
}
# addresses: local_file.member["asha"], local_file.member["ben"]
```

**Picture it.** Lockers with **name tags** instead of numbers. When Asha leaves, only Asha's locker is emptied.

**The one thing to know.** Inside the block: `each.key` (the map key or set element) and `each.value` (the map value; for a set, same as the key). For a plain list, wrap it: `for_each = toset(var.names)`.

**Try it now:** redo the topic 21 experiment with `for_each = toset(var.names)` and remove the first name. Only **one** destroy.

**You've got it if you can:** choose between `count` and `for_each` for a given case and explain why.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

```hcl
resource "TYPE" "NAME" {
  for_each = <map, or set of strings, with keys known at plan time>
  # each.key, each.value available here
}
```

| `for_each` input | `each.key` | `each.value` |
|---|---|---|
| `toset(["a", "b"])` | `"a"`, `"b"` | same as key |
| `{ a = "x", b = "y" }` (map of strings) | `"a"`, `"b"` | `"x"`, `"y"` |
| `{ a = { port = 80 } }` (map of objects) | `"a"` | `{ port = 80 }` → `each.value.port` |

**Not allowed directly**: a plain **list** (it has positions, not keys: convert with `toset()` or build a map), or a set of objects.

### Addresses and references

| Expression | Value |
|---|---|
| `local_file.member` | A **map** of instances: key → object |
| `local_file.member["ben"]` | One instance |
| `local_file.member["ben"].filename` | One attribute |
| `[for f in local_file.member : f.filename]` | List of an attribute from all instances |
| `{ for k, f in local_file.member : k => f.filename }` | Map key → attribute |
| `values(local_file.member)[*].filename` | Same list, using `values()` + splat |
| `keys(local_file.member)` | The keys |

(Splat `[*]` doesn't work directly on the map; use `values()` first or a `for` expression. Topic 23 covers `for` expressions.)

Shell quoting for CLI addresses: `terraform state show 'local_file.member["ben"]'`.

### Chaining: `for_each` over another `for_each` resource

```hcl
resource "local_file" "badge" {
  for_each = local_file.member                   # one badge per member file
  filename = "${path.module}/${each.key}.badge"
  content  = "badge for ${each.value.filename}\n"  # each.value = the member instance
}
```

Same keys flow through, so instances line up one-to-one.

### `count` vs `for_each`

| Use `count` when... | Use `for_each` when... |
|---|---|
| Instances are **identical** and interchangeable | Each instance is **distinct** (a name, its own settings) |
| It's an **on/off** switch (`? 1 : 0`) | The set of things may change in the middle |
| You need N copies and N is a number | You have a list/map of named items |

Rule of thumb: **if the items have names, use `for_each`.**

---

## Why it exists

Topic 21 showed `count` tracking instances by position, so removing one item replaced everything after it. `for_each` (added in Terraform 0.12.6) tracks by **key**: Terraform matches `local_file.member["ben"]` in config with `local_file.member["ben"]` in state, no matter what else changed. It also gives per-instance settings naturally through `each.value`.

---

## Analogy

**Name-tagged lockers.** Every locker has its owner's name on it. Asha leaves → her locker is emptied; Ben and Chen don't move. A new member Dev arrives → a new locker "dev" appears. Nobody else is affected. And each locker can hold different things (`each.value`): gold members get a towel, basic members don't.

---

## How it works inside

- During plan, Terraform evaluates `for_each` and creates one graph node per **key**. Keys must be known **at plan time**; **values** may be unknown.
- State stores each instance with its `index_key` as the **string key**:

```json
"instances": [
  { "index_key": "asha", "attributes": { ... } },
  { "index_key": "ben",  "attributes": { ... } }
]
```

- Diffing compares **key to key**:
  - key in config, not in state → create,
  - key in state, not in config → destroy,
  - key in both → update/replace only if **its own** arguments changed.
- **Renaming a key** = destroy the old key + create the new one. To rename without destroying, use a `moved` block (topic 40).
- Sensitive values can't be used as `for_each` keys (keys appear in addresses, which are printed).

---

## Code / commands

`~/tf-lab/22-for-each/` with `versions.tf` (`local`).

```hcl
variable "names" {
  type    = list(string)
  default = ["asha", "ben", "chen"]
}

variable "members" {
  type = map(object({
    locker = number
    plan   = optional(string, "basic")
  }))
  default = {
    asha = { locker = 12, plan = "gold" }
    ben  = { locker = 7 }
    chen = { locker = 3, plan = "gold" }
  }
}

# 1. Set of strings
resource "local_file" "name_tag" {
  for_each = toset(var.names)
  filename = "${path.module}/tags/${each.key}.txt"
  content  = "Hello, ${each.value}!\n"
}

# 2. Map of objects: per-instance settings
resource "local_file" "member" {
  for_each = var.members
  filename = "${path.module}/members/${each.key}.txt"
  content  = "locker=${each.value.locker} plan=${each.value.plan}\n"
}

# 3. Chained: one badge per member, only for gold members
resource "local_file" "gold_badge" {
  for_each = { for k, v in var.members : k => v if v.plan == "gold" }
  filename = "${path.module}/badges/${each.key}.txt"
  content  = "GOLD - locker ${each.value.locker} - ${local_file.member[each.key].filename}\n"
}

output "member_files" {
  value = { for k, f in local_file.member : k => f.filename }
}
```

- `{ for k, v in var.members : k => v if v.plan == "gold" }`: a **filtered map**: only gold members. (Topic 23.)
- `local_file.member[each.key]`: look up the matching instance of another `for_each` resource by the same key.

### Apply

```
$ terraform apply -auto-approve
local_file.name_tag["chen"]: Creating...
local_file.member["ben"]: Creating...
local_file.name_tag["asha"]: Creating...
...
local_file.gold_badge["asha"]: Creating...
local_file.gold_badge["chen"]: Creating...
...
Outputs:

member_files = {
  "asha" = "./members/asha.txt"
  "ben" = "./members/ben.txt"
  "chen" = "./members/chen.txt"
}
```

### Remove the first name: only one change

Set `names = ["ben", "chen"]`:

```
$ terraform plan
  # local_file.name_tag["asha"] will be destroyed
  # (because key ["asha"] is not in for_each map)
  - resource "local_file" "name_tag" {
      - content  = "Hello, asha!\n" -> null
      - filename = "./tags/asha.txt" -> null
      ...
    }

Plan: 0 to add, 0 to change, 1 to destroy.
```

Compare with topic 21's "2 to add, 3 to destroy". That's the whole point of `for_each`.

### Change one member's settings: only that member changes

Change `ben = { locker = 7 }` to `ben = { locker = 8, plan = "gold" }`:

```
  # local_file.gold_badge["ben"] will be created
  + resource "local_file" "gold_badge" { ... }

  # local_file.member["ben"] must be replaced
-/+ resource "local_file" "member" {
      ~ content = <<-EOT # forces replacement
          - locker=7 plan=basic
          + locker=8 plan=gold
        EOT
      ...

Plan: 2 to add, 0 to change, 1 to destroy.
```

`asha` and `chen` aren't mentioned at all.

### Keys must be known at plan time

```hcl
resource "random_pet" "p" { count = 2 }

resource "local_file" "bad" {
  for_each = toset(random_pet.p[*].id)     # names unknown until apply
  filename = "${path.module}/${each.key}.txt"
  content  = "x"
}
```

```
╷
│ Error: Invalid for_each argument
│
│   on main.tf line 40, in resource "local_file" "bad":
│   40:   for_each = toset(random_pet.p[*].id)
│     ├────────────────
│     │ random_pet.p is tuple with 2 elements
│
│ The "for_each" set includes values derived from resource attributes that
│ cannot be determined until apply, and so Terraform cannot determine the full
│ set of keys that will identify the instances of this resource.
│
│ When working with unknown values in for_each, it's better to use a map value
│ where the keys are defined statically in your configuration and where only
│ the values contain apply-time results.
╵
```

The fix the message suggests: **static keys, unknown values**:

```hcl
resource "local_file" "good" {
  for_each = { first = random_pet.p[0].id, second = random_pet.p[1].id }
  filename = "${path.module}/${each.key}.txt"     # keys known: "first", "second"
  content  = each.value                            # value unknown until apply: fine
}
```

---

## Hands-on lab

1. Build the project. Apply. Look at `tags/`, `members/`, `badges/`.
2. Remove `"asha"` from `names` and plan: one destroy. Compare with your topic 21 result.
3. Make Ben gold and plan: only Ben's resources change.
4. **Rename a key**: change `chen` to `chen_l` in `members`. Plan: destroy `["chen"]`, create `["chen_l"]`. (Topic 40 shows how to rename without that.)
5. Try `for_each = var.names` (a list, no `toset`). Read the error: `The given "for_each" argument value is unsuitable: the "for_each" argument must be a map, or set of strings, and you have provided a value of type list of string.`
6. Reproduce the `Invalid for_each argument` error and fix it with static keys.
7. Use `terraform console`: `keys(local_file.member)`, `values(local_file.member)[*].content`, `local_file.member["ben"].filename`.
8. `terraform destroy`.

---

## Where you'll use it

- **Named infrastructure**: one S3 bucket per entry in a map, one IAM user per team member, one DNS record per hostname, one subnet per AZ (topic 32).
- **Per-instance settings**: a map of objects describing each service (size, port, replicas).
- **Modules**: `module "service" { for_each = var.services ... }` to stamp out a whole stack per item (topic 44).
- **Data-driven configs**: `for_each = fileset(...)` or a decoded YAML file listing things to create.

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `the "for_each" argument must be a map, or set of strings, and you have provided a value of type list` | Passed a list. | `toset(list)` or build a map with a `for` expression. |
| `Invalid for_each argument ... cannot be determined until apply` | Keys depend on apply-time values. | Static keys, unknown values; or keys from variables. |
| `Invalid for_each set argument ... Sensitive values, or values derived from sensitive values, cannot be used as for_each arguments` | A sensitive value in the keys. | Use non-sensitive keys; keep secrets in values only (or `nonsensitive()` the keys if they're truly not secret). |
| Renaming a key destroys and recreates | Keys are identity. | `moved { from = x["old"]  to = x["new"] }` (topic 40). |
| `Duplicate object key` in a `for` expression building the map | Two items produce the same key. | Choose a unique key, or group with `...` (topic 23). |
| `Missing resource instance key` | Referenced `x.attr` without `["key"]`. | Use `x["key"].attr` or a `for` expression. |

---

## Quick recap

- `for_each` = one instance per map key or set element; `each.key` / `each.value` inside.
- Addresses use **keys**: `x["ben"]`. Removing one key affects only that instance.
- Lists must be converted (`toset`) or turned into maps; keys must be known at plan time (values can be unknown).
- Use `for_each` for **named, distinct** things; `count` for identical copies and on/off switches.
- Renaming a key = destroy + create, unless you add a `moved` block.

### Interview questions

1. **Q: `count` vs `for_each`: when do you use each?**
   A: `count` for N identical instances or a 0/1 switch. `for_each` for a collection of distinct, named items: instances are keyed by name, so adding or removing one doesn't affect the others.
2. **Q: Why can't you pass a list to `for_each`?**
   A: `for_each` needs stable, unique identifiers. A list's elements are identified by position and may repeat. Convert it to a set (`toset`) or a map keyed by a meaningful attribute.
3. **Q: How do you use apply-time values with `for_each`?**
   A: Keep the keys static or derived from known inputs, and put the unknown values in the map's values. Keys must be known at plan time; values don't have to be.
