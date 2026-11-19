# 45 · Module Sources and Versioning

> **Level:** ★★ intermediate
>
> **You will learn:** every way to point `source` at a module (local paths, the **Terraform Registry**, **Git** with `?ref=` and `//subdir`, GitHub shortcuts, S3/HTTP archives, private registries), how module **versions** work (`version` for registries, `?ref=` tags for Git), **semantic versioning** for modules, a release process (tags + changelog), pinning to a **commit SHA** for supply-chain safety, how `init -upgrade` updates modules, why modules are **not** in the lock file, and Terraform 1.15's variables in `source`/`version`.
>
> **You need:** topics 05 (Git), 10 (version constraints, lock file), 43–44.
>
> **Part 6 · Modules** · Previous: [44 · Writing reusable modules](44-writing-modules.md) · Next: [46 · Using public modules](46-public-modules.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** `source` says **where** the module's code comes from; a **version** says **which release**.

```hcl
module "vpc" {                                  # public registry: source + version
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 6.0"
}

module "bucket" {                               # Git: version via ?ref=<tag>
  source = "git::https://github.com/acme/tf-modules.git//s3-secure-bucket?ref=v1.4.0"
}

module "local" {                                # local path: no version (same repo)
  source = "./modules/app-config"
}
```

**Picture it.** Ordering a book: from your own shelf (local path), from a bookshop catalogue by edition (registry + version), or from a specific photocopy of a manuscript (Git tag/commit).

**The one thing to know.** **Always pin** remote modules: a `version` constraint for registry modules, a **tag or commit** in `?ref=` for Git. An unpinned module can change under you on the next `init`.

**Try it now:** call a module from a Git repo with `?ref=v1.0.0`, then tag `v1.1.0`, change the ref, and run `terraform init -upgrade`.

**You've got it if you can:** write a Git source with a subdirectory and a tag, and explain why modules aren't in `.terraform.lock.hcl`.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Source types

| Source | Example | Version via |
|---|---|---|
| **Local path** | `./modules/network`, `../shared/modules/s3` (must start with `./` or `../`) | — (same commit as the caller) |
| **Terraform Registry** | `terraform-aws-modules/vpc/aws` (`<NAMESPACE>/<NAME>/<PROVIDER>`) | `version = "~> 6.0"` |
| **Private registry** | `app.terraform.io/acme/network/aws` | `version` |
| **Git (any host)** | `git::https://github.com/acme/tf-modules.git//network?ref=v2.1.0` | `?ref=` tag, branch or commit |
| **Git over SSH** | `git::ssh://git@github.com/acme/tf-modules.git//network?ref=v2.1.0` | `?ref=` |
| **GitHub shorthand** | `github.com/acme/tf-modules//network?ref=v2.1.0` | `?ref=` |
| **S3 bucket** | `s3::https://s3-ap-south-1.amazonaws.com/acme-modules/network-v2.1.0.zip` | Name of the archive |
| **HTTP(S) archive** | `https://example.com/network-v2.1.0.zip` | Name of the archive |

Syntax details for Git:

- `//subdir` (double slash) = the module lives in a sub-folder of the repo. Lets one repo hold many modules.
- `?ref=` = a tag (`v2.1.0`), a branch (`main`, ⚠️ moves), or a full commit SHA (most immutable).
- `?depth=1` = shallow clone (faster for big repos).
- Authentication uses your normal Git credentials (SSH keys, credential helpers, `GIT_*` env vars). In CI, a deploy key or token.

### Versioning modules

Modules follow **semantic versioning** like providers (topic 10):

| Change | Version bump | Examples |
|---|---|---|
| Breaking | **MAJOR** (`v2.0.0`) | Removed/renamed variable or output; changed a default in a way that replaces resources; raised minimum provider major; changed resource addresses without `moved` blocks |
| New feature, compatible | **MINOR** (`v1.5.0`) | New optional variable, new output, new optional resource |
| Fix | **PATCH** (`v1.4.1`) | Bug fix with no interface change |

A release process:

1. Merge changes to the module repo's main branch through PRs (with tests, topic 55).
2. Update `CHANGELOG.md`.
3. Tag: `git tag -a v1.5.0 -m "Add lifecycle rules" && git push origin v1.5.0`.
4. Registries (public, or HCP Terraform's private registry) pick up new tags automatically.
5. Callers bump `version`/`ref` in a PR, run `init -upgrade`, review the plan.

### Version constraints for registry modules

Same operators as providers: `~> 6.0` (any 6.x), `~> 6.2.0` (6.2.x), `= 6.2.1`. Root configurations usually pin to a minor range or exact version; read the module's changelog before moving.

### Modules are not in the lock file

`.terraform.lock.hcl` records **providers only**. For modules:

- Registry modules: `init` picks the **newest** version matching the constraint **at that time**, so two `init`s on different days can get different versions unless you pin exactly (`= 6.2.1`).
- Git modules: the `ref` is the pin. A **tag** can be moved by someone with push access (rare, but possible); a **commit SHA** can't.
- That's why security-sensitive organisations pin exact versions or SHAs, and use bots (Renovate/Dependabot, which understand Terraform module sources) to propose upgrades.

### Variables in `source` and `version` (Terraform 1.15+)

Since 1.15, `source` and `version` may use **variables and locals** whose values are known during `init` (for example, a shared `local.modules_ref`). Older versions require literal strings. Use it to bump many module calls at once; keep it simple, because readers expect to see versions directly.

---

## Why it exists

Local modules are enough inside one repo. As soon as several repos (teams, environments, services) share modules, you need **distribution** (a place to get them) and **versioning** (so a change in the module doesn't instantly change everyone's infrastructure). Pinning turns "someone edited the module" into "a new version exists; upgrade when you're ready, after reading the plan".

---

## Analogy

**Books.**

- **Local path** = a book on your own shelf: you always read the copy you have.
- **Registry + version** = ordering from a bookshop's catalogue by **edition**: "6th edition or any later printing of it" (`~> 6.0`).
- **Git tag** = a dated photocopy of the author's manuscript ("version of 1 March"). A **commit SHA** = the same photocopy with a tamper-proof seal.
- **Branch** = "whatever the author has on their desk today". Fine to read, dangerous to build a house from.

---

## How it works inside

```
terraform init
  for each module call:
    local path   → nothing to download
    registry     → GET https://registry.terraform.io/v1/modules/terraform-aws-modules/vpc/aws/versions
                   pick the highest version matching the constraint
                   GET the download URL (usually a GitHub archive at that tag) → unpack
    git::...     → git clone (--depth if set) → checkout ref → copy subdir
  store in .terraform/modules/<call key>/   and record in modules.json
  (an existing download is reused unless the source/version changed or -upgrade)

terraform init -upgrade
  re-resolve every module (and provider) to the newest allowed version
```

---

## Code / commands

### A Git-hosted module repo

Create a module repo locally (a "remote" on your disk works exactly like GitHub for this lab):

```bash
mkdir -p ~/tf-lab/45-repo/tf-modules && cd ~/tf-lab/45-repo/tf-modules
git init -b main
mkdir app-config
cp ~/tf-lab/43-modules/modules/app-config/*.tf app-config/
git add . && git commit -m "app-config module" && git tag v1.0.0
```

Use it from another folder:

```hcl
# ~/tf-lab/45-consumer/main.tf
terraform {
  required_providers {
    local = { source = "hashicorp/local", version = "~> 2.5" }
  }
}

module "api" {
  source      = "git::file:///home/YOU/tf-lab/45-repo/tf-modules//app-config?ref=v1.0.0"
  app_name    = "api"
  environment = "dev"
}
```

(Replace `YOU` with your Linux user; `echo $HOME`. On GitHub it would be `git::https://github.com/acme/tf-modules.git//app-config?ref=v1.0.0`.)

```
$ terraform init
Initializing modules...
Downloading git::file:///home/you/tf-lab/45-repo/tf-modules?ref=v1.0.0 for api...
- api in .terraform/modules/api/app-config
```

- Downloaded the repo at the tag, used the `app-config` sub-folder.

### Release v1.1.0 and upgrade

```bash
cd ~/tf-lab/45-repo/tf-modules
# add a new optional variable (a MINOR change)
cat >> app-config/variables.tf <<'EOF'

variable "log_level" {
  type    = string
  default = "info"
}
EOF
sed -i 's/port=${var.port}/port=${var.port}\n    log=${var.log_level}/' app-config/main.tf
git commit -am "Add log_level (optional)" && git tag v1.1.0
```

In the consumer, change `?ref=v1.0.0` to `?ref=v1.1.0`:

```bash
cd ~/tf-lab/45-consumer
terraform plan
```

```
│ Error: Module source has changed
│
│ The source address was changed since this module was installed. Run
│ "terraform init" to install all modules required by this configuration.
```

```bash
terraform init      # (or init -upgrade for registry modules within a constraint)
terraform plan      # shows the content change from the new default log_level
```

### Pin to a commit SHA

```bash
cd ~/tf-lab/45-repo/tf-modules && git rev-parse v1.1.0
```

```
3f9c2a1d7b6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a
```

```hcl
source = "git::https://github.com/acme/tf-modules.git//app-config?ref=3f9c2a1d7b6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a"  # v1.1.0
```

Keep the tag name in a comment; tools like Renovate can update both.

### A registry module

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 6.0"
  # inputs... (topic 46)
}
```

```
$ terraform init
Initializing modules...
Downloading registry.terraform.io/terraform-aws-modules/vpc/aws 6.5.1 for vpc...
- vpc in .terraform/modules/vpc
```

The exact version chosen (`6.5.1` here, illustrative) is printed. Pin `= 6.5.1` if you need byte-for-byte repeatability.

---

## Hands-on lab

1. Build the local Git module repo, tag `v1.0.0`, consume it with `?ref=v1.0.0`.
2. Release `v1.1.0` (optional variable), upgrade the consumer, read the plan.
3. Make a **breaking** change in the module (rename `app_name` to `name`), tag `v2.0.0`, upgrade the consumer, read the error, update the call.
4. Pin to a commit SHA instead of a tag.
5. Point `?ref=main` and see that every new commit changes what you get (⚠️ why branches are bad pins).
6. Look at `.terraform/modules/modules.json` after each step.
7. `terraform destroy`.

---

## Where you'll use it

- **Shared module repos** (one repo, many modules in sub-folders, tagged releases) or **one repo per module** (cleaner versioning, the registry's convention: `terraform-<PROVIDER>-<NAME>`).
- **Private registries** in HCP Terraform/Enterprise or GitLab for internal modules.
- **Dependency bots** opening PRs when module versions are released.
- **Supply-chain reviews**: "are third-party modules pinned to immutable versions?"

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `Module source has changed` / `Module version requirements have changed` | Source/version changed since `init`. | `terraform init` (or `-upgrade`). |
| `Failed to download module ... git ... Permission denied (publickey)` / `Authentication failed` | No Git access (CI especially). | Deploy key/token; correct URL scheme (`git::ssh://` vs `https://`). |
| `no available releases match the given constraints` | No registry version satisfies the constraint. | Check versions on the registry; fix the constraint. |
| `Invalid module source address` | Local path without `./`, or wrong registry format. | `./modules/x`; `NAMESPACE/NAME/PROVIDER`. |
| `Module not found ... subdir` | Wrong `//subdir`. | Check the repo layout at that ref. |
| Infrastructure changed after an `init` nobody noticed | Unpinned or loosely pinned module picked a new version. | Pin exact versions/SHAs; review module upgrades in PRs. |
| Breaking change in a MINOR release | Module author didn't follow SemVer. | Pin exactly; report it; read changelogs. |

---

## Quick recap

- `source`: local (`./`), registry (`ns/name/provider` + `version`), Git (`git::URL//subdir?ref=TAG|SHA`), archives, private registries.
- Version modules with **SemVer** tags and a changelog; breaking interface changes = new major.
- **Pin** remote modules (version constraint or tag/SHA); branches aren't pins.
- Modules aren't in the lock file; `init -upgrade` re-resolves them. Exact pins or SHAs give repeatability.

### Interview questions

1. **Q: How do you version and consume a module from a Git repository?**
   A: Tag releases with semantic versions in the module repo, and consume it with `source = "git::https://...repo.git//path?ref=vX.Y.Z"` (or a commit SHA for immutability). Upgrade by changing the ref in a PR and re-running `init`.
2. **Q: Are module versions recorded in `.terraform.lock.hcl`?**
   A: No, only providers. Registry modules resolve to the newest version matching the constraint at `init` time, so pin exactly if you need repeatability; Git modules are pinned by their `ref`.
3. **Q: What counts as a breaking change in a module?**
   A: Removing or renaming inputs or outputs, changing defaults so resources are replaced, changing resource addresses without `moved` blocks, or requiring a new major provider version. Those need a major version bump.
