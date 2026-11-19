# 50 · HCP Terraform and Other Platforms

> **Level:** ★★ intermediate
>
> **You will learn:** what a **Terraform automation platform** adds on top of the CLI and a CI pipeline, how **HCP Terraform** (formerly Terraform Cloud) works: organizations, projects, **workspaces**, **runs**, variable sets, VCS-driven vs CLI-driven workflows, the `cloud` block and `terraform login`, private registry, policies, drift detection, agents; what **Terraform Stacks** are; **Terraform Enterprise** (self-hosted); and the alternatives: **Atlantis**, Spacelift, env0, Scalr and plain CI. You'll run a CLI-driven remote run on HCP Terraform's free tier.
>
> **You need:** topics 35–37 (backends, locking, drift), 48 (CLI workspaces, to avoid confusing the two), 49.
>
> **Part 7 · Environments and team structure** · Previous: [49 · Environments and repository layout](49-environments-and-layout.md) · Next: [51 · Secrets and sensitive data](51-secrets.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Running Terraform for a team needs more than the CLI: shared state, locking, a place to run plans and applies with the right credentials, approvals, policy checks, audit logs. You can **build** that from a CI system (Part 9), or **use a platform** that does it:

| Platform | Type |
|---|---|
| **HCP Terraform** | HashiCorp's SaaS (Software as a Service) |
| **Terraform Enterprise** | HashiCorp's self-hosted version |
| **Atlantis** | Open-source, self-hosted; runs plan/apply from PR comments |
| **Spacelift, env0, Scalr, Terrateam** | Commercial platforms (support Terraform and OpenTofu) |

Connecting a configuration to HCP Terraform:

```hcl
terraform {
  cloud {
    organization = "acme"
    workspaces {
      name = "network-prod"
    }
  }
}
```

```bash
terraform login        # once: stores an API token in ~/.terraform.d/credentials.tfrc.json
terraform init
terraform plan         # runs REMOTELY; output streams to your terminal
```

**Picture it.** Cooking at home (CLI) vs a professional **shared kitchen** that provides the ovens, the fridge with labelled shelves, the head chef's sign-off and a logbook of every dish.

**The one thing to know.** An **HCP Terraform workspace** = state + variables + credentials + permissions + run history for one configuration/environment. It's **not** the same as a CLI workspace (topic 48).

**Try it now:** create a free HCP Terraform account and run the lab below (uses only the `random` provider, so nothing is created in any cloud).

**You've got it if you can:** list five things a platform gives you that plain CLI + S3 backend don't.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### What a platform adds

| Need | CLI + S3 backend | CI pipeline (Part 9) | Platform (HCP Terraform, Spacelift...) |
|---|---|---|---|
| Shared state + locking | ✅ | ✅ | ✅ built in |
| Plans on PRs, applies on merge | ❌ | ✅ you build it | ✅ built in |
| Run queue per state | ❌ (locks only) | Partly (concurrency groups) | ✅ |
| Approvals | ❌ | ✅ (environments) | ✅ |
| Policy as code | ❌ | ✅ you integrate it (topic 54) | ✅ built in |
| Drift detection, health checks | ❌ | ✅ you schedule it (topic 37) | ✅ built in |
| RBAC per workspace/project, audit log | IAM on the bucket | CI permissions | ✅ fine-grained |
| Private module/provider registry | ❌ | Git tags | ✅ |
| Cost estimation | ❌ | Infracost (topic 54) | Some platforms |

### HCP Terraform concepts

| Concept | What it is |
|---|---|
| **Organization** | Your company's account: users, teams, billing, settings. |
| **Project** | A folder of workspaces with shared permissions. |
| **Workspace** | One root configuration's state, variables (Terraform and environment variables, including secrets), credentials, settings and **run history**. One per environment × component is typical. |
| **Run** | One plan (+ optional apply), queued per workspace. Statuses: pending → planning → planned → (policy check) → confirmed → applying → applied. |
| **Variable sets** | Variables shared by many workspaces (e.g. AWS role ARNs per account). |
| **Execution mode** | **Remote** (runs on HashiCorp's workers), **Agent** (runs on your own agents inside your network), **Local** (HCP Terraform only stores state; you run the CLI). |
| **Workflows** | **VCS-driven** (connected to a Git repo: PRs get speculative plans, merges trigger runs), **CLI-driven** (`terraform plan/apply` from a terminal or CI, executed remotely), **API-driven** (tools call the API). |
| **Private registry** | Publish internal modules/providers with versions (topic 45). |
| **Policies** | Sentinel or OPA policies run between plan and apply (topic 54). Separately, Terraform 1.17 made HashiCorp's newer **Terraform Policy** generally available in the CLI itself (`plan`/`apply -policies=...`), so it doesn't need HCP Terraform. |
| **Run tasks** | Call external tools (security scanners, cost tools) during runs. |
| **Health** | Scheduled **drift detection** and **continuous validation** of `check` blocks (topics 25, 37). |
| **Dynamic provider credentials** | Workspaces get short-lived cloud credentials via OIDC, with no stored keys (the same idea as topic 57). |

Managing HCP Terraform itself as code: the **`tfe` provider** (`hashicorp/tfe`) creates organizations' workspaces, teams, variable sets and policies. Reading another workspace's outputs: the **`tfe_outputs`** data source (a safer alternative to `terraform_remote_state`, topic 38).

### Terraform Stacks

**Stacks** (an HCP Terraform feature, with the `terraform stacks` CLI since 1.13) manage **many deployments of the same set of components** as one unit:

- **Components** (`*.tfcomponent.hcl`): which modules make up the stack (network, cluster, apps) and how they connect.
- **Deployments** (`*.tfdeploy.hcl`): the copies to run (dev, prod-eu, prod-us) with their inputs and credentials.
- HCP Terraform orchestrates them, including **deferred changes** (e.g. creating a cluster and then the Kubernetes resources that need it, which is topic 28's chicken-and-egg problem, handled across two plans automatically).

They're HashiCorp's answer to what Terragrunt solves (topic 49). They require HCP Terraform; check the current feature status in HashiCorp's docs, since Stacks evolve quickly.

### Pricing and licensing (check current terms)

- HCP Terraform has a **free tier** limited by the number of managed resources and features; paid tiers are billed per managed resource. Terms change; check HashiCorp's pricing page.
- Terraform Enterprise is a separately licensed self-hosted product.
- The BSL licence (topic 04) is why some competing platforms now emphasise OpenTofu support.

### Atlantis

**Atlantis** is an open-source server you run yourself (often in Kubernetes). It listens to Git webhooks:

```
PR opened ──▶ Atlantis runs `terraform plan` for changed projects ──▶ posts the plan as a PR comment
reviewer comments "atlantis apply" ──▶ Atlantis applies (locks the project to that PR) ──▶ posts the result
merge after apply ("apply before merge" workflow)
```

Pros: free, simple, self-hosted, works with any backend. Cons: you operate and secure it (it holds powerful cloud credentials); fewer governance features.

---

## Why it exists

As teams grow, the hard part of Terraform isn't writing HCL; it's **operating** it: who can apply what, where credentials live, how runs are queued, how policies and approvals are enforced, how drift is noticed, and how everything is audited. Platforms package those answers. Building them yourself in CI is perfectly valid (and common); platforms trade money for less glue code.

---

## Analogy

**Home kitchen vs a professional shared kitchen.**

- **CLI + S3** = your home kitchen with a shared fridge: works, but every cook brings their own tools and there's no head chef.
- **CI pipeline** = you set up stations, rules and a sign-off sheet yourself.
- **Platform** = a professional kitchen you rent: ovens provided (runners), labelled shelves (workspaces/state), the head chef signs every dish (approvals), a health inspector checks automatically (policies), and a logbook records everything (audit).
- **Atlantis** = a sous-chef you hire and train yourself, who cooks when someone writes "cook it" on the order (PR comment).

---

## How it works inside

A CLI-driven remote run on HCP Terraform:

```
terraform plan  (on your laptop, with the cloud block)
  1. uses the token from `terraform login`
  2. uploads the configuration directory (as a tarball) to the workspace
  3. HCP Terraform queues a run → a worker: terraform init + plan
       with the workspace's variables and credentials (NOT your laptop's)
  4. plan logs stream back to your terminal; the state never leaves HCP Terraform
terraform apply
  same, then waits for confirmation (in the terminal or the UI), then applies on the worker
```

So with remote execution, providers run **on the platform's workers**: a `local_file` would be written on their machine, and a `docker` provider would look for Docker there. Use **agents** or **local execution mode** for things that must run inside your network.

---

## Code / commands

### Lab: a CLI-driven remote run (free)

1. Sign up at `app.terraform.io` (HCP Terraform), create an **organization** (e.g. `tf-course-<yourname>`).
2. Log in from WSL:

```bash
terraform login
```

```
Terraform will request an API token for app.terraform.io using your browser.
...
Token for app.terraform.io:
  Enter a value: ****************************

Retrieved token for user you
...
   Welcome to HCP Terraform!
```

The token is stored in `~/.terraform.d/credentials.tfrc.json` (treat it like a password).

3. `~/tf-lab/50-hcp/main.tf`:

```hcl
terraform {
  required_version = ">= 1.14"
  cloud {
    organization = "tf-course-yourname"
    workspaces {
      name = "tf-course-lab"
    }
  }
  required_providers {
    random = { source = "hashicorp/random", version = "~> 3.7" }
  }
}

variable "team" {
  type    = string
  default = "platform"
}

resource "random_pet" "name" {
  prefix = var.team
}

output "name" {
  value = random_pet.name.id
}
```

4. Run it:

```bash
terraform init
terraform apply
```

```
Initializing HCP Terraform...
...
Running apply in HCP Terraform. Output will stream here. Pressing Ctrl-C
will cancel the remote apply if it's still pending. If the apply started it
will stop streaming the logs, but will not stop the apply running remotely.

Preparing the remote apply...

To view this run in a browser, visit:
https://app.terraform.io/app/tf-course-yourname/tf-course-lab/runs/run-AbC123...

Waiting for the plan to start...

Terraform v1.17.2
on linux_amd64
Initializing plugins and modules...

Terraform will perform the following actions:

  # random_pet.name will be created
  + resource "random_pet" "name" { ... }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions in workspace "tf-course-lab"?
  ...
  Enter a value: yes

random_pet.name: Creating...
random_pet.name: Creation complete after 0s [id=platform-smart-gecko]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

name = "platform-smart-gecko"
```

- The **plan ran on HCP Terraform's worker** (`on linux_amd64` there), not on your laptop.
- Open the run URL: you'll see the plan, apply, the state version and who triggered it.

5. In the UI, set a **workspace variable** `team = "data"` (category: Terraform variable), then run `terraform plan` locally: the remote variable is used, so the pet is replaced with a `data-` prefix.
6. Clean up: `terraform destroy`, then delete the workspace in the UI (Settings → Destruction and Deletion).

### An Atlantis repo config (for reading)

```yaml
# atlantis.yaml
version: 3
projects:
  - name: dev-app
    dir: envs/dev/app
    autoplan:
      when_modified: ["*.tf", "../../../modules/web-service/**/*.tf"]
    apply_requirements: [approved, mergeable]
  - name: prod-app
    dir: envs/prod/app
    apply_requirements: [approved, mergeable]
```

---

## Hands-on lab

1. Do the HCP Terraform lab above (free).
2. In the workspace UI, find: **Runs**, **States** (versions you can download), **Variables**, **Settings → Locking** (lock the workspace manually and try `terraform plan`).
3. Switch the workspace's **execution mode** to **Local**, and run `terraform apply` again: now the run happens on your laptop and only state is stored remotely.
4. Read the Atlantis config above and map it to topic 49's layout.
5. Write a comparison for your team: CI-only vs HCP Terraform vs Atlantis (cost, effort, security, features).

---

## Where you'll use it

- Companies on **HCP Terraform/Enterprise**: VCS-driven workspaces per environment, Sentinel/OPA policies, private registry.
- Companies using **Atlantis**: PR-comment workflows on self-hosted infrastructure.
- Companies using **GitHub Actions/Jenkins** directly (Part 9): the same concepts, built by hand.
- **Migrations** between them: state moves via `terraform init -migrate-state` (S3 → `cloud` block) or the platform's import tools.

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `Required token could not be found` | Not logged in. | `terraform login` (CI: `TF_TOKEN_app_terraform_io` env var). |
| `Error: Unsupported block type ... cloud` | Old Terraform, or `cloud` and `backend` both present. | Terraform ≥ 1.1; use one or the other. |
| Remote run can't find credentials | Workspace has no cloud credentials (your laptop's don't travel). | Workspace variables, variable sets, or dynamic provider credentials (OIDC). |
| `local_file`/Docker/`external` resources behave strangely | They ran on the remote worker. | Local execution mode or agents. |
| Run stuck in "pending" | Another run is queued/locked on the workspace, or agent pool offline. | Check the run queue and agents. |
| Atlantis apply blocked | Apply requirements (approval, mergeable) not met, or another PR holds the project lock. | Get approval; `atlantis unlock` the other PR if abandoned. |

---

## Quick recap

- Platforms add run queues, remote execution, credentials, approvals, policies, drift detection, RBAC, audit and private registries on top of state.
- **HCP Terraform**: organizations → projects → **workspaces** (state + vars + creds + runs); VCS-, CLI- or API-driven; remote/agent/local execution; `cloud` block + `terraform login`.
- **Stacks** orchestrate many deployments of the same components in HCP Terraform.
- **Atlantis** (open source, self-hosted) runs plan/apply from PR comments. Spacelift, env0, Scalr are commercial alternatives. Plain CI works too.

### Interview questions

1. **Q: What does HCP Terraform give you that an S3 backend doesn't?**
   A: Remote, queued runs with managed credentials, VCS integration with plans on PRs, approvals, policy checks, drift detection, fine-grained permissions, audit history, a private registry, and state management with locking built in.
2. **Q: What's the difference between an HCP Terraform workspace and a CLI workspace?**
   A: A CLI workspace is just another state for the same configuration and backend. An HCP Terraform workspace is a full unit with its own state, variables, credentials, permissions, settings and run history, so one per environment is normal there.
3. **Q: How does Atlantis work?**
   A: It's a self-hosted server receiving Git webhooks: it runs `terraform plan` on pull requests and posts the result as a comment; an approved reviewer comments `atlantis apply` to apply, with per-project locks so two PRs can't change the same state at once.
