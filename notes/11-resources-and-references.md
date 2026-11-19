# 11 · Resources and References

> **Level:** ★ foundation
>
> **You will learn:** what a resource is in depth: its **type**, **local name** and **address**; **arguments** (you set them) vs **attributes** (you can read them, some are computed by the provider); how one resource **references** another, and how that creates a dependency; how `(known after apply)` spreads through a plan; the **create / read / update / delete** life cycle each provider implements; and the **`random`** provider, which you'll use in many labs.
>
> **You need:** topics 07–10.
>
> **Part 1 · First steps and the core loop** · Previous: [10 · Providers](10-providers.md) · Next: [12 · State: the basics](12-state-basics.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **resource** block = "this object should exist". Its **address** is `TYPE.NAME`. You **set arguments**; after creation you can **read attributes**, including ones the provider fills in (like an `id`). Using one resource's attribute in another resource is a **reference**, and it tells Terraform which one to create first.

```hcl
resource "random_pet" "server" {        # makes a random name like "brave-otter"
  length = 2
}

resource "local_file" "motd" {
  filename = "${path.module}/motd.txt"
  content  = "Welcome to ${random_pet.server.id}\n"   # reference → random_pet first
}
```

**Picture it.** A bakery order: "a cake, and a card that says the cake's name". The card can't be written until the cake is named. Terraform sees that from the reference.

**The one thing to know.** `TYPE.NAME.ATTRIBUTE` is how you read any value from any resource. If the value isn't known until creation, the plan shows `(known after apply)`.

**Try it now:** run the code above (`init`, `apply`), `cat motd.txt`, then `apply` again: the name stays the same. Random values are generated **once** and kept in state.

**You've got it if you can:** say which values in a resource are arguments and which are computed attributes, and why `motd` waits for `server`.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Anatomy

```hcl
resource "local_file" "motd" {
#        └── TYPE ──┘ └NAME┘
  filename = "..."           # argument
  content  = "..."           # argument
}
```

- **Type** (`local_file`): decided by the provider. The prefix before the first `_` picks the provider (`local`).
- **Local name** (`motd`): your choice. Unique **per type** in a module. Use `snake_case` and describe the role (`web`, `logs`, `main`), not the type again (`aws_s3_bucket.bucket` is a weak name).
- **Address** (`local_file.motd`): how Terraform names it in plans, state and commands. Later you'll see longer addresses: `local_file.motd[0]` (with `count`), `local_file.motd["a"]` (with `for_each`), `module.app.local_file.motd` (inside a module).

### Arguments vs attributes

| | Arguments | Attributes |
|---|---|---|
| Who sets them | **You**, in the block | The provider reports them after reading/creating |
| Example (`local_file`) | `filename`, `content`, `file_permission` | `id`, `content_md5`, plus all arguments |
| Docs section | "Argument Reference" | "Attribute Reference" |

Every argument is also readable as an attribute. Attributes that only the provider sets are called **computed**. Some are **optional + computed**: if you don't set them, the provider picks a value (for example an AWS VM's private IP).

### References

Any expression can read attributes from another resource:

```hcl
random_pet.server.id              # TYPE.NAME.ATTRIBUTE
random_pet.server                 # the whole object (all attributes)
```

A reference does two jobs:

1. **Gives a value.**
2. **Creates a dependency**: Terraform must create (or update) `random_pet.server` before `local_file.motd`, and destroy them in the reverse order. Topic 20 covers dependencies fully.

### The `random` provider

`hashicorp/random` creates random values **inside Terraform**, with no external system. They're stored in state, so they **stay the same** on every run until something forces a new one.

| Resource | Produces | Typical use |
|---|---|---|
| `random_pet` | Readable names: `brave-otter` | Lab names, unique-ish suffixes |
| `random_string` | Random characters | Short unique suffixes |
| `random_id` | Random bytes, as hex/base64 | Globally unique S3 bucket names |
| `random_integer` | A number in a range | Picking a port, a subnet number |
| `random_password` | Like `random_string` but its `result` is **sensitive** | Initial database passwords (see topic 51 for the caveats) |
| `random_shuffle` | A shuffled list | Spreading things across zones |

**`keepers`**: a map argument on most `random_*` resources. When any value in it changes, a **new** random value is generated (the resource is replaced). Use it to say "make a new name whenever the image changes".

### The life cycle a provider implements

For every resource type, the provider implements **CRUD** (Create, Read, Update, Delete):

| Operation | When core calls it | Example: `local_file` |
|---|---|---|
| **Create** | `+` in the plan | Write the file |
| **Read** | Every refresh (start of plan/apply) | Check the file still exists with the same content |
| **Update** | `~` in the plan | (none: every argument forces replacement) |
| **Delete** | `-` in the plan | Remove the file |

Which arguments can be updated in place and which force replacement is decided **per argument** by the provider. That's why you read `# forces replacement` in plans.

---

## Why it exists

- **References** let you write values **once**. The VM's ID flows into the DNS record, the DNS name flows into the certificate, and so on. No copy-pasting IDs from the console.
- **Computed attributes** exist because many values only exist after creation (IDs, IPs, ARNs = Amazon Resource Names).
- **Dependencies from references** mean you almost never write the order yourself.

---

## Analogy

**A bakery order system.**

- Each **resource** is an order slip: type ("cake"), your reference name ("birthday"), and options you choose (arguments: flavour, size).
- After the bakery makes it, the slip gets **filled in** by the bakery (computed attributes): order number, collection shelf.
- The **card** order says "write the cake's order number on it" (a reference). The bakery automatically makes the cake first.
- The `random_pet` is the bakery's **name generator**: it names the cake once and writes it on the slip; asking again gives the same name, unless you change the "keepers" ("new name if the flavour changes").

---

## How it works inside

How unknown values flow through a plan:

```
plan time:
  random_pet.server.id          = (known after apply)      ← doesn't exist yet
  local_file.motd.content       = "Welcome to ${random_pet.server.id}"
                                = (known after apply)      ← depends on an unknown
apply time (graph order):
  1. provider(random).Create(random_pet.server)  → id = "brave-otter"  → saved to state
  2. core re-evaluates local_file.motd.content   → "Welcome to brave-otter\n"
  3. provider(local).Create(local_file.motd)     → file written         → saved to state
```

- Unknown values **spread**: anything built from an unknown is unknown.
- Core gives the provider the final values only at apply time, in graph order.
- If an **unknown** value lands in an argument that **forces replacement**, the plan must assume it might change, so it shows a replacement. That's one source of "why is it replacing?!" surprises (topic 24).

---

## Code / commands

`~/tf-lab/11-resources/versions.tf`:

```hcl
terraform {
  required_version = ">= 1.14"
  required_providers {
    local  = { source = "hashicorp/local", version = "~> 2.5" }
    random = { source = "hashicorp/random", version = "~> 3.7" }
  }
}
```

`main.tf`:

```hcl
resource "random_pet" "server" {
  length = 2
  keepers = {
    image = "nginx:1.29"        # change this → new name
  }
}

resource "random_password" "db" {
  length  = 20
  special = true
}

resource "local_file" "motd" {
  filename = "${path.module}/motd.txt"
  content  = <<-EOT
    Welcome to ${random_pet.server.id}
    Image:     ${random_pet.server.keepers.image}
  EOT
}
```

### Plan

```
$ terraform plan

Terraform will perform the following actions:

  # local_file.motd will be created
  + resource "local_file" "motd" {
      + content              = (known after apply)
      + directory_permission = "0777"
      + file_permission      = "0777"
      + filename             = "./motd.txt"
      + id                   = (known after apply)
      ...
    }

  # random_password.db will be created
  + resource "random_password" "db" {
      + bcrypt_hash = (sensitive value)
      + id          = (known after apply)
      + length      = 20
      + lower       = true
      + min_lower   = 0
      ...
      + result      = (sensitive value)
      + special     = true
      + upper       = true
    }

  # random_pet.server will be created
  + resource "random_pet" "server" {
      + id        = (known after apply)
      + keepers   = {
          + "image" = "nginx:1.29"
        }
      + length    = 2
      + separator = "-"
    }

Plan: 3 to add, 0 to change, 0 to destroy.
```

| Line | Meaning |
|---|---|
| `content = (known after apply)` | It references `random_pet.server.id`, which doesn't exist yet. |
| `lower = true`, `min_lower = 0` | Arguments you didn't set: **defaults** from the provider's schema. |
| `result = (sensitive value)` | The provider marks the password as sensitive, so it's hidden in output. **It's still stored in state in plain text** (topic 51). |
| `separator = "-"` | Default: words joined with `-`. |

The plan **lists** resources alphabetically by address. The order they're **created** in comes from the dependency graph (next lines).

### Apply

```
$ terraform apply -auto-approve
random_password.db: Creating...
random_pet.server: Creating...
random_pet.server: Creation complete after 0s [id=brave-otter]
random_password.db: Creation complete after 0s [id=none]
local_file.motd: Creating...
local_file.motd: Creation complete after 0s [id=9c1f...]

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
```

- `random_password.db` and `random_pet.server` start **at the same time**: no dependency between them, so Terraform runs them in parallel.
- `local_file.motd` starts only after `random_pet.server` completes.
- `[id=none]`: `random_password` uses the literal ID `none`; its value is in `result`.

```bash
cat motd.txt
terraform state show random_pet.server
```

```
Welcome to brave-otter
Image:     nginx:1.29

# random_pet.server:
resource "random_pet" "server" {
    id        = "brave-otter"
    keepers   = {
        "image" = "nginx:1.29"
    }
    length    = 2
    separator = "-"
}
```

`terraform state show <address>` prints everything Terraform knows about one resource: arguments **and** attributes.

### `keepers` → replacement

Change `image = "nginx:1.29"` to `"nginx:1.30"`:

```
  # local_file.motd must be replaced
-/+ resource "local_file" "motd" {
      ~ content = <<-EOT # forces replacement
          - Welcome to brave-otter
          - Image:     nginx:1.29
        EOT -> (known after apply)
      ...
    }

  # random_pet.server must be replaced
-/+ resource "random_pet" "server" {
      ~ id      = "brave-otter" -> (known after apply)
      ~ keepers = { # forces replacement
          ~ "image" = "nginx:1.29" -> "nginx:1.30"
        }
        # (2 unchanged attributes hidden)
    }

Plan: 2 to add, 0 to change, 2 to destroy.
```

One change **cascaded**: the new pet name is unknown, and `content` (which forces replacement) depends on it. Recognising a cascade like this in a plan is a key production skill.

### Reading attributes with `terraform console`

```bash
terraform console
> random_pet.server.id
"brave-otter"
> random_password.db.result
(sensitive value)
> nonsensitive(random_password.db.result)
"q9!Lx...<20 chars>"
> exit
```

`terraform console` evaluates expressions against the current state (topic 18). `nonsensitive()` removes the sensitive mark. Fine in a lab, but a smell in real code.

---

## Hands-on lab

1. Create the project above in `~/tf-lab/11-resources`. `init`, `plan`, read every line, `apply`.
2. Run `apply` again → `No changes`. The pet name is stable.
3. Change the `keepers` image and plan. Find the `# forces replacement` lines and explain the cascade.
4. Add `random_integer`:

   ```hcl
   resource "random_integer" "port" {
     min = 8000
     max = 8999
   }
   ```
   And add `Port: ${random_integer.port.result}` to the `motd` content. Apply. Which resources change?
5. Use `terraform state list` and `terraform state show` on each resource.
6. Open the docs page for `random_pet` on the Registry and match "Argument Reference" and "Attribute Reference" with what you saw.
7. `terraform destroy`. Watch the order: `motd` is destroyed **before** `server` (reverse dependency order).

---

## Where you'll use it

- **References everywhere**: subnet ID → VM, VM IP → DNS record, role ARN → policy.
- **`random_id` / `random_pet`** to make globally unique names (S3 bucket names must be unique across **all** AWS accounts).
- **`keepers`** to rotate a value when something else changes.
- **`terraform state show`** to see the real ID and attributes of something during an incident.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `Reference to undeclared resource` | Typo in a reference, or the resource is in another configuration or module. | Fix the name; across configurations use data sources (topic 19) or outputs. |
| `Unsupported attribute: This object has no argument, nested block, or exported attribute named "ip"` | That attribute doesn't exist on this type. | Check "Attribute Reference" in the docs, or `terraform state show`. |
| Everything downstream shows `(known after apply)` and is replaced | An upstream value became unknown (it's being replaced) and feeds arguments that force replacement. | Find the **first** resource in the cascade; fix that cause (topic 24). |
| Random value changed unexpectedly | Something in `keepers` changed, or the resource was replaced/recreated (state lost). | Check `keepers`; protect state (Part 5). |
| `Error: Invalid reference` when using a resource name with a dash first, or a number first | Identifiers can't start with a digit. | Rename: `web_1`, not `1_web`. |

---

## Quick recap

- A resource = TYPE + NAME → address `TYPE.NAME`. The type's prefix picks the provider.
- **Arguments** you set; **attributes** you read; **computed** attributes come from the provider.
- A **reference** (`TYPE.NAME.ATTR`) gives a value **and** creates a dependency.
- Unknown values show as `(known after apply)` and spread to anything built from them.
- `random_*` values are generated once and kept in state; `keepers` forces a new value.

### Interview questions

1. **Q: What is the difference between arguments and attributes?**
   A: Arguments are values you set in the resource block. Attributes are values you can read from the resource, including all arguments and computed values the provider sets, like IDs and IPs.
2. **Q: What does `(known after apply)` mean?**
   A: The value can't be determined until the provider creates or updates the object, such as a generated ID. Anything derived from it is also unknown at plan time.
3. **Q: How does Terraform decide the order to create resources?**
   A: From references: if resource B uses an attribute of A, B depends on A, so A is created first. Independent resources are created in parallel. `depends_on` adds hidden dependencies (topic 20).
