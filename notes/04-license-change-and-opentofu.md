# 04 · The License Change and OpenTofu

> **Level:** ★ foundation
>
> **You will learn:** what a software licence is in practice, what changed in **August 2023** when HashiCorp moved Terraform from **MPL** to **BSL**, who that actually affects, what **OpenTofu** is, how compatible the two are today, what IBM's purchase of HashiCorp means, and what to say in an interview.
>
> **You need:** topic 03 (core vs providers, the Registry).
>
> **Part 0 · Foundations** · Previous: [03 · What is Terraform?](03-what-is-terraform.md) · Next: [05 · Linux, Git and cloud API recap](05-linux-git-and-api-recap.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Until version **1.5**, Terraform was **open source** under the **MPL 2.0** (Mozilla Public License). In **August 2023**, HashiCorp switched new versions (1.6 onward) to the **BSL 1.1** (Business Source License). The BSL says: use it freely, including in production at your company, **but don't build a product that competes with HashiCorp** using it.

Companies that sold Terraform-running platforms were hit. They and the community **forked** the last open-source code (copied it into a new project) and created **OpenTofu**, now run by the **Linux Foundation** under the MPL 2.0. Its CLI command is `tofu` instead of `terraform`.

**Picture it.** A popular free recipe book becomes "free to cook from at home, but restaurants that compete with the author can't use it". Some chefs copy the last free edition and keep improving it together.

**The one thing to know.** For you, as an engineer at a normal company, **both are free to use**, the language (HCL) is the same, and almost everything in these notes works in both. These notes use `terraform`. Where OpenTofu differs, there's a note.

**Try it now:** say aloud, in 30 seconds: "Terraform moved to the BSL in 2023, which restricts competing commercial products, not normal use. OpenTofu is the open-source fork under the Linux Foundation. They share HCL and providers, and have started to differ on newer features."

**You've got it if you can:** explain who the BSL restricts and who it doesn't.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Licences in one minute

A **software licence** is the legal permission to use, change and share code.

- **Open source** licences (MPL, Apache, MIT, GPL) let anyone use, change and redistribute the code, **for any purpose**, including selling services built on it.
- **Source-available** licences (BSL is one) let you read the code, but restrict some uses.

### What changed

| | Before (≤ 1.5.x) | After (≥ 1.6) |
|---|---|---|
| Licence | **MPL 2.0**, open source | **BSL 1.1** (Business Source License), source-available |
| Can your company use it in production? | Yes | **Yes** |
| Can you use it in your CI/CD, for clients, as a consultant? | Yes | **Yes** |
| Can you build and sell a product that offers hosted Terraform and competes with HashiCorp? | Yes | **No** (without a commercial agreement) |
| Does the code become open source again? | n/a | Each version converts to MPL **four years** after its release |

What did **not** change:

- **HashiCorp's providers** (like `hashicorp/aws`) stayed under **MPL 2.0**. Most third-party providers have their own open licences.
- **HCL** and the provider **protocol** didn't change, which is why one provider works with both tools.

### Timeline

| When | What |
|---|---|
| 2014 | Terraform released (MPL 2.0). |
| Aug 2023 | HashiCorp announces the BSL for future releases. |
| Aug–Sep 2023 | The **OpenTF manifesto**: companies and engineers ask HashiCorp to reverse it; it doesn't. They fork. The fork is renamed **OpenTofu** and joins the **Linux Foundation**. |
| Jan 2024 | **OpenTofu 1.6.0**, first stable release. |
| Apr 2024 → Feb 2025 | IBM announces and then completes its purchase of HashiCorp. Terraform stays BSL. |
| 2024–2026 | Both projects release new versions independently. In November 2026: **Terraform 1.17**, **OpenTofu 1.13**. |

### OpenTofu

- A **fork** of Terraform 1.5 (a fork = a copy of a project's code that then develops separately).
- Licence: **MPL 2.0**. Run by the **Linux Foundation** with a public steering committee.
- Binary: `tofu`. Commands are the same: `tofu init`, `tofu plan`, `tofu apply`.
- Registry: `registry.opentofu.org`, which serves the **same providers** (it points at the providers' public GitHub releases).
- State file format: compatible for migration in either direction for versions that line up, but **always back up state and test first**.

---

## Why it exists

Why did HashiCorp change the licence? Their stated reason: other companies were selling Terraform-based platforms built on HashiCorp's free work without contributing back, and HashiCorp needed to protect its business: **HCP Terraform** (HCP = HashiCorp Cloud Platform; a hosted service that runs Terraform for teams and stores their state, formerly called Terraform Cloud) and **Terraform Enterprise** (the self-hosted version of the same thing). Topic 50 covers both.

Why did OpenTofu appear? Companies building on Terraform (Spacelift, env0, Harness, Gruntwork, Scalr and others) suddenly had a legal problem, and many in the community wanted infrastructure tooling to stay under neutral, open governance, like Kubernetes is under the CNCF.

---

## Analogy

**A shared road built by one company.**

- For years a company built and maintained a road anyone could drive on, including delivery firms that charged for using it.
- The company put up a sign: "Private cars and company cars: free. Delivery firms that compete with our own delivery service: you need a deal with us."
- Several delivery firms built a **parallel road** from the last public plans, owned by a neutral foundation.
- Normal drivers (you) can use either road. Both lead to the same places (the same providers).

---

## How it works inside

How compatible are they? Picture the layers:

```
                 Terraform 1.17 (BSL)            OpenTofu 1.13 (MPL)
                 ───────────────────             ───────────────────
 Language        HCL ──────────── same core language ──────────── HCL
                 + newer Terraform-only            + newer OpenTofu-only
                   features (below)                  features (below)
 Provider        plugin protocol v5/v6 ──────── same protocol ───── v5/v6
 protocol        (the rules core and provider plugins use to talk; topic 10)
 Providers       hashicorp/aws, docker, ... ───── same providers ─── (via OpenTofu registry)
 State format    JSON v4 ─────────────── same format, but check ──── JSON v4
                                          feature use before switching
 CLI             terraform                                           tofu
```

Features that exist in only one of them (as of November 2026, examples only):

| Terraform-only (examples) | OpenTofu-only (examples) |
|---|---|
| HCP Terraform integration (the hosted service, see "Why it exists" below) and **Stacks** (deploying many copies of a configuration together, topic 50) | **Client-side state encryption** (1.7+) |
| `actions` blocks and `terraform query` / list resources (1.14+) | `for_each` on provider blocks (1.9+) |
| `-minimal-refresh` plan option (1.17) | `-exclude` planning flag (1.9+) |
| Terraform Policy, built into the CLI with `-policies` (1.17) | OCI (Open Container Initiative) registries for providers/modules |
| | `prevent_destroy` that can refer to variables (1.12) |

Some features arrived in both, at different versions: for example `lifecycle { destroy = false }` is in OpenTofu 1.12 and Terraform 1.16, and both have a `convert()` function (OpenTofu 1.13, Terraform 1.15).

> The gap grows every release. If you use a feature from one column, moving to the other tool needs work.

---

## Code / commands

The same project runs with either binary:

```bash
terraform init && terraform plan          # HashiCorp Terraform
tofu init && tofu plan                    # OpenTofu
```

```
$ terraform version
Terraform v1.17.2
on linux_amd64

$ tofu version
OpenTofu v1.13.9
on linux_amd64
```

- `v1.17.2`: major 1, minor 17, patch 2.
- `on linux_amd64`: operating system and CPU (Central Processing Unit) architecture. Providers are downloaded for this platform (topic 10).

Files that OpenTofu reads **instead of** the Terraform ones, if both exist:

| Terraform | OpenTofu also reads | Why |
|---|---|---|
| `main.tf` | `main.tofu` | Lets a shared repo hold OpenTofu-only code next to Terraform code |
| `.terraformrc` | `.tofurc` | CLI config file |

---

## Hands-on lab

1. Read the official statements yourself (10 minutes): HashiCorp's "HashiCorp adopts Business Source License" blog post (August 2023) and the OpenTofu site's "Manifesto" and FAQ. Write down, in your own words, one sentence each side would say.
2. Check the licence file on GitHub: open `github.com/hashicorp/terraform` → `LICENSE` (BSL text, with the "Additional Use Grant"), then `github.com/opentofu/opentofu` → `LICENSE` (MPL 2.0).
3. After topic 06, optionally install OpenTofu too, and run the topic 07 project with `tofu`. Note any output differences.

---

## Where you'll use it

- **Interviews**: "Do you know about the Terraform license change?" is a common question in 2024–2026 interviews. Answer with facts, not opinions.
- **Choosing a tool**: platform teams decide based on support contracts (HashiCorp/IBM), HCP Terraform features, OpenTofu-only features like state encryption, and company policy on licences.
- **Reading job ads**: "Terraform/OpenTofu" is now common. The skills transfer almost completely.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `tofu init` fails: `provider registry.terraform.io/... could not be found` | A provider that isn't mirrored in the OpenTofu registry, or a source address pinned to `registry.terraform.io` explicitly. | Check the OpenTofu registry, or use the provider's explicit address. |
| Code works in OpenTofu but fails in Terraform (or the reverse) with `Unsupported block type` / `Unsupported argument` | You used a feature that only one tool has. | Stick to the shared subset, or standardise on one tool. |
| Your company's legal team asks "are we allowed to use Terraform?" | The BSL allows normal internal use. | Point them to the "Additional Use Grant" in the licence; restricted use is offering a competing hosted product. |
| `Error: Unsupported Terraform Core version` after switching | `required_version` checks the **tool's** version, and the two tools' versions differ. | Write the constraint for the tool you use. |

---

## Quick recap

- **Aug 2023**: Terraform 1.6+ moved from **MPL 2.0** (open source) to **BSL 1.1** (source-available).
- The BSL restricts **competing commercial products**, not normal use at your company.
- **OpenTofu** = open-source fork under the Linux Foundation, CLI `tofu`, same HCL and providers.
- HashiCorp is part of **IBM** since 2025. Terraform 1.17 and OpenTofu 1.13 are current (November 2026); newer features are starting to differ.

### Interview questions

1. **Q: What happened with Terraform's license?**
   A: In August 2023 HashiCorp moved Terraform from MPL 2.0 to the Business Source License from version 1.6. Normal use, including in production, is still free; building a competing commercial product isn't allowed without an agreement.
2. **Q: What is OpenTofu?**
   A: A fork of Terraform 1.5 maintained by the Linux Foundation under MPL 2.0. It uses the same language, providers and workflow, with the `tofu` CLI, and has added its own features like state encryption.
3. **Q: Would you choose Terraform or OpenTofu?**
   A: It depends: Terraform for HCP Terraform and Stacks or an existing HashiCorp support contract; OpenTofu for an open-source licence and features like built-in state encryption. The skills and code mostly transfer, so I'd follow the company's standard.
