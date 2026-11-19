# 03 · What Is Terraform?

> **Level:** ★ foundation
>
> **You will learn:** what Terraform is, its two halves (**Terraform core** and **providers**), what **HCL** looks like, what the **state file** is for (in one paragraph; topic 12 goes deeper), the **write → init → plan → apply** workflow, and how Terraform differs from Kubernetes' always-on reconciliation. This is the mental model for the whole course.
>
> **You need:** topics 01–02.
>
> **Part 0 · Foundations** · Previous: [02 · The IaC tool landscape](02-iac-tool-landscape.md) · Next: [04 · The license change and OpenTofu](04-license-change-and-opentofu.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Terraform is **one program** (`terraform`) that reads `.tf` files describing what you want, compares them with what it created last time (the **state file**) and with reality, shows you a **plan**, and after you approve, calls cloud APIs to make reality match.

It has two halves:

- **Terraform core**: the `terraform` binary. Reads your files, works out the plan, decides the order.
- **Providers**: plugins that know one platform's API (AWS, Docker, Kubernetes, GitHub...). Core never talks to AWS itself; the AWS provider does.

**Picture it.** Core is a **project manager** who reads the blueprint and makes the schedule. Providers are **specialist contractors** (plumber, electrician) who do the real work on their own system.

**The one thing to know.** The loop you'll type thousands of times:

```bash
terraform init      # download the providers this code needs
terraform plan      # preview: what will be created, changed, destroyed
terraform apply     # do it (asks "yes" first)
terraform destroy   # delete everything this code created
```

**Try it now:** read this, and predict what Terraform will do on the first `apply`, and on a second `apply` with no changes:

```hcl
resource "local_file" "hello" {
  filename = "hello.txt"
  content  = "Hello, Terraform!"
}
```

(First: create `hello.txt`. Second: nothing, "No changes", because it's idempotent. You'll run it for real in topic 07.)

**You've got it if you can:** name the two halves of Terraform and the four commands, and say what the state file is for in one sentence.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

**Terraform** is an open-source-*style* (see topic 04 for the licence) **infrastructure as code tool** made by **HashiCorp** (owned by IBM since 2025). It was first released in **2014**, reached **1.0** in **2021**, and the current line is **1.17** (November 2026). Since 1.0, HashiCorp has promised that code written for any 1.x version keeps working on later 1.x versions.

It's a **single binary** written in Go. There's no server, no daemon, no agent on your machines. You download one file and run it, on your laptop or in a CI runner.

### The pieces

| Piece | What it is | Where it lives |
|---|---|---|
| **Configuration** | Your `.tf` files, written in HCL. Describes the desired state. | Your Git repo |
| **Terraform core** | The `terraform` CLI. Parses config, manages state, builds the plan, orders operations. | Installed on your machine (topic 06) |
| **Providers** | Plugins, one per platform. Translate "create `aws_s3_bucket`" into real AWS API calls. | Downloaded by `terraform init` into `.terraform/` |
| **State** | A JSON file recording what Terraform created and the real IDs. | `terraform.tfstate` locally, or a remote backend (topic 35) |
| **Modules** | Reusable folders of `.tf` files (Part 6). | Local folders, Git, the Registry |
| **Registry** | Public catalogue of providers and modules: `registry.terraform.io`. | Internet |

### HCL in ten seconds

**HCL = HashiCorp Configuration Language.** It's made of **blocks** with **arguments**:

```hcl
resource "docker_container" "web" {    # block type, then two labels
  name  = "web"                        # argument = value
  image = "nginx:1.29"
}
```

It's meant to be read by humans first. Topic 08 covers the syntax fully.

### The state file in one paragraph

Terraform needs to remember **"the thing I call `docker_container.web` in the code is the real container with ID `a1b2c3...`"**. It writes that mapping, plus every attribute it knows about each object, into the **state file**. Without state, Terraform couldn't tell "update my container" from "create another one", and it couldn't know what to delete when you remove a block. State is so important it gets nine topics (12 and 34–42).

---

## Why it exists

From topic 02: teams needed **one** tool and **one** workflow for **every** platform, with a **preview** before changes. Terraform's design delivers that:

- **Providers** make it platform-agnostic. HashiCorp, cloud vendors and the community write providers; core stays the same.
- **The plan** makes changes reviewable. You see `+ create`, `~ update`, `- destroy` before anything happens.
- **The dependency graph** works out the order (create the network before the VM) and does independent things in parallel.
- **State** makes it possible to update and delete, not only create.

---

## Analogy

**Renovating a house with a project manager.**

- **The blueprint** (your `.tf` files): what the house should look like.
- **The project manager** (Terraform core): reads the blueprint, checks the **logbook**, walks around the house, and writes a **work order**: "add a window in the kitchen, repaint bedroom 2, remove the shed".
- **The logbook** (state): "the shed in the blueprint is the one at the back, built in March, ID 42".
- **You** (the human) read the work order and sign it (plan → approve).
- **Specialist contractors** (providers): the glazier for the window, the painter for the bedroom. The manager doesn't do the work; they direct.
- **The project manager isn't on site 24/7.** They only come when you call them (run `terraform`). If a neighbour paints your door red in between, nobody notices until the manager's next visit. That's the key difference from Kubernetes.

---

## How it works inside

### The architecture

```
   you
    │  terraform plan / apply
    ▼
┌──────────────────────────────────┐        ┌───────────────────────────┐
│ TERRAFORM CORE (terraform binary)│        │ state file                │
│  • parse .tf files (HCL)         │◀──────▶│ terraform.tfstate         │
│  • load state                    │        │ (local or remote backend) │
│  • build the dependency graph    │        └───────────────────────────┘
│  • compute the plan (diff)       │
│  • walk the graph in parallel    │
└───────┬──────────────┬───────────┘
        │ gRPC         │ gRPC          gRPC = Google Remote Procedure Call:
        ▼              ▼               a fast way for two programs to call
┌──────────────┐ ┌──────────────┐      each other's functions
│ AWS provider │ │Docker provider│     (each provider runs as its own process)
└──────┬───────┘ └──────┬───────┘
       │ HTTPS          │ Unix socket /var/run/docker.sock
       ▼                ▼
   AWS APIs         Docker daemon
```

- Each provider is a **separate program**. Core starts it as a child process and talks to it over **gRPC**. That's why a buggy provider can't corrupt core, and why providers can be written and released independently.
- Core knows nothing about AWS. It only knows "resource of type `aws_s3_bucket`, ask the provider that owns the `aws_` prefix what to do".

### The workflow

```
 write .tf ──▶ terraform init ──▶ terraform plan ──▶ review ──▶ terraform apply
                  │                   │                            │
       download providers     read state + ask provider    call APIs in graph order,
       and modules, set up    for reality, diff with       save the new state
       the backend            config, print the plan
```

Topic 13 traces each step in detail.

### Terraform vs Kubernetes: run-once vs always-on

You know Kubernetes from your CKA notes. Both are declarative, but:

| | Kubernetes | Terraform |
|---|---|---|
| When does it compare desired vs actual? | **All the time** (controllers in a loop) | **Only when you run** `plan`/`apply` |
| Where is desired state stored? | etcd (in the cluster) | Your `.tf` files in Git |
| Where is "what I created" stored? | etcd | The state file |
| Fixes drift automatically? | Yes, continuously | No. It shows drift on the next plan; you decide |
| Who runs it? | Controllers in the cluster | You, or a CI/CD pipeline |

So Terraform is "declarative, **on demand**". Production teams often schedule a `terraform plan` every few hours to spot drift (topic 37).

---

## Code / commands

You'll install Terraform in topic 06. Here is a whole small project, so you know where the course is heading. Read every comment.

```hcl
# versions.tf: which Terraform and which providers this code needs
terraform {
  required_version = ">= 1.14"           # refuse to run on older Terraform

  required_providers {
    local = {
      source  = "hashicorp/local"        # Registry address: <namespace>/<name>
      version = "~> 2.5"                 # any 2.x from 2.5 up (topic 10)
    }
  }
}

# main.tf: what we want to exist
resource "local_file" "hello" {          # resource TYPE "local_file", NAME "hello"
  filename = "${path.module}/hello.txt"  # path.module = this folder
  content  = "Hello, Terraform!\n"
}

# outputs.tf: values to print after apply
output "file_path" {
  value = local_file.hello.filename      # a reference: TYPE.NAME.ATTRIBUTE
}
```

Running `terraform apply` on it prints (sample shape):

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # local_file.hello will be created
  + resource "local_file" "hello" {
      + content              = <<-EOT
            Hello, Terraform!
        EOT
      + content_base64sha256 = (known after apply)
      + content_base64sha512 = (known after apply)
      + content_md5          = (known after apply)
      + content_sha1         = (known after apply)
      + content_sha256       = (known after apply)
      + content_sha512       = (known after apply)
      + directory_permission = "0777"
      + file_permission      = "0777"
      + filename             = "./hello.txt"
      + id                   = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + file_path = "./hello.txt"

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

local_file.hello: Creating...
local_file.hello: Creation complete after 0s [id=4b3c0f3e8f1b2a9d7c6e5f4a3b2c1d0e9f8a7b6c]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

file_path = "./hello.txt"
```

| Line | Meaning |
|---|---|
| `+ create` | The legend: `+` means "will be created". |
| `# local_file.hello will be created` | The resource's **address** (TYPE.NAME) and the action. |
| `content_md5 = (known after apply)` etc. | Checksums of the content. The provider computes them, so they're only known after creation. |
| `+ directory_permission = "0777"` | A value you didn't set: the provider's **default**. (Your Linux `umask` still trims it, as in [Linux topic 026](../../linux-mastery-rhel/notes/026-file-permissions-and-umask.md).) |
| `id = (known after apply)` | Terraform can't know the ID until the provider creates the object. |
| `Plan: 1 to add, 0 to change, 0 to destroy.` | The summary. Read it every time. |
| `Enter a value: yes` | Only the exact word `yes` approves. Anything else cancels. |
| `Creation complete after 0s [id=...]` | The provider created it and returned the real ID, which goes into state. |
| `Outputs:` | Values from `output` blocks. |

---

## Hands-on lab

No install yet. Prepare your mental model:

1. Draw (on paper) the architecture diagram above from memory: you → core → state, core → providers → APIs.
2. Look at the AWS provider page on the Registry (`registry.terraform.io/providers/hashicorp/aws/latest`). Find: the current version, the "Documentation" tab, and the doc page for `aws_s3_bucket`. Notice each resource page has **Argument Reference** (what you can set) and **Attribute Reference** (what you can read back). You'll use these pages every day.
3. Answer: if you run `terraform apply` twice on the project above, then delete `hello.txt` by hand and run `terraform plan`, what will the plan say? (Answer: `1 to add`, because Terraform checks reality during plan, sees the file is gone, and plans to create it again.)

---

## Where you'll use it

- Creating cloud accounts' baseline: networks, IAM roles, logging.
- Building Kubernetes clusters (EKS/AKS/GKE) and their add-ons.
- Managing SaaS settings as code: GitHub repos and teams, Cloudflare DNS, Datadog monitors, PagerDuty schedules.
- In CI/CD: plan on every PR, apply on merge (Part 9).

---

## When it breaks

Early surprises, explained:

| Symptom | Meaning | Fix |
|---|---|---|
| `Error: Inconsistent dependency lock file` / `Required plugins are not installed` | You ran `plan` before `init`, or changed providers without re-running `init`. | Run `terraform init`. (Topic 10.) |
| Plan wants to **create** things that already exist | Terraform has no state for them (state lost, wrong folder, wrong backend). | Don't apply. Find the right state (topic 35) or import them (topic 39). |
| Someone's console change disappeared after your apply | Terraform put reality back to match the code. | Make changes in code, not the console (topic 37). |
| `Error: No configuration files` | You're in a folder without `.tf` files. | `cd` to the right folder. Terraform only reads the **current folder**. |

---

## Quick recap

- Terraform = **core** (one Go binary: parse, plan, graph, state) + **providers** (plugins, one per platform, talked to over gRPC).
- You write **HCL** in `.tf` files. Terraform records what it created in the **state** file.
- Workflow: **init → plan → (review) → apply**, and **destroy** to clean up.
- Terraform reconciles **only when run**, unlike Kubernetes' always-on controllers.

### Interview questions

1. **Q: What are Terraform core and providers?**
   A: Core is the CLI that parses configuration, manages state, builds the dependency graph and computes plans. Providers are separate plugin processes that core calls over gRPC; each translates resource operations into a specific platform's API calls.
2. **Q: Why does Terraform need a state file?**
   A: To map resources in code to real objects (by ID), store their last known attributes, track dependencies for correct delete order, and speed up planning. Without it Terraform couldn't update or destroy what it created.
3. **Q: Does Terraform fix drift automatically like Kubernetes?**
   A: No. It only compares desired and actual state when you run plan or apply. Teams schedule plans or use drift detection features to find drift.
