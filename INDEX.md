# Terraform: From Zero to Production

One learning path, in reading order. Each topic uses only ideas explained before it. When a later idea has to appear early, it is explained right there in a line or two, and a pointer like "(topic 36)" just tells you where the full story is. You never need to jump ahead to follow along.

**Every topic has two layers:**

1. **⚡ Core: the 10-minute version.** The idea in plain words, a mental model, the one command or block to know, a mini-lab, and a self-check.
2. **The full reference**, always in the same shape: What it is · Why it exists · Analogy · How it works inside · Code / commands (sample `plan`/`apply` output, explained line by line) · Hands-on lab · Where you'll use it · When it breaks · Quick recap + interview questions.

**Current Terraform (checked against the official release pages, 19 November 2026):** targets **Terraform 1.17** (latest 1.17.2, 11 November 2026), and notes **OpenTofu 1.13** (latest 1.13.9) differences where they matter. Examples use `required_version = ">= 1.14"`, so anything from the last year works. When a feature needs a newer version, the topic says so. Deprecated or discouraged patterns (provisioners, `terraform taint`, `terraform import` without a block, `null_resource`, DynamoDB state locking, workspaces used as environments, `-target` as a habit…) are marked ⚠️ where they come up and are never taught as the main way.

**Your lab:** Terraform inside the **WSL2 Ubuntu** from your Docker notes. Labs use only free local providers (`local`, `random`, `docker`, `kubernetes` on kind) for topics 07–28. From Part 4 (topic 29) they use the **AWS free tier**. Anything that costs money is marked 💰, and every AWS lab ends with `terraform destroy`.

**Related notes:** [Linux](../linux-mastery-rhel/INDEX.md) · [Docker](../Docker/INDEX.md) · [Kubernetes](../CKA-Certification-Course-2025/INDEX.md) · [Jenkins](../Jenkins/INDEX.md) · [GitHub Actions](../GitHub-Actions-Basics-To-Production/notes/INDEX.md). Topics link straight to the matching topic file wherever Terraform builds on them.

**Sample outputs** show the *shape* of real output. IDs, timings and versions are only examples. Your terminal is the source of truth.

**Levels:** ★ foundation · ★★ intermediate · ★★★ advanced.

**Status:** ✅ Complete: all 71 topics (Parts 0–11) written and reviewed, November 2026.

## How to study with these notes

| Your goal | Read | Time |
|---|---|---|
| **First pass: get the whole picture** | Only the **⚡ Core** of each topic, in order, doing every mini-lab | ~12 hours |
| **Become job-ready** | Second pass: the full reference plus every Hands-on lab and the capstone | A few weeks |
| **Interview tomorrow** | Cores of Parts 0–5, then [69](notes/69-interview-question-bank.md) + [70](notes/70-senior-scenario-interviews.md) | An evening |
| **Fixing something right now** | [64](notes/64-debugging-terraform.md) → [65](notes/65-error-catalogue.md) → [71](notes/71-cheat-sheet.md) | Minutes |

---

## Part 0 · Foundations you bring

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 01 | ★ | [What is Infrastructure as Code?](notes/01-what-is-iac.md) | Life before IaC (ClickOps, snowflake servers, drift, "works in staging"), what IaC fixes, declarative vs imperative, idempotency, mutable vs immutable infrastructure |
| 02 | ★ | [The IaC tool landscape](notes/02-iac-tool-landscape.md) | Provisioning vs configuration management, Terraform vs CloudFormation vs Pulumi vs CDK vs Ansible vs Crossplane, where each fits |
| 03 | ★ | [What is Terraform?](notes/03-what-is-terraform.md) | HashiCorp, HCL (HashiCorp Configuration Language), core vs providers, state at a glance, write → plan → apply, the mental model for everything that follows |
| 04 | ★ | [The license change and OpenTofu](notes/04-license-change-and-opentofu.md) | MPL → BSL (2023), what it means for you, OpenTofu (Linux Foundation fork), IBM's purchase of HashiCorp, compatibility, which one interviewers expect |
| 05 | ★ | [Linux, Git and cloud API recap for Terraform](notes/05-linux-git-and-api-recap.md) | PATH, env vars, files and permissions, Git commits/branches/`.gitignore`, what an API (Application Programming Interface) call and HTTP are, why Terraform is "a program that calls APIs" |
| 06 | ★ | [Lab setup](notes/06-lab-setup.md) | Install Terraform in WSL2 from HashiCorp's apt repo, version managers (tfenv), VS Code extension, checking Docker Engine, shell autocomplete |

## Part 1 · First steps and the core loop

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 07 | ★ | [Your first Terraform project](notes/07-first-terraform-project.md) | A `local_file` in 10 lines: init → plan → apply → change → destroy, and what files appear on disk |
| 08 | ★ | [HCL syntax](notes/08-hcl-syntax.md) | Blocks, labels, arguments, attributes, expressions, comments, how `.tf` files are merged, `terraform fmt` and `validate` |
| 09 | ★ | [The core workflow commands](notes/09-core-workflow-commands.md) | `init`, `validate`, `fmt`, `plan`, `apply`, `destroy`, saved plan files, `-auto-approve`, exit codes, `show`, plan symbols `+ ~ - -/+` |
| 10 | ★ | [Providers](notes/10-providers.md) | What a provider plugin is, the Registry, source addresses, version constraints, `.terraform.lock.hcl`, what `init` downloads and where, provider `alias` |
| 11 | ★ | [Resources and references](notes/11-resources-and-references.md) | Resource blocks, arguments vs attributes, resource addresses, referencing one resource from another, the `random` provider, "known after apply" |
| 12 | ★ | [State: the basics](notes/12-state-basics.md) | What `terraform.tfstate` is, why Terraform needs it, reading the file, `state list`/`show`, why you never edit it by hand |
| 13 | ★★ | [What happens inside `plan` and `apply`](notes/13-inside-plan-and-apply.md) | The full trace: load config → init plugins → refresh state → build the dependency graph → diff → plan → walk the graph in parallel → provider gRPC (Google Remote Procedure Call) → cloud API → write state |

## Part 2 · Making configuration flexible

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 14 | ★ | [Input variables](notes/14-input-variables.md) | Types, defaults, `validation`, `sensitive`, `nullable`, `.tfvars`, `-var`, `TF_VAR_*`, the full precedence order |
| 15 | ★ | [Outputs](notes/15-outputs.md) | Output values, `sensitive` outputs, `terraform output -json`, using outputs in scripts and CI |
| 16 | ★ | [Locals](notes/16-locals.md) | Naming expressions, DRY (Don't Repeat Yourself) tags and names, when locals help and when they hide things |
| 17 | ★ | [Types and expressions](notes/17-types-and-expressions.md) | string/number/bool, list/set/map/object/tuple, conditionals, operators, string templates, heredocs, splat `[*]`, `null` |
| 18 | ★★ | [Built-in functions and `terraform console`](notes/18-functions-and-console.md) | The 30 functions you'll actually use, testing expressions interactively, `try`, `can`, `lookup`, `merge`, `templatefile`, `cidrsubnet` |
| 19 | ★ | [Data sources](notes/19-data-sources.md) | Reading things Terraform doesn't manage, when data sources are read, `local_file` / `http` / `external` examples |
| 20 | ★★ | [Dependencies and the graph](notes/20-dependencies-and-the-graph.md) | Implicit vs explicit (`depends_on`) dependencies, `terraform graph`, parallelism, create/destroy order, dependency cycles |
| 21 | ★★ | [`count`](notes/21-count.md) | Making N copies, `count.index`, conditional resources (`count = var.x ? 1 : 0`) |
| 22 | ★★ | [`for_each`](notes/22-for-each.md) | Maps and sets, `each.key`/`each.value`, why `count` shifts indexes and destroys the wrong thing, choosing between them |
| 23 | ★★ | [`for` expressions and `dynamic` blocks](notes/23-for-expressions-and-dynamic-blocks.md) | Transforming lists and maps, filtering, generating nested blocks, when `dynamic` makes code worse |
| 24 | ★★ | [The `lifecycle` meta-argument](notes/24-lifecycle.md) | `create_before_destroy`, `prevent_destroy`, `ignore_changes`, `replace_triggered_by`, why resources get replaced |
| 25 | ★★ | [Custom conditions and `check` blocks](notes/25-conditions-and-checks.md) | `precondition`, `postcondition`, `check` blocks, failing early with clear messages |

## Part 3 · Real infrastructure on your laptop

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 26 | ★ | [Docker recap and the Docker provider](notes/26-docker-provider.md) | Docker images/containers/networks/volumes recap, how the provider talks to the Docker socket, managing an nginx container with Terraform |
| 27 | ★★ | [Project 1: a full app stack in Docker](notes/27-project-1-docker-stack.md) | Network + volume + database + app + reverse proxy, variables, outputs, `for_each`, health checks; break it and fix it |
| 28 | ★★ | [Terraform and Kubernetes (kind)](notes/28-terraform-and-kubernetes.md) | Kubernetes recap, the `kubernetes` and `helm` providers on kind, where Terraform should stop and GitOps should start |

## Part 4 · AWS

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 29 | ★ | [AWS basics for Terraform](notes/29-aws-basics.md) | Account, root vs IAM (Identity and Access Management) users, regions and AZs (Availability Zones), free tier, billing alarms, AWS CLI, access keys, never using root keys |
| 30 | ★★ | [The AWS provider and authentication](notes/30-aws-provider-and-auth.md) | The credential chain, profiles, SSO, `assume_role`, `default_tags`, multiple regions and accounts with aliases |
| 31 | ★ | [First AWS resources: S3 and EC2](notes/31-first-aws-resources.md) | An S3 bucket and an EC2 instance on the free tier, AMI lookup with a data source, reading real `plan` output for cloud resources |
| 32 | ★★ | [AWS networking with Terraform](notes/32-aws-networking.md) | VPC, subnets, internet gateway, route tables, security groups, `cidrsubnet`, public vs private subnets |
| 33 | ★★ | [Project 2: a web server on AWS](notes/33-project-2-web-server-on-aws.md) | VPC + EC2 + `user_data` + security group + outputs, end to end, then destroy |

## Part 5 · State in depth

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 34 | ★★ | [State deep dive](notes/34-state-deep-dive.md) | The JSON format, `serial` and `lineage`, refresh, `-refresh-only`, `-refresh=false`, secrets in state, state vs reality vs config |
| 35 | ★★ | [Remote backends](notes/35-remote-backends.md) | Why local state fails in teams, the S3 backend, encryption and versioning, partial backend config, migrating state, the bootstrap ("chicken and egg") problem |
| 36 | ★★ | [State locking](notes/36-state-locking.md) | Why two applies at once corrupt things, S3 native locking (`use_lockfile`), ⚠️ DynamoDB locking (deprecated), lock errors, `force-unlock` and when it's safe |
| 37 | ★★ | [Drift](notes/37-drift.md) | What drift is, detecting it (`plan -refresh-only`, scheduled plans), fixing it (code it, revert it, or ignore it) |
| 38 | ★★ | [Sharing state between configurations](notes/38-sharing-state.md) | `terraform_remote_state`, data-source alternatives, SSM Parameter Store, tight vs loose coupling |
| 39 | ★★ | [Importing existing infrastructure](notes/39-import.md) | `import` blocks, `-generate-config-out`, bulk imports, ⚠️ legacy `terraform import` command |
| 40 | ★★ | [Refactoring: `moved` and `removed` blocks](notes/40-moved-and-removed.md) | Renaming without destroying, moving into modules, forgetting a resource safely, vs `state mv` / `state rm` |
| 41 | ★★ | [Replacing resources on purpose](notes/41-replace.md) | `-replace`, ⚠️ `taint` (deprecated), reading "forces replacement", replacement surprises |
| 42 | ★★★ | [State surgery and recovery](notes/42-state-recovery.md) | `state pull/push/mv/rm`, lost or corrupted state, restoring an S3 version, lineage/serial errors, a step-by-step recovery runbook |

## Part 6 · Modules

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 43 | ★★ | [Modules: the basics](notes/43-modules-basics.md) | Root vs child modules, inputs and outputs, `source`, what `init` does with modules, module addresses in state |
| 44 | ★★ | [Writing reusable modules](notes/44-writing-modules.md) | Standard file layout, designing inputs, `versions.tf`, documentation, `count`/`for_each` on modules, provider passing |
| 45 | ★★ | [Module sources and versioning](notes/45-module-sources-and-versioning.md) | Registry, Git with `?ref=` tags, private registries, semantic versioning, pinning strategy |
| 46 | ★★ | [Using public modules](notes/46-public-modules.md) | `terraform-aws-modules/vpc`, judging a module's quality, wrapping vs forking |
| 47 | ★★★ | [Module design patterns](notes/47-module-patterns.md) | Composition, thin vs fat modules, blast radius, anti-patterns (wrapper of a wrapper, modules for one resource) |

## Part 7 · Environments and team structure

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 48 | ★★ | [Workspaces](notes/48-workspaces.md) | CLI workspaces, `terraform.workspace`, how state is stored per workspace, ⚠️ why they're a poor fit for dev/staging/prod |
| 49 | ★★★ | [Environments and repository layout](notes/49-environments-and-layout.md) | Folder per environment, per-component state, monorepo vs multi-repo, Terragrunt (what it solves), keeping environments the same |
| 50 | ★★ | [HCP Terraform and other platforms](notes/50-hcp-terraform-and-platforms.md) | HCP Terraform (ex-Terraform Cloud), remote runs, Stacks, Atlantis, Spacelift/env0, when you need one |

## Part 8 · Security and quality

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 51 | ★★★ | [Secrets and sensitive data](notes/51-secrets.md) | `sensitive`, what still leaks into state, ephemeral resources and values, write-only arguments, Vault and AWS Secrets Manager, protecting state |
| 52 | ★★ | [Provisioners, and what to use instead](notes/52-provisioners-and-alternatives.md) | ⚠️ Provisioners (last resort), `terraform_data` (replaces `null_resource`), `user_data`/cloud-init, Packer images, Ansible after Terraform |
| 53 | ★★ | [Linting and security scanning](notes/53-linting-and-scanning.md) | `fmt`/`validate`, TFLint, Trivy, Checkov, pre-commit hooks, what each catches |
| 54 | ★★★ | [Policy as code](notes/54-policy-as-code.md) | Why rules beat reviews, OPA (Open Policy Agent)/Conftest on plan JSON, Sentinel, cost checks with Infracost |
| 55 | ★★★ | [Testing Terraform](notes/55-testing.md) | `terraform test`, `.tftest.hcl` files, `plan` vs `apply` tests, mocking providers, Terratest at a glance |

## Part 9 · CI/CD (Continuous Integration / Continuous Delivery)

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 56 | ★★ | [The Git workflow for Terraform](notes/56-git-workflow.md) | Branch → PR (Pull Request) → plan → review → merge → apply, saved plans, "plan what you apply", locking in pipelines |
| 57 | ★★★ | [Terraform in GitHub Actions](notes/57-github-actions.md) | OIDC (OpenID Connect) to AWS with no stored keys, plan on PR with a comment, apply with environment approval, concurrency |
| 58 | ★★★ | [Terraform in Jenkins](notes/58-jenkins.md) | Declarative pipeline, credentials binding, `input` approval, stashing the plan, same flow as GitHub Actions |

## Part 10 · Running it in production

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 59 | ★★★ | [Capstone part 1: foundation](notes/59-capstone-1-foundation.md) | Bootstrap the state bucket, repo layout, modules for network and compute, dev environment |
| 60 | ★★★ | [Capstone part 2: environments and CI/CD](notes/60-capstone-2-environments-and-cicd.md) | Add prod, GitHub Actions with OIDC, scanning, tests, approvals |
| 61 | ★★★ | [Capstone part 3: day-2 operations](notes/61-capstone-3-day-2.md) | Drift, an import, a refactor with `moved`, a provider upgrade, a state recovery drill, tear-down |
| 62 | ★★★ | [Upgrades: Terraform, providers and modules](notes/62-upgrades.md) | `required_version`, `init -upgrade`, multi-platform lock files, reading changelogs, major provider upgrades safely |
| 63 | ★★★ | [Terraform at scale](notes/63-terraform-at-scale.md) | Slow plans, `-parallelism`, splitting state, ⚠️ `-target` (emergency only), API rate limits, large organisations |
| 64 | ★★ | [Debugging Terraform](notes/64-debugging-terraform.md) | `TF_LOG`, `TF_LOG_PATH`, reading provider logs, crash logs, a method: symptom → phase → tool |
| 65 | ★★ | [Error catalogue](notes/65-error-catalogue.md) | Every common error: lock, drift, cycles, version conflicts, unexpected replacement, auth, "inconsistent final plan", what each means and the fix |
| 66 | ★★★ | [Production break-fix scenarios](notes/66-production-scenarios.md) | Reproducible incidents to break and fix yourself |

## Part 11 · Interviews and reference

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 67 | ★★ | [What's new and what's next](notes/67-whats-new.md) | Recent Terraform features in one place, Terraform vs OpenTofu feature gaps, what to learn after these notes |
| 68 | ★★ | [Production-readiness checklist](notes/68-production-checklist.md) | One page before you let a Terraform repo touch production |
| 69 | ★★ | [Interview question bank](notes/69-interview-question-bank.md) | Questions with model answers across all parts |
| 70 | ★★★ | [Senior scenario interviews](notes/70-senior-scenario-interviews.md) | Design and incident questions, with how to reason through them |
| 71 | ★ | [Cheat sheet](notes/71-cheat-sheet.md) | Every command and block, grouped by task, with the topic that explains it |
