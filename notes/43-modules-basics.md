# 43 · Modules: The Basics

> **Level:** ★★ intermediate
>
> **You will learn:** what a **module** is (any folder of `.tf` files), the **root module** vs **child modules**, the `module` block (`source`, inputs, `version`, meta-arguments), how values go **in** (variables) and come **out** (outputs as `module.NAME.OUTPUT`), what `terraform init` does with modules (`.terraform/modules/`), and how module resources are **addressed in state** (`module.app.local_file.x`).
>
> **You need:** topics 14–16 (variables, outputs, locals), 20 (graph), 21–22 (`count`, `for_each`), 40 (`moved` into modules).
>
> **Part 6 · Modules** · Previous: [42 · State surgery and recovery](42-state-recovery.md) · Next: [44 · Writing reusable modules](44-writing-modules.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **module** is a folder of `.tf` files with **inputs** (variables) and **outputs**. You've been writing one all along: the folder you run `terraform` in is the **root module**. Any other module you call from it is a **child module**:

```hcl
module "web_config" {                       # a call to a child module, named "web_config"
  source      = "./modules/app-config"      # where the module's code is
  app_name    = "web"                       # inputs = the module's variables
  environment = "dev"
}

output "web_config_path" {
  value = module.web_config.file_path       # outputs = module.<NAME>.<OUTPUT>
}
```

**Picture it.** A module is a **function** in programming, or a **recipe card**: you pass ingredients in (variables), it does its work (resources), and hands results back (outputs). Calling it twice with different inputs gives two dishes.

**The one thing to know.** A module only sees what you pass in, and you only see what it outputs. Everything else inside is private.

**Try it now:** build the lab below: one `app-config` module called three times with `for_each`.

**You've got it if you can:** explain root vs child module, and write `module.X.Y` to use a module's output.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Root and child modules

| | Root module | Child module |
|---|---|---|
| What | The folder where you run `terraform` | A folder (or package) called with a `module` block |
| Has a state? | **Yes**: one state per root module (per workspace) | No: its resources live **inside the root's state** |
| Has a backend? | Yes | No (backends only in the root) |
| `provider` blocks | Yes, configure providers here | ⚠️ Shouldn't: receives providers from the root (topic 44) |
| Variables come from | tfvars, `-var`, `TF_VAR_` | The `module` block's arguments |
| Outputs go to | The CLI, `terraform output`, remote state | The caller, as `module.NAME.OUTPUT` |

### The `module` block

```hcl
module "<NAME>" {
  source  = "<where>"              # required: local path, registry, Git... (topic 45)
  version = "~> 6.0"               # only for registry modules

  # inputs: one argument per variable declared in the module
  app_name = "web"

  # meta-arguments (like resources)
  count      = 1                   # or for_each
  providers  = { aws = aws.us }    # which provider configurations it uses (topic 44)
  depends_on = [aws_iam_role.x]    # ⚠️ whole-module dependency; use sparingly (topic 20)
}
```

- Every **required** variable of the module (no default) must be set here.
- Passing an argument that isn't a variable in the module → `Unsupported argument`.

### Outputs and the module object

- `module.web_config.file_path`: one output.
- `module.web_config`: an object with **all** outputs.
- With `for_each`: `module.web_config["api"].file_path`; with `count`: `module.web_config[0].file_path`.
- You **can't** reach inside: `module.web_config.local_file.this` doesn't work. If callers need something, the module must output it.

### Addresses in state

Resources inside a module get the module path as a prefix:

```
module.web_config["api"].local_file.this
└───────┬─────────────┘ └──────┬───────┘
   module instance        resource inside it
```

Modules can call modules (nesting): `module.platform.module.network.aws_vpc.this`.

---

## Why it exists

Without modules, every environment and every service copies the same 200 lines, and every fix must be repeated everywhere. Modules give:

- **Reuse**: write the "secure S3 bucket" once, use it 50 times.
- **Encapsulation**: callers set 5 inputs instead of understanding 12 resources.
- **Standards**: the platform team bakes encryption, tags and logging into the module, so everyone gets them.
- **Consistency across environments**: dev and prod call the same module with different inputs.

---

## Analogy

**A recipe card.** "Tomato soup: give me 4 tomatoes and 1 onion; I'll give you 2 bowls of soup." You don't need to know the chopping steps (encapsulation). Calling it for 2 people or 10 people (`for_each`) uses the same card. If the card is improved, everyone who uses it gets the better soup next time they cook (re-init and apply).

---

## How it works inside

```
terraform init
  for each module block (root and, recursively, inside child modules):
    local path ("./...")  → just recorded; read in place (not copied)
    registry/Git/...      → downloaded into .terraform/modules/<KEY>/
  write .terraform/modules/modules.json (which call → which folder/version)
  collect required_providers from ALL modules → install providers (topic 10)

terraform plan
  load the root + every child module into ONE configuration tree
  each module becomes a namespace in the graph:
    module.web_config["api"].var.app_name ──▶ module.web_config["api"].local_file.this
                                               ──▶ module.web_config["api"].output.file_path
  edges cross module boundaries ONLY through inputs and outputs
  → one plan, one state, one apply for everything
```

So a module isn't a separate run: it's part of the same graph, and resources in different modules run in parallel when nothing links them.

Changing a module's `source`, or adding a new module block, needs `terraform init` again (`Module not installed` otherwise).

---

## Code / commands

`~/tf-lab/43-modules/`:

```
43-modules/
├── main.tf
└── modules/
    └── app-config/
        ├── variables.tf
        ├── main.tf
        └── outputs.tf
```

### The child module

`modules/app-config/variables.tf`:

```hcl
variable "app_name" {
  type        = string
  description = "Application name; used in the file name."
}

variable "environment" {
  type        = string
  description = "Environment name."
}

variable "port" {
  type        = number
  description = "Port the app listens on."
  default     = 8080
}
```

`modules/app-config/main.tf`:

```hcl
terraform {
  required_providers {
    local = { source = "hashicorp/local", version = ">= 2.5" }   # modules: broad minimums (topic 10)
  }
}

resource "local_file" "this" {
  filename = "${path.root}/out/${var.environment}-${var.app_name}.conf"
  content  = <<-EOT
    app=${var.app_name}
    env=${var.environment}
    port=${var.port}
  EOT
}
```

- `this` is the common name for "the one main resource of this module".
- `path.root` = the root module's folder (so files land in `43-modules/out/`, not inside `modules/`). `path.module` would be the module's own folder.

`modules/app-config/outputs.tf`:

```hcl
output "file_path" {
  description = "Path of the generated config file."
  value       = local_file.this.filename
}

output "port" {
  value = var.port
}
```

### The root module

`main.tf`:

```hcl
terraform {
  required_version = ">= 1.14"
  required_providers {
    local = { source = "hashicorp/local", version = "~> 2.5" }
  }
}

locals {
  apps = {
    api    = { port = 8080 }
    web    = { port = 3000 }
    worker = { port = 9090 }
  }
}

module "app_config" {
  source   = "./modules/app-config"
  for_each = local.apps

  app_name    = each.key
  environment = "dev"
  port        = each.value.port
}

output "config_files" {
  value = { for name, m in module.app_config : name => m.file_path }
}
```

### init, plan, apply

```
$ terraform init
Initializing the backend...
Initializing modules...
- app_config in modules/app-config
Initializing provider plugins...
...
```

- `- app_config in modules/app-config`: a **local** module: recorded, not copied.

```
$ terraform plan
  # module.app_config["api"].local_file.this will be created
  + resource "local_file" "this" {
      + content  = <<-EOT
            app=api
            env=dev
            port=8080
        EOT
      + filename = "./out/dev-api.conf"
      ...
  # module.app_config["web"].local_file.this will be created
  ...
  # module.app_config["worker"].local_file.this will be created
  ...
Plan: 3 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + config_files = {
      + api    = "./out/dev-api.conf"
      + web    = "./out/dev-web.conf"
      + worker = "./out/dev-worker.conf"
    }
```

The addresses show exactly where each resource lives.

```bash
terraform apply -auto-approve
terraform state list
cat .terraform/modules/modules.json | jq .
```

```
module.app_config["api"].local_file.this
module.app_config["web"].local_file.this
module.app_config["worker"].local_file.this

{
  "Modules": [
    { "Key": "", "Source": "", "Dir": "." },
    { "Key": "app_config", "Source": "./modules/app-config", "Dir": "modules/app-config" }
  ]
}
```

### Errors you'll see

Forgetting a required input:

```
│ Error: Missing required argument
│
│   on main.tf line 17, in module "app_config":
│   17: module "app_config" {
│
│ The argument "environment" is required, but no definition was found.
```

Reaching inside the module:

```
│ Error: Unsupported attribute
│
│ This object does not have an attribute named "local_file".
```

Adding a module without `init`:

```
│ Error: Module not installed
│
│   on main.tf line 30:
│   30: module "extra" {
│
│ This module is not yet installed. Run "terraform init" to install all modules
│ required by this configuration.
```

### Moving existing code into a module

If `local_file.api_config` used to be in the root, moving it into the module needs a `moved` block (topic 40):

```hcl
moved {
  from = local_file.api_config
  to   = module.app_config["api"].local_file.this
}
```

---

## Hands-on lab

1. Build the lab; apply; check `out/` and `terraform state list`.
2. Add a fourth app to `local.apps`: only one new module instance appears in the plan.
3. Call the module once more, without `for_each`, as `module "admin"` with different inputs. Reference its output.
4. Try to read `module.app_config["api"].local_file.this` from the root; read the error; add an output instead.
5. Change the module (add a `log_level` variable with a default) and plan: all instances update.
6. Rename the module block from `app_config` to `configs` with a `moved` block (`from = module.app_config` `to = module.configs`). Plan: only moves.
7. `terraform destroy`.

---

## Where you'll use it

- **Everywhere** in real Terraform: network, compute, database, IAM and observability modules composed in thin root modules per environment (topics 47, 49).
- **Platform teams** publish modules; product teams call them (topics 44–46).
- **Public modules** like `terraform-aws-modules/vpc` replace hundreds of lines (topic 46).

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `Module not installed` | New/changed module source since `init`. | `terraform init`. |
| `Missing required argument` | A module variable without default not passed. | Pass it. |
| `Unsupported argument` in a module block | Passing a name the module doesn't declare as a variable. | Check the module's `variables.tf`; spelling. |
| `Unsupported attribute` on `module.x` | Reaching into resources, or a missing output. | Add an output to the module. |
| Files/paths end up inside `modules/...` | Used `path.module` where you meant the root. | `path.root` (or pass the path in as a variable). |
| Plan destroys and recreates after wrapping code in a module | Addresses changed. | `moved` blocks (topic 40). |
| `Duplicate module call` | Two `module` blocks with the same name. | Rename one. |

---

## Quick recap

- A module = a folder of `.tf` files. **Root** = where you run Terraform (has state and backend); **child** = called with `module "NAME" { source = ... }`.
- Inputs are the child's variables; outputs come back as `module.NAME.OUTPUT`. Internals are private.
- `count`/`for_each` work on modules; addresses become `module.NAME[KEY].TYPE.NAME`.
- `init` installs modules (`.terraform/modules`); all modules form **one graph, one state**.

### Interview questions

1. **Q: What's the difference between a root module and a child module?**
   A: The root module is the working directory Terraform runs in: it has the backend, state and provider configurations. Child modules are called from it with `module` blocks; their resources live in the root's state and they receive inputs and providers from the caller.
2. **Q: How do you get a value out of a module?**
   A: The module must declare an `output`; the caller reads it as `module.<name>.<output>`. Resources inside a module aren't accessible directly.
3. **Q: Do modules have their own state?**
   A: No. All modules called from a root module share the root's single state; their resources are addressed with a `module.<name>.` prefix.
