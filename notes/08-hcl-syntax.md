# 08 · HCL Syntax

> **Level:** ★ foundation
>
> **You will learn:** how HCL (HashiCorp Configuration Language) is built: **blocks**, **labels**, **arguments**, **nested blocks**, **expressions**, **comments**, strings and `${...}` interpolation, the basic value types, how Terraform **merges every `.tf` file** in a folder, the conventional file layout, every top-level block type you'll meet (with the topic that teaches it), and the two syntax tools: `terraform fmt` and `terraform validate`.
>
> **You need:** topic 07 (you've run a project).
>
> **Part 1 · First steps and the core loop** · Previous: [07 · Your first Terraform project](07-first-terraform-project.md) · Next: [09 · The core workflow commands](09-core-workflow-commands.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** HCL is made of **blocks**. A block has a **type**, zero or more **labels**, and a **body** in `{ }`. Inside the body are **arguments** (`name = value`) and maybe **nested blocks**.

```hcl
resource "docker_container" "web" {   # type: resource · labels: "docker_container", "web"
  name  = "web"                       # argument
  image = "nginx:1.29"                # argument

  ports {                             # nested block (no "=")
    internal = 80
    external = 8080
  }
}
```

**Picture it.** A form with sections. The block type is the form's title, labels fill in "which one", arguments are the fields, nested blocks are sub-sections.

**The one thing to know.** Terraform reads **all `.tf` files in the folder as one**. File names are only for humans. Run `terraform fmt` to format and `terraform validate` to check syntax.

**Try it now:** in `~/tf-lab/08-hcl`, write a block with bad indentation and a misspelled argument, then run `terraform fmt` and `terraform validate` (after `terraform init`).

**You've got it if you can:** point at any line of `.tf` code and say whether it's a block, a label, an argument, or a nested block.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Blocks

```
<BLOCK TYPE> "<LABEL 1>" "<LABEL 2>" {
  <IDENTIFIER> = <EXPRESSION>      # argument
  <NESTED BLOCK TYPE> {            # nested block
    ...
  }
}
```

- The **block type** says what kind of thing this is: `resource`, `variable`, `output`...
- **Labels** are quoted strings that name it. How many labels a block takes depends on its type: `resource` takes two (type and name), `variable` takes one (name), `terraform` and `locals` take none.
- The **body** holds arguments and nested blocks.

### Arguments vs nested blocks

| | Argument | Nested block |
|---|---|---|
| Looks like | `tags = { Name = "web" }` | `ports { internal = 80 }` |
| Has `=` | **Yes** | **No** |
| Can repeat? | No (one value per name) | Often yes (several `ports { }` blocks) |
| Who decides which one? | The **provider's schema** (the resource's docs page) | Same |

That last row matters: `tags = {...}` is a map argument; `ports {...}` is a nested block. They look similar, but writing `tags { ... }` or `ports = { ... }` is an error. Check the resource's documentation page.

### Identifiers and comments

**Identifiers** (argument names, block names, labels you invent) use letters, digits, `_` and `-`, and can't start with a digit. The convention is `snake_case`: `web_server`, not `webServer`.

```hcl
# a comment (preferred style)
// also a comment
/* a
   multi-line comment */
```

### Values (types at a glance; topic 17 goes deep)

```hcl
name       = "web"                          # string (always double quotes)
replicas   = 3                              # number
enabled    = true                           # bool: true / false (no quotes)
zones      = ["a", "b", "c"]                # list (ordered)
tags       = { Team = "platform", Env = "dev" }  # map / object
nothing    = null                           # "not set": use the default
```

### Strings and interpolation

```hcl
greeting = "Hello, ${var.name}!"            # ${ } inserts an expression's value
path     = "${path.module}/hello.txt"
literal  = "Cost: $${amount}"               # $${ } prints a literal ${
script   = <<-EOT
    #!/bin/bash
    echo "Hello from ${var.name}"
  EOT
```

- `${ ... }` = **interpolation**: put an expression's value into a string.
- `<<-EOT ... EOT` = a **heredoc**: a multi-line string. The `-` strips leading indentation. `EOT` is just a marker word; any word works.

### References

Arguments can refer to other things by **address**:

```hcl
content = local_file.hello.content    # <RESOURCE TYPE>.<NAME>.<ATTRIBUTE>
name    = var.app_name                # an input variable (topic 14)
region  = local.region                # a local value (topic 16)
```

A reference also creates a **dependency**: Terraform makes sure the referenced thing exists first (topic 20).

### The top-level block types (your map for the whole course)

| Block | Labels | Purpose | Topic |
|---|---|---|---|
| `terraform { }` | 0 | Settings: required versions, providers, backend | 10, 35 |
| `provider "aws" { }` | 1 | Configure a provider (region, credentials...) | 10 |
| `resource "TYPE" "NAME" { }` | 2 | Something Terraform creates and manages | 11 |
| `data "TYPE" "NAME" { }` | 2 | Something Terraform only reads | 19 |
| `variable "NAME" { }` | 1 | An input | 14 |
| `output "NAME" { }` | 1 | A value to print / pass out | 15 |
| `locals { }` | 0 | Named expressions | 16 |
| `module "NAME" { }` | 1 | Use a reusable group of resources | 43 |
| `check "NAME" { }` | 1 | A health assertion | 25 |
| `import { }` | 0 | Adopt an existing object | 39 |
| `moved { }` / `removed { }` | 0 | Refactor without destroying | 40 |
| `ephemeral "TYPE" "NAME" { }` | 2 | Temporary values never stored in state | 51 |
| `action "TYPE" "NAME" { }` | 2 | Provider-defined operations (1.14+) | 67 |

### How files are combined

- Terraform loads **every file ending in `.tf`** (and `.tf.json`) in the current folder. **Not sub-folders** (those are modules, topic 43).
- File order doesn't matter. Declaration order inside files doesn't matter either: you can reference a resource defined further down or in another file.
- Two blocks with the same address (two `resource "local_file" "hello"`) are an error, even in different files.
- Files named `override.tf` or `*_override.tf` are merged **on top of** the others. Avoid them; they confuse readers.
- `.tf.json` files hold the same thing in JSON, for when a program **generates** configuration.

### Conventional file layout

| File | Holds |
|---|---|
| `versions.tf` (or `terraform.tf`) | `terraform { required_version, required_providers }` |
| `providers.tf` | `provider` blocks |
| `main.tf` | Resources (big projects split by area: `network.tf`, `compute.tf`...) |
| `variables.tf` | `variable` blocks |
| `outputs.tf` | `output` blocks |
| `locals.tf` | `locals` (optional) |
| `terraform.tfvars` | Variable **values** (topic 14). Not a `.tf` file: it holds no blocks. |

---

## Why it exists

HashiCorp wanted a language that is:

- **Easy to read** for people reviewing changes: more readable than JSON, more structured than YAML.
- **Declarative**: no step-by-step logic; you describe things.
- **Expressive enough**: variables, references, functions and loops, without being a full programming language.
- **Machine-friendly**: has a JSON form for tools that generate config.

YAML (which you know from Kubernetes and GitHub Actions) has no native references or expressions. That's why GitHub Actions bolted on `${{ }}`. HCL has expressions built in.

---

## Analogy

A **tax form**:

- The **form** (block type: `resource`) with its **form number and your name** (labels: `"local_file" "hello"`).
- **Fields** (arguments): `filename = ...`. Each field appears once.
- **Repeated sections** (nested blocks): "Dependant 1", "Dependant 2", each a sub-form without an `=`.
- **"See line 12 of form B"** (references): one form's field points at another's.
- The tax office reads **all** your forms together, in any order (all `.tf` files in the folder).

---

## How it works inside

What Terraform does with your text before anything else:

```
all *.tf files in folder
      │ 1. parse each file with the HCL parser → syntax tree
      │    (unclosed braces, bad quotes → "Error: Invalid block definition"/"Unclosed configuration block")
      ▼
merge into ONE configuration
      │ 2. check block types and labels are known
      │    (duplicate addresses → "Error: Duplicate resource ... configuration")
      ▼
decode each block against its schema
      │ 3. terraform/variable/output... → Terraform's own schema
      │    resource/data → the PROVIDER's schema (needs init: the provider tells core
      │    which arguments exist, which are required, their types)
      │    (unknown argument → "Unsupported argument"; missing → "Missing required argument")
      ▼
expressions are evaluated later, during plan (step 4 of topic 13)
```

That's why `terraform validate` needs `init` first: to check resource arguments, it has to ask the provider for its **schema**.

---

## Code / commands

### `terraform fmt`: format the code

Messy input (`main.tf`):

```hcl
resource "local_file" "hello" {
filename= "${path.module}/hello.txt"
    content ="Hello!\n"
  file_permission = "0644"
}
```

```bash
terraform fmt -diff        # show what it changes, and fix the file
```

```
main.tf
--- old/main.tf
+++ new/main.tf
@@ -1,5 +1,5 @@
 resource "local_file" "hello" {
-filename= "${path.module}/hello.txt"
-    content ="Hello!\n"
+  filename        = "${path.module}/hello.txt"
+  content         = "Hello!\n"
   file_permission = "0644"
 }
```

- First line: the files it changed.
- Two-space indentation and **aligned `=` signs** are the standard style.

Useful flags:

| Command | Does |
|---|---|
| `terraform fmt` | Formats `.tf` files in this folder in place |
| `terraform fmt -recursive` | Also sub-folders |
| `terraform fmt -check` | Changes nothing; **non-zero exit code** if anything needs formatting. Use in CI (topic 53). |
| `terraform fmt -diff` | Shows the changes |

### `terraform validate`: check syntax and schema

Broken `main.tf`:

```hcl
resource "local_file" "hello" {
  contnt = "Hello!\n"
}
```

```bash
terraform init -backend=false     # get the provider schema, no backend needed for validate
terraform validate
```

```
╷
│ Error: Missing required argument
│
│   on main.tf line 1, in resource "local_file" "hello":
│    1: resource "local_file" "hello" {
│
│ The argument "filename" is required, but no definition was found.
╵
╷
│ Error: Unsupported argument
│
│   on main.tf line 2, in resource "local_file" "hello":
│    2:   contnt = "Hello!\n"
│
│ An argument named "contnt" is not expected here. Did you mean "content"?
╵
```

How to read every Terraform error:

- The `╷ │ ╵` box = one error.
- `Error: <summary>`: the kind of problem.
- `on main.tf line 2, in resource "local_file" "hello":`: **file, line and block**. Go there first.
- The code line with a line number.
- The detail sentence, often with a suggestion (`Did you mean "content"?`).

Fixed and validated:

```
Success! The configuration is valid.
```

`validate` checks **syntax, types and references inside the config**. It does **not** check credentials, whether a name is free in the cloud, or anything that needs an API call. `plan` does that.

### Arguments vs nested block mistake

```hcl
resource "local_file" "hello" {
  filename = "a.txt"
  content  = "x"
  tags {                      # local_file has no "tags", and if it did,
    Name = "x"                # tags would be an argument: tags = { ... }
  }
}
```

```
│ Error: Unsupported block type
│
│   on main.tf line 4, in resource "local_file" "hello":
│    4:   tags {
│
│ Blocks of type "tags" are not expected here.
```

---

## Hands-on lab

1. Set up: `mkdir -p ~/tf-lab/08-hcl && cd ~/tf-lab/08-hcl`.
2. Create **three files** to prove Terraform merges them:

   `versions.tf`:
   ```hcl
   terraform {
     required_providers {
       local = {
         source  = "hashicorp/local"
         version = "~> 2.5"
       }
     }
   }
   ```

   `main.tf`:
   ```hcl
   resource "local_file" "greeting" {
     filename = "${path.module}/greeting.txt"
     content  = <<-EOT
       Hello from HCL.
       This file's sibling says: ${local_file.signature.content}
     EOT
   }
   ```

   `signature.tf`:
   ```hcl
   # defined in another file, referenced above: order doesn't matter
   resource "local_file" "signature" {
     filename = "${path.module}/signature.txt"
     content  = "-- your Terraform lab"
   }
   ```
3. `terraform init`, `terraform validate`, `terraform apply`. `cat greeting.txt`. Notice the plan creates `signature` **before** `greeting`: the reference made a dependency.
4. Mess up the indentation in `main.tf`. Run `terraform fmt -check; echo "exit=$?"`, then `terraform fmt -diff`.
5. Break things one at a time and read each error box: misspell `filename`; delete a closing `}`; duplicate the `signature` block into `main.tf`; use `tags { }`.
6. `terraform destroy`.

---

## Where you'll use it

- **Reading other people's code** in every Terraform job. Being fast at "block / argument / reference" is half the skill.
- **CI**: `terraform fmt -check` and `terraform validate` are the first two steps of every Terraform pipeline (topics 53, 57).
- **Code review**: standard formatting keeps diffs small and about real changes.

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `Unsupported argument` / `Did you mean...` | Typo, or the argument doesn't exist in this provider version. | Fix the spelling; check the docs for **your** provider version. |
| `Missing required argument` | A required field isn't set. | Add it. The docs mark required arguments. |
| `Unsupported block type` | Used `name { }` where the schema wants `name = { }`, or the block doesn't exist. | Check the docs: argument (with `=`) or block (without). |
| `Duplicate resource "x" "y" configuration` | Same address declared twice, maybe in two files. | Rename or delete one. `grep -rn 'resource "x" "y"' *.tf`. |
| `Unclosed configuration block` / `Argument or block definition required` | Missing `}` or a stray character. | Look at the line **before** the one reported. |
| `Invalid character` / `Invalid quoted string` | Single quotes `'web'` or smart quotes from a document. | HCL strings need straight double quotes `"web"`. |
| `Could not load plugin` on `validate` | No `init` yet. | `terraform init` (or `init -backend=false`). |

---

## Quick recap

- HCL = **blocks** (type + labels + body) containing **arguments** (`name = value`) and **nested blocks** (`name { }`).
- The provider's schema decides what's an argument and what's a block.
- All `.tf` files in one folder form **one** configuration; order and file names don't matter.
- `${ }` interpolates, `<<-EOT` makes multi-line strings, references like `TYPE.NAME.ATTR` create dependencies.
- `terraform fmt` formats, `terraform validate` checks syntax and schema (no API calls).

### Interview questions

1. **Q: What's the difference between an argument and a block in HCL?**
   A: An argument assigns a value with `=` and appears once. A block has no `=`, has its own body, and may repeat. Which one a setting is depends on the provider's schema.
2. **Q: Does the order of files or blocks matter in Terraform?**
   A: No. Terraform merges all `.tf` files in the directory and works out the order from references (the dependency graph).
3. **Q: What does `terraform validate` check, and what doesn't it?**
   A: It checks syntax, argument names and types, and internal references, using the provider schemas. It doesn't call any APIs, so it can't catch permission problems, name conflicts or invalid cloud values. That's `plan`'s job.
