# 10 · Providers

> **Level:** ★ foundation
>
> **You will learn:** what a provider really is (a separate plugin program with a **schema**), **source addresses** (`hashicorp/aws`), the `required_providers` and `required_version` settings, **version constraints** (`~>`, `>=`...), what `terraform init` downloads and how it **verifies** it, the **dependency lock file** `.terraform.lock.hcl` line by line, `provider` configuration blocks, **aliases** for several regions/accounts, and the errors you'll hit when versions disagree.
>
> **You need:** topics 03 (core vs providers, gRPC), 07–09.
>
> **Part 1 · First steps and the core loop** · Previous: [09 · The core workflow commands](09-core-workflow-commands.md) · Next: [11 · Resources and references](11-resources-and-references.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **provider** is a plugin that teaches Terraform one platform's API. You declare which providers you need and which versions are allowed. `terraform init` downloads them and records the exact versions in `.terraform.lock.hcl`, which you **commit**.

```hcl
terraform {
  required_version = ">= 1.14"              # which Terraform CLI versions may run this code

  required_providers {
    aws = {                                  # local name you use in this code
      source  = "hashicorp/aws"              # where to get it: <namespace>/<type>
      version = "~> 6.0"                     # any 6.x, never 7.0
    }
  }
}

provider "aws" {                             # configure it
  region = "ap-south-1"
}
```

**Picture it.** Providers are **power adapters**. Core is your laptop; each platform has a different socket. You list which adapters you need and which models are acceptable, and the lock file is the receipt saying exactly which model you bought.

**The one thing to know.** `~> 6.0` means "**6.anything**, but not 7". Major versions can break your code, so you only move to them on purpose.

**Try it now:** in a lab folder, pin `hashicorp/local` to `= 2.4.0`, `init`, then change it to `~> 2.5` and run `init` again. Read the error, then fix it with `terraform init -upgrade`.

**You've got it if you can:** explain what `~> 2.5` and `~> 2.5.0` allow, and why the lock file is committed.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### A provider is a program

A provider is a **separate executable** (written in Go, usually with HashiCorp's Plugin Framework). Terraform core starts it as a child process and talks to it over **gRPC** (topic 03). Every provider gives core three things:

1. **A schema**: every resource type and data source it supports, each with its arguments and attributes, their types, and which are required, optional, computed (set by the provider) or force replacement.
2. **Operations**: functions core can call: validate config, read, plan a change, apply a change, import.
3. Since Terraform 1.8: optional **provider-defined functions**, called like `provider::aws::arn_parse(...)` (topic 18).

There are thousands of providers:

| Tier | Who maintains it | Example source |
|---|---|---|
| **Official** | HashiCorp | `hashicorp/aws`, `hashicorp/azurerm`, `hashicorp/google`, `hashicorp/kubernetes`, `hashicorp/random`, `hashicorp/local` |
| **Partner** | The vendor, verified by HashiCorp | `cloudflare/cloudflare`, `datadog/datadog`, `integrations/github` |
| **Community** | Individuals or groups | `kreuzwerker/docker` (the Docker provider you'll use in topic 26) |

### Source addresses

```
[<HOSTNAME>/]<NAMESPACE>/<TYPE>
registry.terraform.io / hashicorp / aws
```

- **Hostname**: which registry. Defaults to `registry.terraform.io` (so you write `hashicorp/aws`). A private registry has its own hostname (`app.terraform.io/acme/internal`).
- **Namespace**: the publisher.
- **Type**: the provider's name. Resource types start with it: `aws_instance`, `aws_s3_bucket`.

The **local name** (the key in `required_providers`, e.g. `aws = {...}`) is what you use inside your code. By convention it equals the type.

### Version constraints

Both `required_version` (Terraform CLI) and provider `version` use the same syntax:

| Constraint | Allows | Use for |
|---|---|---|
| `= 2.5.3` or `2.5.3` | Exactly 2.5.3 | Rare: freezing a buggy release out |
| `>= 2.5` | 2.5 and anything newer, **including 3.0, 4.0** | `required_version` in modules (broad) |
| `>= 2.5, < 3.0` | 2.5 up to, not including, 3.0 | Explicit range |
| `~> 2.5` | `>= 2.5, < 3.0` (the **rightmost** number may grow) | ✅ Providers in root configs |
| `~> 2.5.0` | `>= 2.5.0, < 2.6.0` (patches only) | Very cautious pinning |
| `!= 2.6.1` | Anything except 2.6.1 | Skipping one bad release |

`~>` is called the **pessimistic constraint operator**. Read `~> X.Y` as "at least X.Y, still in major X".

Versions follow **semantic versioning** (SemVer): `MAJOR.MINOR.PATCH`. **Major** = may break your code. **Minor** = new features, compatible. **Patch** = fixes.

Rule of thumb:
- **Root configurations** (what you apply): `~> MAJOR.MINOR` for providers, and the lock file pins the exact version.
- **Reusable modules** (topic 44): a broad minimum like `>= 6.0`, so they don't fight the root's choice.

### The dependency lock file

`.terraform.lock.hcl` records, for each provider, the **exact version selected** and **checksums** of the package. On the next `init`, Terraform installs **that exact version** (even if a newer one matching the constraint exists) and refuses a package whose checksum doesn't match. It covers **providers only**, not modules.

### Provider configuration blocks

```hcl
provider "aws" {
  region  = "ap-south-1"
  profile = "lab"                 # credentials: topic 30
}
```

Arguments here configure **how the provider connects**: region, endpoint, credentials, default tags. Each provider documents its own (on its Registry page, "Argument Reference" under the provider overview).

Some providers need no configuration (`local`, `random`); then you can skip the `provider` block entirely.

### Aliases: the same provider, configured twice

```hcl
provider "aws" {                  # the default configuration (no alias)
  region = "ap-south-1"
}

provider "aws" {
  alias  = "us"                   # a second configuration, named aws.us
  region = "us-east-1"
}

resource "aws_s3_bucket" "logs_mumbai" {
  bucket = "acme-logs-mumbai"     # uses the default aws provider
}

resource "aws_s3_bucket" "logs_virginia" {
  provider = aws.us               # meta-argument: use the aliased configuration
  bucket   = "acme-logs-virginia"
}
```

`provider = aws.us` is a **meta-argument**: an argument Terraform core understands on **every** resource, regardless of provider. You'll meet the others (`depends_on`, `count`, `for_each`, `lifecycle`) in Part 2.

Common uses: several regions, several AWS accounts, a CloudFront certificate that must be in `us-east-1`, two Kubernetes clusters.

---

## Why it exists

- **Separation**: core stays small and stable; 4,000+ providers are released on their own schedules by the people who know each API.
- **Version constraints + lock file**: provider releases change behaviour. Without pinning, the same code could plan differently on Monday and Tuesday, or on your laptop and in CI. The lock file makes `init` **reproducible**.
- **Checksums**: a compromised download (supply-chain attack) is refused.

---

## Analogy

**Travel power adapters.**

- Your laptop (core) works everywhere, but each country (platform) has a different socket.
- `required_providers` = your packing list: "UK adapter, version 2.x".
- `init` = buying the adapters; the **lock file** = the receipt with the exact model number and a tamper-proof seal (checksum). Next trip, you must bring **that** model unless you deliberately upgrade (`init -upgrade`).
- **Aliases** = two UK adapters, one labelled "office", one labelled "hotel".

---

## How it works inside

What `terraform init` does for providers:

```
1. Read every required_providers block (root + all modules).
   Combine constraints:  root says "~> 6.0", a module says ">= 5.80"
                         → allowed: 6.x
2. Read .terraform.lock.hcl if it exists.
   Locked 6.21.0 and it satisfies the constraints? → use 6.21.0 (no "latest" lookup)
   Locked version no longer satisfies them?        → ERROR (unless -upgrade)
3. Ask the registry for available versions and download URLs:
   GET https://registry.terraform.io/v1/providers/hashicorp/aws/versions
   GET .../hashicorp/aws/6.21.0/download/linux/amd64
4. Download the zip + SHA256SUMS + signature.
   Verify the signature with the publisher's GPG key, then the checksum.
   (Or link it from plugin_cache_dir if already there.)
5. Unpack into .terraform/providers/registry.terraform.io/hashicorp/aws/6.21.0/linux_amd64/
6. Write/update .terraform.lock.hcl with the version and hashes.
```

When you later run `plan`, core starts that binary, asks for its **schema** over gRPC, and then makes calls like `PlanResourceChange` and `ApplyResourceChange`. The `_x5` at the end of a binary's name (`terraform-provider-aws_v6.21.0_x5`) is the plugin protocol version (5); newer providers often use protocol 6.

---

## Code / commands

### A full `versions.tf`

```hcl
terraform {
  required_version = ">= 1.14, < 2.0"

  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
  }
}
```

### Inspect providers

```bash
terraform providers
```

```
Providers required by configuration:
.
└── provider[registry.terraform.io/hashicorp/local] ~> 2.5

Providers required by state:

    provider[registry.terraform.io/hashicorp/local]
```

- **By configuration**: what your code asks for, with constraints (and, later, per module).
- **By state**: providers that resources in state were created with. A provider needed by state but removed from config is an error until those resources are gone.

```bash
terraform version
```

```
Terraform v1.17.2
on linux_amd64
+ provider registry.terraform.io/hashicorp/local v2.5.3
```

### The lock file, line by line

```hcl
# This file is maintained automatically by "terraform init".
# Manual edits may be lost in future updates.

provider "registry.terraform.io/hashicorp/local" {
  version     = "2.5.3"
  constraints = "~> 2.5"
  hashes = [
    "h1:mC9+u1eaUILTjxey6Ivyf/3djm//RNNze9kBVX/trng=",
    "zh:0c4d7e0b5b6c2f7e1d9a...",
    "zh:3a8f2e1c9b7d6a5e4f3c...",
    # ...one zh: line per platform package
  ]
}
```

| Line | Meaning |
|---|---|
| `provider "registry.terraform.io/hashicorp/local"` | The **full** source address. |
| `version = "2.5.3"` | The exact version selected. Everyone gets this. |
| `constraints = "~> 2.5"` | The constraints at the time (for information). |
| `h1:` | A hash of the **unpacked** package, recorded for the platform(s) you've installed on. |
| `zh:` | Hashes of the **zip files** for every platform, from the publisher's signed `SHA256SUMS`. They let another OS/CPU verify its download. |

When teammates use different platforms (macOS laptop, Linux CI), the `zh:` hashes usually cover it. If you install through a **mirror** (a private copy of the registry, topic 62) or cache that only gives `h1:` checks, pre-record hashes for every platform:

```bash
terraform providers lock -platform=linux_amd64 -platform=darwin_arm64 -platform=windows_amd64
```

### Upgrading on purpose

```bash
terraform init -upgrade
```

```
Initializing provider plugins...
- Finding hashicorp/local versions matching "~> 2.5"...
- Installing hashicorp/local v2.6.0...
- Installed hashicorp/local v2.6.0 (signed by HashiCorp)
Terraform has made some changes to the provider dependency selections recorded
in the .terraform.lock.hcl file. Review those changes and commit them to your
version control system if they represent changes you intended to make.
```

`-upgrade` ignores the locked version and picks the **newest** version allowed by the constraints. Commit the changed lock file in a PR, and read the provider's changelog first (topic 62).

### Seeing a provider's schema

```bash
terraform providers schema -json | jq '.provider_schemas["registry.terraform.io/hashicorp/local"].resource_schemas.local_file.block.attributes.filename'
```

```json
{
  "type": "string",
  "description": "The path to the file that will be created...",
  "description_kind": "plain",
  "required": true
}
```

That's what `validate` checks against (topic 08) and what the Registry docs are generated from.

---

## Hands-on lab

1. `mkdir -p ~/tf-lab/10-providers && cd ~/tf-lab/10-providers`.
2. Pin an **old** version exactly:

   ```hcl
   # versions.tf
   terraform {
     required_providers {
       local = { source = "hashicorp/local", version = "= 2.4.0" }
     }
   }
   ```
   ```hcl
   # main.tf
   resource "local_file" "x" {
     filename = "${path.module}/x.txt"
     content  = "x"
   }
   ```
   `terraform init`, then `terraform version` and `cat .terraform.lock.hcl`.
3. Change the constraint to `"~> 2.5"` and run `terraform init`. Read the error (see "When it breaks"). Then run `terraform init -upgrade` and look at the lock file's `version` again.
4. `ls -R .terraform/providers` and find the binary. Run it directly: `./.terraform/providers/registry.terraform.io/hashicorp/local/*/linux_amd64/terraform-provider-local_*`. It prints a message saying it's a plugin meant to be run by Terraform: proof that it's a separate program.
5. Check the cache: `ls ~/.terraform.d/plugin-cache/registry.terraform.io/hashicorp/local/` (if you set it up in topic 06).
6. Run `terraform providers schema -json | jq '.provider_schemas | keys'`.
7. Clean up: `rm -rf ~/tf-lab/10-providers`.

---

## Where you'll use it

- Every project starts with `versions.tf`.
- **Multi-region / multi-account** AWS setups use provider aliases (topic 30).
- **Provider upgrades** are a regular maintenance task: Dependabot or Renovate (bots that watch for new dependency versions) open PRs that change the lock file (topic 62).
- **Supply-chain security**: committed lock files with checksums are part of audits.

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `locked provider registry.terraform.io/hashicorp/local 2.4.0 does not match configured version constraint ~> 2.5; must use terraform init -upgrade to allow selection of new versions` | The constraint changed and no longer fits the locked version. | `terraform init -upgrade`, then commit the lock file. |
| `Error: Inconsistent dependency lock file` | A provider in the code isn't in the lock file (someone added it without committing the lock file), or the reverse. | `terraform init`, commit the updated lock file. |
| `Error: Failed to query available provider packages ... no available releases match the given constraints >= 5.0, < 5.50, ~> 6.0` | Two modules want versions that don't overlap: a **provider version conflict**. | Upgrade the module that pins the old version, or loosen a constraint. `terraform providers` shows who wants what. |
| `the current package for registry.terraform.io/hashicorp/aws 6.21.0 doesn't match any of the checksums previously recorded in the dependency lock file` | Download differs from the recorded hash: a different platform (with a mirror/cache) or, rarely, tampering. | `terraform providers lock -platform=...` for all your platforms. If the platform is the same, **investigate** before overriding. |
| `Error: Unsupported Terraform Core version` | Your CLI version doesn't match `required_version`. | Install a matching version (topic 06). |
| `Error: Provider configuration not present` | Resources in state used a provider (or alias) that's no longer in the code. | Put the provider block back until those resources are destroyed or moved. |
| `Could not retrieve the list of available versions for provider hashicorp/docker` | Wrong namespace: there is no `hashicorp/docker`. Implicit lookup guessed it. | Declare `source = "kreuzwerker/docker"` in `required_providers`. |

---

## Quick recap

- A provider is a separate plugin program; core asks it for its **schema** and calls it over gRPC.
- Declare providers in `required_providers` with a **source** (`namespace/type`) and a **version constraint**; `~> X.Y` = "this major only".
- `init` downloads, verifies signatures and checksums, and writes `.terraform.lock.hcl`. **Commit it.** `init -upgrade` moves versions on purpose.
- `provider` blocks configure connections; `alias` + `provider = aws.us` gives several configurations.

### Interview questions

1. **Q: What is `.terraform.lock.hcl` and should it be committed?**
   A: The dependency lock file. It records the exact provider versions selected and their checksums so every `init` installs the same verified packages. Yes, commit it; upgrade deliberately with `init -upgrade`.
2. **Q: What does `version = "~> 5.40"` allow?**
   A: Any version >= 5.40 and < 6.0. With `~> 5.40.0`, only 5.40.x patches.
3. **Q: How do you create resources in two AWS regions in one configuration?**
   A: Define two `provider "aws"` blocks, one with an `alias` and a different `region`, and set `provider = aws.<alias>` on the resources that should use it.
