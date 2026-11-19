# 62 · Upgrades: Terraform, Providers and Modules

> **Level:** ★★★ advanced
>
> **You will learn:** how to keep Terraform, providers and modules current **without breaking production**: the Terraform 1.x **compatibility promise** and what can still change, `required_version` strategy, rolling CLI upgrades across laptops and CI, **minor** provider upgrades through the lock file vs **major** upgrades with upgrade guides (e.g. AWS provider v5 → v6), module upgrades, **multi-platform lock files**, **provider mirrors** for air-gapped environments, and a repeatable upgrade runbook.
>
> **You need:** topics 06 (installing, tenv), 10 (constraints, lock file), 34 (schema versions), 45 (module versions), 57 (CI), 61 (drill 4).
>
> **Part 10 · Running it in production** · Previous: [61 · Capstone part 3: day-2 operations](61-capstone-3-day-2.md) · Next: [63 · Terraform at scale](63-terraform-at-scale.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**Three things get upgraded, each differently:**

| What | How it's pinned | How to upgrade | Main risk |
|---|---|---|---|
| **Terraform CLI** | `required_version` + the version installed in CI/laptops | Bump CI and `.terraform-version` together | Newer CLI writes state older CLIs can't read |
| **Providers** | Constraint in `required_providers` + exact version in `.terraform.lock.hcl` | `terraform init -upgrade` (within constraints); change the constraint for majors | Changed defaults/schemas → unexpected diffs; majors remove things |
| **Modules** | `version` / `?ref=` | Bump in a PR, `init -upgrade` | Interface changes, address changes |

The safe loop for **any** upgrade:

```
branch → bump ONE thing → terraform init -upgrade → plan EVERY affected root
       → read changelog + every diff → fix → PR → dev first → prod
```

**Picture it.** Servicing a fleet of delivery vans: you don't change the engine, gearbox and tyres on all vans on the same day. One part at a time, one test van first, then the fleet.

**The one thing to know.** A pure version upgrade should produce **"No changes"**. Any diff is the upgrade telling you something changed; understand it before you apply.

**Try it now:** in a lab with `version = "~> 2.4"` for `hashicorp/local` locked to an old version, run `terraform init -upgrade`, look at the lock-file diff, and plan.

**You've got it if you can:** describe the runbook for a major provider upgrade.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Terraform CLI upgrades

- **1.x compatibility promise** (since 1.0): configurations, state files and the provider protocol written for 1.x keep working on later 1.x versions. Upgrading 1.12 → 1.17 is normally safe to do in one step.
- What can still bite you:
  - **State written by a newer CLI** may not be readable by an older one. Once CI applies with 1.17, a teammate on 1.12 may get `state snapshot was created by Terraform v1.17...`. So **upgrade everyone together**.
  - **Deprecations** produce warnings first; read them (e.g. S3 backend DynamoDB arguments, topic 36).
  - **New behaviour** behind opt-in features is fine; check the changelog's "upgrade notes".
- **`required_version`** strategy:
  - Root modules: a floor matching your team's version, e.g. `>= 1.14, < 2.0`, plus an exact version in CI and `.terraform-version`.
  - Shared modules: the lowest version whose features you use (`>= 1.9`).
- Rollout: one PR that bumps the CI version (`TF_VERSION`), `.terraform-version` and (if needed) `required_version`; run plans for every root; merge; tell the team to update tenv.

### Provider upgrades

**Minor/patch** (e.g. AWS 6.70 → 6.73, inside `~> 6.0`):

- `terraform init -upgrade` picks the newest allowed version and updates `.terraform.lock.hcl`.
- Bots (Dependabot/Renovate) open these PRs (topic 61).
- Usually no config changes; plans should be clean. Read the changelog for bug fixes that change behaviour.

**Major** (e.g. AWS 5.x → 6.x):

- Providers publish an **upgrade guide** (AWS: "Terraform AWS Provider Version 6 Upgrade Guide" on the Registry). It lists removed resources/arguments, renamed attributes, changed defaults.
- Examples from AWS v6: the **per-resource `region` argument** (topic 30), `data.aws_region.name` deprecated in favour of `.region`, several long-deprecated arguments and resources removed, changed behaviour for some S3 and EC2 settings.
- Steps:
  1. Upgrade to the **latest minor of the old major** first, and fix every **deprecation warning** (they're the future errors).
  2. Read the upgrade guide end to end; search your code for each item.
  3. Change the constraint (`~> 5.0` → `~> 6.0`) in one PR; `init -upgrade`.
  4. Plan **every** root; fix errors; explain every diff (or add `moved` blocks where resources were renamed).
  5. Apply dev → staging → prod.
  6. Shared modules: widen their constraint (`>= 5.0` → `>= 5.0, < 7.0` or `>= 6.0`) and release a module version.

### Module upgrades

- Registry modules: bump `version`, `init -upgrade`, plan. Majors usually have an `UPGRADE-x.0.md` in the repo.
- Your own modules: follow SemVer (topic 45), ship `moved` blocks inside the module for internal renames so callers upgrade without state surgery.
- Bump one module at a time per PR when changes are big.

### Lock files across platforms

The lock file must contain hashes valid for every OS/CPU that runs `init` (laptops on macOS ARM, Windows, Linux CI). If your CI uses `-lockfile=readonly` (topic 57), an incomplete lock file fails:

```bash
terraform providers lock \
  -platform=linux_amd64 -platform=linux_arm64 \
  -platform=darwin_arm64 -platform=windows_amd64
```

Run it after every provider version change (bots can be configured to do so).

### Provider mirrors (air-gapped or controlled environments)

When runners can't reach `registry.terraform.io`:

```bash
terraform providers mirror /srv/terraform-mirror     # downloads every required provider for this config
```

```hcl
# ~/.terraformrc (or TF_CLI_CONFIG_FILE) on the runners
provider_installation {
  filesystem_mirror {
    path    = "/srv/terraform-mirror"
    include = ["registry.terraform.io/*/*"]
  }
  direct {
    exclude = ["registry.terraform.io/*/*"]          # never fall back to the internet for these
  }
}
```

Or a **network mirror** (an HTTPS server/artifact repository like Artifactory or Nexus serving the provider mirror protocol). The lock file's checksums still protect against tampering.

---

## Why it exists

Providers release every week or two; Terraform minor versions every few months. Falling behind makes upgrades bigger and riskier (and misses security fixes and new resources); upgrading carelessly causes surprise diffs in production. A boring, repeatable process makes upgrades routine.

---

## Analogy

**Servicing a delivery fleet.** You schedule maintenance (bots), change one part at a time (one upgrade per PR), try it on a test van first (dev), read the manufacturer's recall notices (changelogs, upgrade guides), and make sure every depot stocks the right spare parts (multi-platform lock files, mirrors) so no van is stuck.

---

## How it works inside

```
terraform init -upgrade
  for each provider: query registry versions → newest matching ALL constraints (root + modules)
  download, verify signatures/checksums → rewrite .terraform.lock.hcl (version + hashes)
  modules: re-resolve versions/refs → re-download
terraform plan (with the new provider)
  provider upgrades each instance's state to its new schema_version in memory (UpgradeResourceState)
  → compares config with upgraded state → diffs = behaviour/default changes
terraform apply
  writes upgraded state (older providers may no longer read it → no easy rollback)
```

That last line is why you test upgrades in dev first: rolling back a provider **after** applying with it may require restoring the state version from before (topic 42).

---

## Code / commands

### A minor provider upgrade

```bash
git switch -c chore/aws-provider-6.73
cd envs/dev/web
terraform init -upgrade
git diff .terraform.lock.hcl
```

```diff
 provider "registry.terraform.io/hashicorp/aws" {
-  version     = "6.70.0"
+  version     = "6.73.0"
   constraints = "~> 6.0"
   hashes = [
-    "h1:Xo2...",
+    "h1:9Lq...",
```

```bash
terraform providers lock -platform=linux_amd64 -platform=darwin_arm64 -platform=windows_amd64
terraform plan
```

```
No changes. Your infrastructure matches the configuration.
```

Repeat for every root (or let CI plan them all), open the PR, merge, apply dev → prod.

### A major provider upgrade (shape of the work)

```bash
git switch -c chore/aws-provider-v7          # hypothetical future major
# 1. on the old major, fix deprecations
terraform plan 2>&1 | grep -A3 -i deprecat
# 2. bump constraints everywhere
grep -rl 'hashicorp/aws' --include=*.tf . | xargs sed -i 's/version = "~> 6.0"/version = "~> 7.0"/'
# 3. upgrade and plan every root
for r in envs/*/*; do (cd "$r" && terraform init -upgrade -input=false >/dev/null && terraform plan -no-color > /tmp/plan-$(echo $r | tr / -).txt; echo "$r: $?"); done
```

Then go through each plan file, fixing errors and explaining every diff, before opening the PR.

### A Terraform CLI upgrade

```bash
tenv tf install 1.17.2 && echo "1.17.2" > .terraform-version
sed -i 's/TF_VERSION: "1.16.7"/TF_VERSION: "1.17.2"/' .github/workflows/tf-root.yml
terraform version
for r in envs/*/*; do (cd "$r" && terraform init -input=false >/dev/null && terraform plan -detailed-exitcode >/dev/null; echo "$r exit=$?"); done
```

```
envs/dev/network exit=0
envs/dev/web exit=0
envs/prod/network exit=0
envs/prod/web exit=0
```

All `0`: no changes. Merge, then announce "everyone: `tenv tf use 1.17.2`".

---

## Hands-on lab

1. In a scratch folder pin `hashicorp/local` to `= 2.4.0`, apply something, then change to `~> 2.4` and run `init -upgrade`. Inspect the lock-file diff; plan.
2. Generate multi-platform hashes with `providers lock`; compare the number of `zh:`/`h1:` lines before and after.
3. Read the AWS provider v6 upgrade guide on the Registry and list three items that would affect topics 31–33's code (hint: `aws_region` `.name`).
4. Set up a filesystem mirror for one lab: `terraform providers mirror`, a `.terraformrc` with `filesystem_mirror`, then `init` offline (disable network in WSL temporarily or point `direct` exclude as above).
5. Do a CLI version bump PR on `acme-infra` and plan all roots.

---

## Where you'll use it

- **Weekly**: bot PRs for providers and modules.
- **Quarterly**: Terraform CLI bumps across the team and CI.
- **Yearly-ish**: major provider upgrades as small projects with a plan, a branch and staged rollout.
- **Regulated/air-gapped environments**: provider mirrors and strict lock files.

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `state snapshot was created by Terraform v1.17.2, which is newer than current v1.16.x` | Someone/CI upgraded first. | Upgrade your CLI; align versions everywhere. |
| `locked provider ... does not match configured version constraint` | Constraint changed, lock not updated. | `init -upgrade`; commit the lock file. |
| `the cached package ... does not match any of the checksums recorded` in CI | Lock file lacks hashes for CI's platform. | `terraform providers lock -platform=...`. |
| After upgrade: `Unsupported argument` / `Unsupported block type` | Removed in the new major. | Follow the upgrade guide; replace with the new resource/argument. |
| After upgrade: plan shows diffs you didn't make | New defaults or normalisation. | Read the changelog; set values explicitly; or accept after review. |
| Can't downgrade a provider after apply | State upgraded to a newer schema. | Restore the pre-upgrade state version (topic 42) **and** pin back. Test in dev first. |
| `Failed to query available provider packages` offline | No access to the registry. | Provider mirror / network mirror. |

---

## Quick recap

- CLI: 1.x is compatible, but newer CLIs write state older ones can't read: upgrade **everyone together** (CI, `.terraform-version`, `required_version`).
- Providers: minors via `init -upgrade` + lock file (bots); majors via the upgrade guide, after clearing deprecations, root by root, dev first.
- Modules: bump versions per PR; module authors ship `moved` blocks and follow SemVer.
- Keep lock files multi-platform; use mirrors where the internet isn't available.
- A clean upgrade plans **No changes**; every diff must be understood.

### Interview questions

1. **Q: How do you upgrade the AWS provider to a new major version safely?**
   A: First upgrade to the latest minor of the current major and fix all deprecation warnings. Then read the upgrade guide, bump the constraint in a branch, run `init -upgrade`, plan every root, fix errors and explain every diff, and apply through dev and staging before prod. Shared modules widen their constraints and release a new version.
2. **Q: Why must a team upgrade the Terraform CLI together?**
   A: Once a newer CLI writes the state, older CLIs may refuse to read it. CI, laptops (`.terraform-version`/tenv) and `required_version` should move in one coordinated change.
3. **Q: What is a provider mirror and when do you need one?**
   A: A local or network copy of provider packages (`terraform providers mirror`, `provider_installation` in the CLI config) used where runners can't reach the public registry or the organisation wants to control which providers are used. Lock-file checksums still verify them.
