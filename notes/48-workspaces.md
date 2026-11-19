# 48 · Workspaces

> **Level:** ★★ intermediate
>
> **You will learn:** what Terraform **CLI workspaces** are (several states for one configuration and one backend), the commands (`workspace new/list/select/show/delete`), `terraform.workspace` in code, where workspace state is stored (local `terraform.tfstate.d/`, S3 `env:/<name>/<key>`), what they're **good** for (short-lived copies: feature branches, tests), and ⚠️ why they're a **poor fit for dev/staging/prod**. Also: HCP Terraform "workspaces" are a different thing.
>
> **You need:** topics 12, 14, 35 (backends, keys), 36.
>
> **Part 7 · Environments and team structure** · Previous: [47 · Module design patterns](47-module-patterns.md) · Next: [49 · Environments and repository layout](49-environments-and-layout.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **workspace** is a named, separate **state** for the **same** configuration and backend. Every configuration starts in the workspace `default`.

```bash
terraform workspace new feature-login     # create + switch
terraform workspace list                  #   default
                                          # * feature-login
terraform apply                           # creates a SEPARATE copy of everything
terraform workspace select default        # back to the original state
```

In code, `terraform.workspace` is the current name:

```hcl
locals {
  name = "shop-${terraform.workspace}"     # shop-default, shop-feature-login
}
```

**Picture it.** One recipe, several cooking pots. Same instructions, separate pots of soup. But all pots sit on the **same stove** with the **same gas supply** (same backend, same credentials).

**The one thing to know.** Workspaces are great for **temporary copies** of the same thing (a test environment per pull request). For **dev/staging/prod**, use separate folders/state keys/accounts (topic 49): prod needs different credentials, settings and review, and a forgotten `workspace select` shouldn't be able to hit it.

**Try it now:** in any local lab, create two workspaces, apply in each, and look in `terraform.tfstate.d/`.

**You've got it if you can:** explain where workspace state lives and two reasons workspaces are risky for prod.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Commands

| Command | Does |
|---|---|
| `terraform workspace list` | List workspaces; `*` marks the current one |
| `terraform workspace show` | Print the current one |
| `terraform workspace new NAME` | Create and switch |
| `terraform workspace select NAME` | Switch (`-or-create` to create if missing, 1.4+) |
| `terraform workspace delete NAME` | Delete (only if its state is empty, unless `-force`) |
| `TF_WORKSPACE=NAME terraform plan` | Select by environment variable (used in CI) |

The current workspace is remembered in `.terraform/environment` (a tiny file in the working directory).

### Where the states live

| Backend | `default` workspace | Other workspaces |
|---|---|---|
| local | `terraform.tfstate` | `terraform.tfstate.d/<name>/terraform.tfstate` |
| s3 | `<key>` | `env:/<name>/<key>` (prefix changeable with `workspace_key_prefix`) |
| azurerm / gcs | the configured blob/object | similar, name-suffixed |

Same bucket, same credentials, same lock mechanism, one state per workspace.

### Good uses

| Use | Why workspaces fit |
|---|---|
| **Preview environments per PR/branch** | `workspace new pr-123`, apply, test, destroy, delete. Identical infrastructure, short-lived. |
| **Testing a module or config change** in a throwaway copy | Same code, isolated state |
| **Many identical tenants** with tiny differences | Possible, but usually `for_each` or separate roots are clearer |

### ⚠️ Why not dev/staging/prod

| Problem | Explanation |
|---|---|
| **Same backend, same credentials** | Whoever can apply dev can apply prod; you can't put prod state in the prod account. |
| **Invisible in code** | Looking at the files, you can't tell which environment you're about to change; it's hidden in `.terraform/environment`. |
| **Easy mistakes** | Forget `workspace select dev` → you just planned (or applied) against prod. |
| **Environments diverge anyway** | Prod has more AZs, bigger instances, deletion protection, extra monitoring; you end up with `terraform.workspace == "prod" ? ... : ...` everywhere. |
| **Same provider version and module versions forced on all envs** | You can't roll a module upgrade through dev first in a separate PR. |

HashiCorp's own documentation says CLI workspaces aren't suitable for isolating environments that need separate credentials and access controls.

### HCP Terraform workspaces are different

In HCP Terraform (topic 50), a **workspace** is a full unit: its own state, variables, credentials, permissions, run history and VCS settings. Using one HCP workspace per environment is normal and fine. Don't confuse the two.

---

## Why it exists

Workspaces were designed for "I want another copy of this, temporarily, without copying files": testing a change in parallel with the main deployment. They're convenient, which is why people stretched them to dev/staging/prod, and why the problems above are now well-known interview material.

---

## Analogy

**One recipe, several pots on one stove.**

- The recipe (configuration) is the same; each pot (workspace) holds its own soup (state).
- All pots share the **same stove and gas** (backend, credentials). If the "prod" pot needs a locked kitchen with a different key, a shared stove can't provide that.
- The label saying which pot you're stirring is on a sticky note on your sleeve (`.terraform/environment`), not on the pot. Easy to stir the wrong one.

---

## How it works inside

```
terraform plan   (in workspace "feature-login")
  backend s3: key = "lab/app/terraform.tfstate", workspace_key_prefix = "env:"
  → state object: env:/feature-login/lab/app/terraform.tfstate
  → lock object:  env:/feature-login/lab/app/terraform.tfstate.tflock
  → terraform.workspace = "feature-login" for expressions
```

Everything else (config, providers, credentials) is identical between workspaces.

---

## Code / commands

`~/tf-lab/48-ws/main.tf` (local + random providers):

```hcl
locals {
  ws = terraform.workspace
  settings = {
    default = { replicas = 1 }
  }
  # unknown workspaces (e.g. feature branches) fall back to default settings
  replicas = try(local.settings[local.ws].replicas, local.settings.default.replicas)
}

resource "random_pet" "name" {
  prefix = local.ws
}

resource "local_file" "info" {
  filename = "${path.module}/out/${local.ws}.txt"
  content  = "workspace=${local.ws} name=${random_pet.name.id} replicas=${local.replicas}\n"
}
```

```bash
terraform init
terraform apply -auto-approve
terraform workspace new feature-login
terraform apply -auto-approve
terraform workspace list
ls out/ ; find terraform.tfstate.d -type f
```

```
  default
* feature-login

default.txt  feature-login.txt
terraform.tfstate.d/feature-login/terraform.tfstate
```

- `default` state is `./terraform.tfstate`; the new workspace's state is in `terraform.tfstate.d/`.
- Two separate `random_pet` names, two files: two independent copies.

```bash
terraform workspace select default
terraform plan             # No changes: this workspace's state already has its own copy
terraform workspace delete feature-login
```

```
╷
│ Error: Workspace is not empty
│
│ Workspace "feature-login" is currently tracking the following resource instances:
│   - local_file.info
│   - random_pet.name
│
│ Deleting this workspace would cause Terraform to lose track of any associated
│ remote objects, which would then require you to delete them manually outside
│ of Terraform. You should destroy these objects with Terraform before deleting
│ the workspace.
```

Destroy first, then delete:

```bash
terraform workspace select feature-login && terraform destroy -auto-approve
terraform workspace select default && terraform workspace delete feature-login
```

### A per-PR preview environment in CI (sketch)

```bash
WS="pr-${PR_NUMBER}"
terraform init -input=false
terraform workspace select -or-create "$WS"
terraform apply -auto-approve -input=false -var="name_suffix=$WS"
# ... run tests against the preview ...
# on PR close:
terraform workspace select "$WS" && terraform destroy -auto-approve && \
  terraform workspace select default && terraform workspace delete "$WS"
```

---

## Hands-on lab

1. Run the lab; create two extra workspaces; apply in each; inspect `terraform.tfstate.d/`.
2. Put the lab's state in your S3 backend (topic 35), create a workspace, and find `env:/<name>/...` in the bucket.
3. Run `TF_WORKSPACE=feature-login terraform plan` and compare with `workspace show`.
4. Simulate the classic mistake: be in workspace `default` while thinking you're in `feature-login`, and run `plan`. Notice nothing in the output warns you except the resource names. (Some teams print `terraform workspace show` in their shell prompt for this reason.)
5. Clean up every workspace: destroy, then delete.

---

## Where you'll use it

- **Ephemeral environments** per feature branch or PR.
- **Load-testing copies** of a stack, created and destroyed by a pipeline.
- **Reading legacy code** that uses `terraform.workspace` for environments, and migrating it to separate roots (topic 49).
- **HCP Terraform** workspaces, which are a different, full-featured concept (topic 50).

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| Plan wants to create everything | You're in a new/empty workspace (or the wrong one). | `terraform workspace show`; select the right one. |
| You changed prod while "testing" | Wrong workspace selected. | Don't use workspaces for prod; separate roots and credentials. |
| `Workspace is not empty` on delete | It still tracks resources. | Destroy in that workspace first (or `-force`, which orphans resources). |
| `Workspace "x" doesn't exist` | Typo, or never created in this backend. | `workspace list`; `select -or-create`. |
| Code full of `terraform.workspace == "prod" ? ... : ...` | Workspaces stretched into environments. | Per-environment roots + tfvars (topic 49). |

---

## Quick recap

- A CLI **workspace** = another state for the same config and backend; `terraform.workspace` gives its name.
- Local: `terraform.tfstate.d/<ws>/`; S3: `env:/<ws>/<key>`.
- Great for **short-lived copies** (PR previews, tests). ⚠️ Poor for dev/staging/prod: shared credentials and backend, invisible selection, divergence.
- HCP Terraform workspaces are a different, full-featured concept.

### Interview questions

1. **Q: What are Terraform workspaces?**
   A: Named states for one configuration in one backend, selected with `terraform workspace select`, with `terraform.workspace` available in expressions. Each workspace tracks its own copy of the resources.
2. **Q: Should you use CLI workspaces for dev, staging and prod?**
   A: Generally no. They share the backend and credentials, so you can't isolate prod access, the selected environment isn't visible in code, it's easy to apply to the wrong one, and environments need different settings anyway. Use separate root configurations or state keys per environment, ideally in separate accounts.
3. **Q: When are workspaces a good fit?**
   A: Short-lived, identical copies of the same infrastructure, such as preview environments per pull request or isolated test deployments, created and destroyed by automation.
