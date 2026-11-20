# 67 · What's New and What's Next

> **Level:** ★★ intermediate
>
> **You will learn:** every significant Terraform feature from 1.0 to **1.17** in one timeline (so you can recognise "old" vs "modern" code at a glance and answer "what changed recently?" in interviews), the newest features in more detail (**list resources and `terraform query`**, **actions**, **Stacks**, `destroy = false`, import in modules), the **Terraform vs OpenTofu** differences as of November 2026, and what to learn after this course.
>
> **You need:** the whole course (this topic links back to where each feature was taught).
>
> **Part 11 · Interviews and reference** · Previous: [66 · Production break-fix scenarios](66-production-scenarios.md) · Next: [68 · Production-readiness checklist](68-production-checklist.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Terraform changed a lot since 1.0 (2021). Modern code looks different from tutorials written in 2020:

| Old pattern ⚠️ | Modern replacement | Since |
|---|---|---|
| `terraform state mv` for refactors | `moved` blocks | 1.1 |
| `terraform taint` | `apply -replace` | 0.15.2 |
| `null_resource` | `terraform_data` | 1.4 |
| `terraform import` command | `import` blocks (+ `-generate-config-out`) | 1.5 |
| `terraform refresh` | `apply -refresh-only` | 0.15.4 |
| Hand-made test scripts | `terraform test` (+ mocks) | 1.6 / 1.7 |
| `terraform state rm` to stop managing | `removed` blocks | 1.7 |
| `sensitive` and hoping state is safe | Ephemeral values + write-only arguments | 1.10 / 1.11 |
| DynamoDB state locking | S3 `use_lockfile` | 1.10 / 1.11 |
| `local-exec` for API calls | `action` blocks (provider-defined) | 1.14 |

**The one thing to know.** If you can explain **why** each modern feature replaced the old one, you understand Terraform's direction: everything moves **into the plan**, where it can be reviewed.

**Try it now:** read an old Terraform repo (any GitHub project from 2020) and list which modern features would improve it.

**You've got it if you can:** name five features added since 1.5 and what problem each solves.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Timeline (Terraform 1.x)

| Version | Date | Headline features | Topic |
|---|---|---|---|
| **1.0** | Jun 2021 | Stability and compatibility promise for 1.x | 03 |
| 1.1 | Dec 2021 | `moved` blocks; `cloud` block for HCP Terraform | 40, 50 |
| 1.2 | May 2022 | `precondition`/`postcondition`; `replace_triggered_by` | 24, 25 |
| 1.3 | Sep 2022 | `optional()` object attributes with defaults | 17 |
| 1.4 | Mar 2023 | `terraform_data` (replaces `null_resource`) | 24, 52 |
| **1.5** | Jun 2023 | **`import` blocks**, `-generate-config-out`; **`check` blocks**; `plantimestamp()` | 25, 39 |
| **1.6** | Oct 2023 | **`terraform test`**; first **BSL**-licensed release; S3 backend reworked | 04, 55 |
| 1.7 | Jan 2024 | **`removed` blocks**; test **mocks**; `for_each` in `import` | 39, 40, 55 |
| 1.8 | Apr 2024 | **Provider-defined functions**; cross-type `moved` | 18, 40 |
| 1.9 | Jun 2024 | Variable `validation` can reference other variables/objects; `templatestring()`; `moved` from `null_resource` to `terraform_data` | 14, 25, 40 |
| **1.10** | Nov 2024 | **Ephemeral** values, resources, variables, outputs; S3 native locking (`use_lockfile`, experimental) | 36, 51 |
| **1.11** | Feb 2025 | **Write-only arguments**; `use_lockfile` GA; DynamoDB locking deprecated | 36, 51 |
| 1.12 | May 2025 | Import by **identity**; `terraform test -parallelism`; short-circuit `&&`/`\|\|`; OCI backend | 17, 39, 55 |
| 1.13 | Aug 2025 | `terraform stacks` CLI; parallel test teardown | 50 |
| **1.14** | Nov 2025 | **List resources + `terraform query`**; **`action` blocks** | 39, 52 |
| 1.15 | Apr 2026 | `deprecated` on variables/outputs; `convert()`; variables in module `source`/`version`; output type constraints; S3 backend auth via `aws login`; Windows ARM64 | 14, 45 |
| 1.16 | Aug 2026 | `lifecycle { destroy = false }` (⚠️ listed in the 1.16.0-beta1 notes; confirm it in the final 1.16.0 CHANGELOG before quoting it); action `on_failure` modes; `import` blocks in modules; `terraform_data` `store` block for ephemeral/sensitive values; action destroy events; `graph -format=mermaid`; `state show -json` | 24, 39, 52 |
| **1.17** | Oct 2026 | Variables/locals in provider requirements; `-minimal-refresh` plan option; **Terraform Policy** GA in the CLI (`-policies` on `plan`/`apply`/`query`) | 54, 63 |

(Dates are the x.0 release months, checked against the official changelogs in November 2026: 1.12.0 May 14 2025, 1.13.0 Aug 20 2025, 1.14.0 Nov 19 2025, 1.15.0 Apr 29 2026, 1.16.0 Aug 26 2026, 1.17.0 Oct 21 2026. Latest patch at that time: 1.17.2. Earlier: `-replace` and `-refresh-only` came in 0.15.x, just before 1.0.)

### The newest features in more detail

**List resources and `terraform query` (1.14+).** Providers can expose **list resources**: queries like "all EC2 instances with tag X". You write `list` blocks in `*.tfquery.hcl` files and run `terraform query`, which prints the matching real objects and can **generate configuration for importing them** with `-generate-config-out` (`terraform validate -query` checks query files offline). It closes the loop on drift's blind spot (unmanaged objects, topic 37) and on bulk adoption (topic 39). Provider support grows release by release; check your provider's docs for "List Resources".

**Actions (1.14+).** Provider-defined **imperative operations** (invoke a Lambda, invalidate a CDN cache, run an SSM command), declared with `action` blocks and triggered from resource lifecycle events (`before_`/`after_` `create`/`update`, in an `action_trigger` block inside `lifecycle`) or explicitly with `-invoke`. They replace many `local-exec` provisioners with typed, validated, plan-visible operations (topic 52). 1.16 added `before_destroy`/`after_destroy` events.

**Stacks (HCP Terraform).** Components (`*.tfcomponent.hcl`) + deployments (`*.tfdeploy.hcl`) orchestrated together, including deferred changes for chicken-and-egg cases like "cluster then Kubernetes resources" (topics 28, 50).

**`lifecycle { destroy = false }` (1.16+, ⚠️ verify in the final CHANGELOG)** and **import blocks in modules (1.16+, confirmed in the v1.16.0 release post)**: per-resource "forget instead of destroy", and modules that can adopt existing objects themselves (topics 24, 39). OpenTofu has had `destroy = false` since its 1.12.

**Terraform Policy in the CLI (1.17+).** `plan`, `apply` and `query` take `-policies` to check policy files locally or in any CI, without HCP Terraform (topic 54).

### Terraform vs OpenTofu (November 2026)

Both share HCL, providers and the core workflow (topic 04). Notable differences:

| Area | Terraform 1.17 | OpenTofu 1.13 |
|---|---|---|
| Licence / governance | BSL 1.1, HashiCorp (IBM) | MPL 2.0, Linux Foundation |
| State encryption | Backend-side only | **Client-side state encryption** (since 1.7) |
| Provider `for_each` | No | **Yes** (1.9+) |
| `-exclude` planning | No (only `-target`) | **Yes** (1.9+) |
| Dynamic `prevent_destroy` | No (literal only) | **Yes** (1.12) |
| OCI registries for providers/modules | No | **Yes** (1.10+) |
| `.tofu` file overrides | — | **Yes** |
| HCP Terraform, Stacks, Sentinel | **Yes** | No (other platforms support OpenTofu) |
| Terraform Policy (`-policies`) | **Yes** (1.17+) | No |
| List resources / `terraform query`, `action` blocks | **Yes** (1.14+) | No (not in 1.12–1.13) |
| `lifecycle { destroy = false }` | Yes (1.16+) | Yes (1.12+) |
| Ephemeral values, write-only args, import blocks, tests, mocks | Yes | Yes (with some syntax differences over time) |

Moving between them is usually easy **if** you avoid tool-specific features; test with a copy of state first.

### What to learn next

| Area | Why | Where you have a head start |
|---|---|---|
| **Packer** | Immutable images for VMs (topic 52) | HCL syntax |
| **Ansible** | Configuration inside servers | Topic 02, 52 |
| **Kubernetes + GitOps** (Argo CD/Flux) | Where apps go after Terraform builds clusters | Your CKA notes, topic 28 |
| **AWS depth** (IAM, networking, EKS, RDS) | Most Terraform jobs are AWS-heavy | Topics 29–33, capstone |
| **Policy (OPA/Rego)** | Used in Terraform, Kubernetes, APIs | Topic 54 |
| **Observability** (Prometheus, Grafana, CloudWatch) | Operating what you build | CKA notes topics 79–80 |
| **Certifications** | HashiCorp Certified: Terraform Associate (check the current exam version and objectives); AWS Solutions Architect / DevOps Engineer | This course covers the Associate objectives |

---

## Why it exists

Interviewers often ask "what's new in Terraform?" to see whether you keep up. More practically, recognising old patterns lets you modernise codebases safely, and knowing the roadmap tells you which hacks will soon have proper replacements.

---

## Analogy

Car models over the years: the controls you learned still work, but newer models add safety features (moved/import/removed blocks, ephemeral values) and driver aids (tests, query, actions). A good driver knows which features their car has, and doesn't rely on workarounds the new model made obsolete.

---

## How it works inside

The common thread: Terraform keeps moving operations **from imperative commands into declarative, plan-visible configuration**:

```
state mv → moved     import cmd → import block     state rm → removed
taint → -replace     refresh → -refresh-only       local-exec → actions
"hope state is safe" → ephemeral + write-only       scripts to find resources → terraform query
```

Everything in the plan can be reviewed, tested, policy-checked and applied by a pipeline.

---

## Code / commands

Check your versions and look for modern features in a repo:

```bash
terraform version
grep -rlE 'null_resource|terraform taint|dynamodb_table|provisioner "remote-exec"' --include=*.tf . | sort
grep -rlE '^\s*(moved|removed|import|check)\s*\{' --include=*.tf . | sort
```

The first list is your modernisation backlog; the second shows what's already modern.

---

## Hands-on lab

1. Pick one of your old labs (or a public 2020-era repo) and write a modernisation plan: which deprecated patterns, which replacement, which topic.
2. If your AWS provider version supports list resources, try a `terraform query` for your EC2 instances (follow the provider's docs for the exact `list` block).
3. Install OpenTofu (topic 06) and run your capstone `dev/network` plan with `tofu` against a **copy** of the state (`tofu init` with a different backend key). Note any differences.

---

## Where you'll use it

- **Interviews**: "What's new in Terraform?", "How do you import/refactor/handle secrets now?"
- **Modernising codebases**: replacing `null_resource`, taints, import commands, DynamoDB locking.
- **Tool choices**: Terraform vs OpenTofu decisions with facts.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `Unsupported block type "import"/"removed"/"ephemeral"/"action"` | Terraform version too old for the feature | Check the timeline; upgrade (topic 62) |
| Feature works in OpenTofu but not Terraform (or the reverse) | Tool-specific feature | Stick to the shared subset or standardise on one tool |
| Tutorial code fails (`Unsupported argument "versioning"` in `aws_s3_bucket`) | Outdated tutorial (provider v3 era) | Translate to modern resources (topic 31) |

---

## Quick recap

- 1.1 `moved` · 1.2 conditions · 1.3 `optional()` · 1.4 `terraform_data` · 1.5 `import`/`check` · 1.6 `test` (+ BSL) · 1.7 `removed`/mocks · 1.8 provider functions · 1.10 ephemeral · 1.11 write-only/`use_lockfile` · 1.14 query/actions · 1.16 `destroy = false`, import in modules · 1.17 `-minimal-refresh`, Terraform Policy GA in the CLI.
- Direction: imperative commands → declarative, plan-visible blocks.
- OpenTofu differs in licence, state encryption, provider `for_each`, `-exclude`, OCI registries; Terraform has HCP, Stacks, query and actions.

### Interview questions

1. **Q: What are the most important Terraform features of the last few years?**
   A: Import blocks with config generation (1.5), `terraform test` with mocks (1.6–1.7), `removed` blocks (1.7), ephemeral values and write-only arguments for secrets (1.10–1.11), native S3 locking (1.10–1.11), and list resources with `terraform query` and actions (1.14).
2. **Q: How has handling secrets changed in recent Terraform?**
   A: Ephemeral resources and variables (1.10) let Terraform use values that are never stored in state or plans, and write-only arguments (1.11) let providers receive secrets without persisting them, so secrets no longer have to live in state.
3. **Q: Terraform or OpenTofu: what are the practical differences today?**
   A: Same language, providers and workflow. OpenTofu is MPL-licensed under the Linux Foundation and adds client-side state encryption, provider `for_each`, `-exclude` and OCI registries; Terraform has HCP Terraform, Stacks, Sentinel/Terraform Policy, `terraform query` and actions.
