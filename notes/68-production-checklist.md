# 68 · Production-Readiness Checklist

> **Level:** ★★ intermediate (reference)
>
> **You will learn:** nothing new. This is a **one-page checklist** to review before a Terraform repository is allowed to manage production, grouped by area, with the topic that explains each item. Use it for your own repos, for reviewing someone else's, and as an interview answer to "what makes a Terraform setup production-ready?".
>
> **You need:** the topics referenced.
>
> **Part 11 · Interviews and reference** · Previous: [67 · What's new and what's next](67-whats-new.md) · Next: [69 · Interview question bank](69-interview-question-bank.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Production-ready Terraform = **safe state** + **controlled change** + **secure code** + **recoverability** + **ownership**. If you can tick every box in the eight sections below, the setup is in good shape.

**Picture it.** A pilot's pre-flight checklist: not because pilots forget how to fly, but because skipping one item occasionally is how accidents happen.

**The one thing to know.** The five boxes that prevent most disasters: **versioned + locked remote state**, **plan-reviewed PR workflow with saved-plan applies**, **no long-lived cloud keys**, **`prevent_destroy`/deletion protection on data**, **no secrets in state**.

**Try it now:** score your capstone (topics 59–61) against this list.

**You've got it if you can:** name an item from each section from memory.

> 📖 **That's the core.** The checklist follows.

---

## What it is / Why it exists / Analogy / How it works inside

A list of controls distilled from Parts 5–10. Each item exists because its absence caused real incidents (topic 66). Like a pre-flight checklist, it's short, binary (yes/no) and reviewed every time, not only once.

---

## Code / commands: the checklist

### 1. State

- [ ] Remote backend (S3/azurerm/gcs/HCP Terraform); **no** local state for shared environments (35)
- [ ] State bucket: **versioning**, **encryption** (KMS where required), **public access blocked**, **TLS-only** policy, `prevent_destroy`, lifecycle expiry of old versions (35)
- [ ] **Locking** enabled (`use_lockfile`; DynamoDB migrated off) (36)
- [ ] **One state per environment × component**; sizes keep plans fast (38, 49, 63)
- [ ] Prod state in the **prod account** (or at least separate keys and IAM), readable only by the pipeline and break-glass admins (35, 49)
- [ ] State restore **rehearsed** (S3 version restore; delete-marker removal) (42)

### 2. Code structure

- [ ] Thin root modules per environment; shared, **versioned** modules (43–47, 49)
- [ ] `required_version` and `required_providers` everywhere; **`.terraform.lock.hcl` committed** with multi-platform hashes (10, 62)
- [ ] Modules: no provider blocks, typed/validated inputs, secure defaults, documented outputs, examples, tests (44, 55)
- [ ] `for_each` (not `count`) for named things; `moved`/`removed` blocks for refactors (21, 22, 40)
- [ ] No deprecated patterns: `null_resource`, provisioners (except documented exceptions), `taint`, import commands in runbooks (52, 67)

### 3. Change process

- [ ] All changes via **PR**; branch protection; required checks and approvals; `CODEOWNERS` for prod paths (56)
- [ ] CI posts **plans on PRs** for every affected root (57, 58, 60)
- [ ] Applies only from CI, using the **saved, reviewed plan**; prod applies need **human approval** (56, 57)
- [ ] **Concurrency** per state; never cancel applies; `-lock-timeout` (36, 57)
- [ ] Promotion order dev → staging → prod (49, 60)
- [ ] Emergency/break-glass procedure, with same-day follow-up PRs (37, 56)

### 4. Security

- [ ] **No long-lived cloud keys** in CI: OIDC / instance roles (57, 58)
- [ ] Separate **plan (read-only)** and **apply (write)** roles; apply role usable only from the protected environment/branch (57)
- [ ] `allowed_account_ids` in every provider block (30)
- [ ] **No secrets in code, tfvars or state**: cloud-managed secrets, ephemeral values, write-only arguments (51)
- [ ] Plan artifacts and logs restricted and short-lived; CI masking (51, 57)
- [ ] Static scanning (Trivy/Checkov) and linting (TFLint) gating PRs; suppressions documented (53)
- [ ] **Policy as code** for tags, sizes, regions, protected deletes (54)
- [ ] Third-party modules/actions/providers pinned (versions, SHAs) and reviewed (45, 46, 57)
- [ ] Humans have read-only console access by default; guardrails (SCPs) (37)

### 5. Safety of stateful resources

- [ ] `prevent_destroy` **and** provider deletion protection on databases, buckets with data, KMS keys, DNS zones, state buckets (24, 47)
- [ ] Backups/snapshots for data stores, and `skip_final_snapshot = false` on databases (Terraform can't restore deleted data) (47, 56)
- [ ] `create_before_destroy` + name prefixes where replacement would cause downtime (24)
- [ ] Policy denies `delete` on protected types in automated runs (54)

### 6. Drift and observability

- [ ] **Scheduled drift detection** with alerts routed to the owning team (37, 61)
- [ ] `check` blocks / smoke tests after apply for critical endpoints (25, 33)
- [ ] `default_tags`: `owner`, `project`, `environment`, `managed_by`, `repo` (30)
- [ ] Unmanaged-resource visibility (AWS Config, tag inventory, `terraform query`) (37, 67)

### 7. Maintenance

- [ ] Bot PRs for providers/modules/actions; reviewed like any change (61, 62)
- [ ] Terraform CLI version pinned in CI and `.terraform-version`; upgraded together (06, 62)
- [ ] Module changelogs and SemVer tags (45)
- [ ] Tests (`terraform test`, mocks) for modules in CI (55)
- [ ] Runbooks: stale lock, failed apply, state restore, rotation (36, 42, 51, 66)

### 8. Ownership and documentation

- [ ] Every state has an **owner** (team) and an on-call path (63)
- [ ] README per root: what it manages, how to run it, who owns it, links to dashboards (44)
- [ ] Architecture/layout documented (which states exist and how they connect) (38, 49)

### A quick self-audit script (partial, for a repo)

```bash
echo "lock files committed:"; git ls-files '**/.terraform.lock.hcl' | wc -l
echo "local state committed (should be 0):"; git ls-files '*.tfstate*' | wc -l
echo "backend blocks:"; grep -rl 'backend "' --include=*.tf . | wc -l
echo "use_lockfile:"; grep -rl 'use_lockfile *= *true' --include=*.tf --include=*.tfbackend . | wc -l
echo "dynamodb_table (migrate off):"; grep -rl 'dynamodb_table' --include=*.tf --include=*.tfbackend . | wc -l
echo "allowed_account_ids:"; grep -rl 'allowed_account_ids' --include=*.tf . | wc -l
echo "provisioners (review):"; grep -rn 'provisioner "' --include=*.tf . | wc -l
echo "null_resource (migrate):"; grep -rn 'null_resource' --include=*.tf . | wc -l
echo "prevent_destroy:"; grep -rn 'prevent_destroy *= *true' --include=*.tf . | wc -l
```

---

## Hands-on lab

1. Score your capstone repo: count the ticked boxes per section.
2. Run the self-audit script; fix two findings.
3. Use the checklist to review a public Terraform repository and write five recommendations.

---

## Where you'll use it

Go-live reviews, audits, onboarding to a new company's Terraform, and the interview question "what would you check in an existing Terraform setup?".

---

## When it breaks

A checklist only helps if it's used: put it in the PR template for new roots, review it quarterly, and add an item after every incident's post-mortem.

---

## Quick recap

Eight areas: **state, code structure, change process, security, stateful-resource safety, drift/observability, maintenance, ownership**. The top five: versioned locked remote state; reviewed saved-plan applies from CI; no long-lived keys; protected data resources; no secrets in state.

### Interview questions

1. **Q: What makes a Terraform setup production-ready?**
   A: Encrypted, versioned, locked remote state split per environment and component; all changes via PRs with plans, scans, tests and policies, applied by CI from the reviewed plan with approvals for prod; OIDC roles with read-only plan and restricted apply; no secrets in state; protected stateful resources with backups; drift detection; pinned versions with routine upgrades; and clear ownership.
2. **Q: You join a company with an existing Terraform repo. What do you check first?**
   A: Where state lives and whether it's versioned, locked and access-controlled; how changes are applied (who, from where, with what credentials); whether plans are reviewed; whether secrets are in state; and what protects databases from deletion.
3. **Q: Name three controls that prevent accidental data loss.**
   A: `prevent_destroy` plus provider deletion protection, a policy that blocks deletes of protected types in CI, and reviewed saved-plan applies, plus backups or snapshots because Terraform can't restore deleted data.
