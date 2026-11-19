# 40 · Refactoring: `moved` and `removed` Blocks

> **Level:** ★★ intermediate
>
> **You will learn:** how to **change resource addresses without destroying anything**, using **`moved` blocks** (1.1+): renaming, `count` → `for_each`, moving into or between modules, and cross-type moves (1.8+, when the provider supports it); and how to **stop managing** a resource **without deleting it**, using **`removed` blocks** (1.7+) with `destroy = false`. You'll compare them with the ⚠️ older `terraform state mv` / `state rm` commands, and fix topic 21's `count` problem for good.
>
> **You need:** topics 21–22 (`count`, `for_each`, keys), 24 (`destroy = false`), 34, 39 (import).
>
> **Part 5 · State in depth** · Previous: [39 · Importing existing infrastructure](39-import.md) · Next: [41 · Replacing resources on purpose](41-replace.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Terraform identifies resources by **address**. Change the address in code (rename, add `for_each`, move into a module) and Terraform sees "old address deleted, new address added": **destroy + create**. A `moved` block tells it: "same object, new name".

```hcl
moved {
  from = aws_instance.web_server
  to   = aws_instance.web
}
```

To stop managing something but **keep it running**:

```hcl
removed {
  from = aws_s3_bucket.old_logs
  lifecycle {
    destroy = false        # forget it; don't delete it
  }
}
```

**Picture it.** Moving house without moving the furniture: you just update the address on the post office's records (`moved`). Or you hand a piece of furniture to a neighbour: it stays in the world, but it's no longer on your inventory (`removed`).

**The one thing to know.** Refactoring = change the code **and** add `moved`/`removed` blocks **in the same PR**, so the plan shows `has moved to` instead of `destroy`/`create`.

**Try it now:** rename a resource in a lab, plan (destroy + create), then add a `moved` block and plan again (`0 to add, 0 to change, 0 to destroy`).

**You've got it if you can:** migrate a `count`-based resource to `for_each` with zero changes, and remove a resource from Terraform without deleting it.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### `moved`

```hcl
moved {
  from = <old address>
  to   = <new address>
}
```

What it can express:

| Refactor | `from` → `to` |
|---|---|
| Rename | `aws_instance.web_server` → `aws_instance.web` |
| Add `count` | `aws_instance.web` → `aws_instance.web[0]` (Terraform does this one **automatically**) |
| `count` index → `for_each` key | `aws_instance.web[0]` → `aws_instance.web["blue"]` |
| Rename a `for_each` key | `aws_instance.web["old"]` → `aws_instance.web["new"]` |
| Into a module | `aws_vpc.main` → `module.network.aws_vpc.main` |
| Rename a module | `module.net` → `module.network` (moves **everything** inside) |
| Between modules | `module.a.aws_s3_bucket.x` → `module.b.aws_s3_bucket.x` |
| Change resource **type** (1.8+) | `null_resource.x` → `terraform_data.x`; `kubernetes_namespace.x` → `kubernetes_namespace_v1.x`; only if the **provider supports** moving between those types |

(A **module** is a reusable folder of `.tf` files; its resources get addresses prefixed with `module.<name>.`. Topic 43 covers modules; here you only need to know that the prefix is part of the address.)

Rules:

- `from` must **not** exist in the configuration any more; `to` must.
- Moves can be **chained** (`a → b`, later `b → c`); Terraform follows the chain, so keep old `moved` blocks when others may still be on an old version.
- **Keep `moved` blocks** after applying, at least until every state that uses the code has been applied. In **shared modules**, keep them for a long time (callers upgrade at different speeds). Removing them is safe only when no state has the old address.
- They can't move to another **state** (another configuration). Use `removed` + `import` for that.

### `removed`

```hcl
removed {
  from = aws_s3_bucket.old_logs           # the resource block is deleted from code
  lifecycle {
    destroy = false                       # false = forget; true = destroy (same as just deleting the block)
  }
}
```

- `from` can be a resource (all instances) or a whole module (`module.legacy`), but not one instance key.
- With `destroy = false`, the plan says the object **will no longer be managed**, and apply removes it from **state only**.
- With `destroy = true` (or omitted), it's an explicit, documented destroy.
- You can delete the `removed` block after it has been applied everywhere.

Since 1.16 there's also a per-resource `lifecycle { destroy = false }` (topic 24): it marks a resource that should be forgotten instead of destroyed whenever Terraform would destroy it.

### ⚠️ The older CLI commands

| Command | Equivalent block | Difference |
|---|---|---|
| `terraform state mv OLD NEW` | `moved` | Edits state **immediately**, no plan, not in code review; teammates/CI with other states don't get the move |
| `terraform state rm ADDR` | `removed` + `destroy = false` | Same: immediate, invisible to review |

The commands are still useful in **emergencies** and for one-off repairs (topic 42). For planned refactoring, use blocks: they're code, reviewed, applied by the pipeline, and work for **every** state that uses the code (e.g. every environment, every caller of a module).

---

## Why it exists

Code needs refactoring: better names, `for_each` instead of `count`, extracting modules, splitting a big configuration, deprecated resource types. Before `moved` (Terraform 1.1), each refactor meant running `terraform state mv` by hand in **every** environment, in the right order, outside code review: error-prone and scary. Before `removed` (1.7), handing a resource over to another team or another configuration meant `state rm` by hand. The blocks make these **declarative and reviewable**.

---

## Analogy

**Your home inventory with the post office.**

- **`moved`** = you relabel a box "Kitchen stuff" → "Kitchen: plates". Same box, same plates, new label. Or you move the box from "garage" to "attic" in the inventory (into a module). Nothing is thrown away and re-bought.
- Without `moved`, the inventory says "garage box missing → buy a new one; attic box unknown → throw it out".
- **`removed` + `destroy = false`** = you give the bookshelf to your neighbour. It still exists, but it's crossed off **your** inventory. (If the neighbour adds it to their inventory, that's `import` in the other configuration.)

---

## How it works inside

```
plan
  1. read all moved blocks; build the chain old → new
  2. for each moved block whose `from` exists in STATE:
        rename that object in the in-memory state to `to`
        (if `to` already exists in state → error)
  3. for each removed block whose `from` exists in state:
        destroy = false → mark the object "forget"
        destroy = true  → mark it "delete"
  4. continue planning as usual: config vs (renamed) refreshed state
  plan output: "has moved to" notes; "will no longer be managed" for forgets
apply
  write the renamed / forgotten entries to state (and any real changes)
```

A move changes **only addresses in state**: no API calls to the real object. If the resource's **arguments** also changed in the same PR, you'll see an update on top of the move, which is fine as long as you expect it.

---

## Code / commands

### 1. Fixing topic 21's `count` problem

`~/tf-lab/40-refactor/` with the `local` provider. Start with the old code and apply it:

```hcl
variable "names" {
  type    = list(string)
  default = ["asha", "ben", "chen"]
}

resource "local_file" "member" {
  count    = length(var.names)
  filename = "${path.module}/member-${count.index}.txt"
  content  = "member: ${var.names[count.index]}\n"
}
```

```bash
terraform init && terraform apply -auto-approve
```

Refactor to `for_each`, but keep the **same file names and contents** so only the addresses change. The real migration:

```hcl
locals {
  # name → old index, so file names stay identical during the migration
  members = { for i, n in var.names : n => i }
}

resource "local_file" "member" {
  for_each = local.members
  filename = "${path.module}/member-${each.value}.txt"
  content  = "member: ${each.key}\n"
}

moved {
  from = local_file.member[0]
  to   = local_file.member["asha"]
}
moved {
  from = local_file.member[1]
  to   = local_file.member["ben"]
}
moved {
  from = local_file.member[2]
  to   = local_file.member["chen"]
}
```

```
$ terraform plan
local_file.member["asha"]: Refreshing state... [id=...]
...
Terraform will perform the following actions:

  # local_file.member[0] has moved to local_file.member["asha"]
    resource "local_file" "member" {
        id       = "a83f..."
        # (9 unchanged attributes hidden)
    }

  # local_file.member[1] has moved to local_file.member["ben"]
    ...
  # local_file.member[2] has moved to local_file.member["chen"]
    ...

Plan: 0 to add, 0 to change, 0 to destroy.
```

- `has moved to`: state addresses will change; nothing else.
- `0 to add, 0 to change, 0 to destroy`: a perfect refactor.

Apply. Now remove `"asha"` from `names` and plan: only `local_file.member["asha"]` is destroyed. Topic 21's index shift is gone forever.

Without the `moved` blocks, the same refactor would plan `3 to add, 3 to destroy`. Try it: comment them out and plan (don't apply).

### 2. A simple rename

```hcl
resource "local_file" "readme" {          # was: resource "local_file" "notes"
  filename = "${path.module}/README.txt"
  content  = "hello\n"
}

moved {
  from = local_file.notes
  to   = local_file.readme
}
```

### 3. Into a module (preview of topic 43)

```hcl
# before: resource "aws_vpc" "main" { ... } in the root
# after:  module "network" { source = "./modules/network" }   (which contains aws_vpc.main)

moved {
  from = aws_vpc.main
  to   = module.network.aws_vpc.main
}
```

Same mechanism; the address just gains the `module.network.` prefix.

### 4. Cross-type move (1.8+; this pair 1.9+)

Replacing the deprecated `null_resource` (topic 52) with the built-in `terraform_data`:

```hcl
resource "terraform_data" "bootstrap" {
  triggers_replace = [var.version]
}

moved {
  from = null_resource.bootstrap
  to   = terraform_data.bootstrap
}
```

This works because `terraform_data` knows how to convert a `null_resource`'s state. Unsupported combinations fail with `Unsupported cross-resource-type move`.

### 5. `removed`: hand over without deleting

Suppose `local_file.legacy_report` should now be owned by another team's configuration. Delete its resource block and add:

```hcl
removed {
  from = local_file.legacy_report
  lifecycle {
    destroy = false
  }
}
```

```
$ terraform plan
  # local_file.legacy_report will no longer be managed by Terraform, but will not be destroyed
  # (destroy = false is set in the configuration)
 . resource "local_file" "legacy_report" {
        id       = "77c1..."
        # (9 unchanged attributes hidden)
    }

Plan: 0 to add, 0 to change, 0 to destroy, 1 to forget.

╷
│ Warning: Some objects will no longer be managed by Terraform
│
│ If you apply this plan, Terraform will discard its tracking information for
│ the following objects, but it will not delete them:
│  - local_file.legacy_report
│
│ After applying this plan, Terraform will no longer manage these objects. You
│ will need to import them into Terraform to manage them again.
╵
```

- `.` symbol and `1 to forget`: removed from state, the file stays on disk.
- The other team then adds an `import` block (topic 39) in **their** configuration. That's how you **move a resource between states**: `removed` (forget) here + `import` there.

Without the `removed` block, deleting the resource block would plan `1 to destroy`.

### The CLI equivalents (for emergencies)

```bash
terraform state mv 'local_file.member[0]' 'local_file.member["asha"]'
terraform state rm local_file.legacy_report
```

```
Move "local_file.member[0]" to "local_file.member[\"asha\"]"
Successfully moved 1 object(s).

Removed local_file.legacy_report
Successfully removed 1 resource instance(s).
```

Immediate, no plan. Back up first (`terraform state pull > backup.tfstate`). Topic 42 covers these for repairs.

---

## Hands-on lab

1. Build the `count` version, apply, then migrate to `for_each` with `moved` blocks. Reach `0 to add, 0 to change, 0 to destroy`. Apply.
2. Remove `"asha"`: confirm only one destroy.
3. Rename a resource with and without `moved`; compare the plans.
4. Use `removed` with `destroy = false` on one file; apply; confirm the file still exists and `terraform state list` no longer shows it. Then `import` it back (topic 39).
5. Chain moves: rename `a → b`, apply; later rename `b → c`, keeping both `moved` blocks; check a **fresh** copy of the old state (from your `state pull` backup) also plans cleanly with the chain.
6. `terraform destroy`.

---

## Where you'll use it

- **Renames and cleanups** in every long-lived codebase.
- **`count` → `for_each` migrations**, one of the most common refactors.
- **Extracting modules** from a big root configuration, and **splitting states** (removed here + import there).
- **Provider upgrades** that rename resource types (`kubernetes_*` → `*_v1`, `null_resource` → `terraform_data`).
- **Module authors**: `moved` blocks inside a module let every caller upgrade without touching their state.

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| Refactor plans destroy + create | No `moved` block (or a typo in it). | Add/fix `moved`; plan until `0 to add/destroy`. |
| `Moved object still exists ... the configuration still has a resource at the old address` | `from` is still declared in the code. | Remove/rename the old block. |
| `Ambiguous move statements` / `Cyclic dependency in move statements` | Two moves to the same `to`, or a loop. | One clear chain per object. |
| `Unsupported cross-resource-type move` | The target type can't convert the source type's state. | Use `removed` (forget) + `import` into the new type instead. |
| Plan shows a move **and** an update | The refactor also changed arguments (e.g. a file name derived from the key). | Fine if intended; otherwise keep arguments identical during the move and change them in a separate PR. |
| Removed a `moved` block too early; an environment now plans destroy + create | That environment was still on the old address. | Put the `moved` block back. |
| `removed` block: `Removed resource still exists` | The resource block is still in the code. | Delete the resource block. |

---

## Quick recap

- Addresses are identity. Changing them in code means destroy + create unless you add **`moved { from, to }`**.
- `moved` handles renames, `count` → `for_each`, modules, and (1.8+, provider permitting) type changes. Keep the blocks until every state is migrated.
- **`removed { from  lifecycle { destroy = false } }`** forgets a resource without deleting it; pair it with `import` elsewhere to move between states.
- ⚠️ `terraform state mv` / `state rm` do the same immediately, without review: emergencies only.

### Interview questions

1. **Q: How do you rename a resource without destroying it?**
   A: Rename it in the code and add a `moved` block from the old address to the new one. The plan shows "has moved to" with no changes, and apply updates state.
2. **Q: How do you migrate from `count` to `for_each` safely?**
   A: Switch the resource to `for_each` and add one `moved` block per instance mapping each index to its key, keeping the arguments the same, so the plan shows only moves.
3. **Q: How do you stop managing a resource with Terraform without deleting it?**
   A: Delete its resource block and add a `removed` block with `lifecycle { destroy = false }`. Apply removes it from state only. (The old way was `terraform state rm`.)
