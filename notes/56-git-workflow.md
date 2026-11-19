# 56 · The Git Workflow for Terraform

> **Level:** ★★ intermediate
>
> **You will learn:** the team workflow that every Terraform pipeline implements: **branch → PR → automatic checks and plan → review of the plan → merge → apply**, the two main variants (**apply after merge** vs **apply before merge**), why you **apply the plan you reviewed**, one change per root, small PRs, `CODEOWNERS` and branch protection, how to read a plan **in a PR**, emergency changes, and why "rollback" in Terraform means **roll forward with a revert**.
>
> **You need:** topics 05 (Git), 09 (saved plans, stale plans, exit codes), 36 (locking), 37 (drift), 49 (layout), 53–55. Background: [GitHub Actions topic 02](../../GitHub-Actions-Basics-To-Production/notes/02-git-and-github-recap.md), [30](../../GitHub-Actions-Basics-To-Production/notes/30-branch-protection-and-rulesets.md); [Jenkins topics 08](../../Jenkins/notes/08-branching-strategies.md), [45](../../Jenkins/notes/45-trunk-based-flow-with-mbp.md).
>
> **Part 9 · CI/CD** · Previous: [55 · Testing Terraform](55-testing.md) · Next: [57 · Terraform in GitHub Actions](57-github-actions.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** In a team, **nobody runs `terraform apply` from a laptop** against shared environments. Changes flow through Git:

```
feature branch ──push──▶ Pull Request
                           │ CI: fmt · validate · lint · scan · test · PLAN (posted on the PR)
                           │ humans review the CODE and the PLAN
                           ▼
                         merge to main
                           │ CI: plan again → (approval for prod) → APPLY that saved plan
                           ▼
                         infrastructure updated; state in the backend; plan + logs kept
```

**Picture it.** A building company's change-order process: a written request (PR), an itemised quote (plan) that the client signs (review), and the crew builds **exactly** what's on the signed quote (apply the saved plan).

**The one thing to know.** Reviewers approve the **plan**, not just the diff. A 3-line code change can mean "destroy the database"; only the plan shows that.

**Try it now:** think of your last lab change. Write the PR description you'd want a reviewer to read: what, why, and the plan summary (`Plan: X to add, Y to change, Z to destroy`) with any replacements explained.

**You've got it if you can:** draw the flow, name the two variants, and explain why the apply must use the reviewed plan.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### The standard flow ("apply after merge")

| Step | Who/what | Detail |
|---|---|---|
| 1. Branch | Engineer | `git switch -c feat/add-redis` from `main` |
| 2. Local checks | Engineer + pre-commit | `fmt`, `validate`, TFLint, a local `plan` against dev if allowed |
| 3. Open PR | Engineer | Description: what, why, risk, rollback idea |
| 4. CI on the PR | Pipeline (read-only role) | fmt/validate/lint/scan/test → `plan` for every **affected root** → plan posted as a PR comment → policy checks |
| 5. Review | `CODEOWNERS` | Code **and plan**: unexpected destroys/replacements, IAM, security groups, costs |
| 6. Merge | Engineer | Branch protection requires green checks + approvals |
| 7. CI on `main` | Pipeline (write role) | `plan -out=tfplan` again (main may have moved) → **approval gate** for prod → `apply tfplan` |
| 8. Verify | Pipeline + humans | Smoke tests, `check` blocks, monitoring |

### Variant: "apply before merge" (Atlantis style)

```
PR → plan → review → comment "apply" → apply from the PR branch → merge after success
```

| | Apply **after** merge | Apply **before** merge |
|---|---|---|
| `main` means | "What we intend" (may briefly be ahead of reality) | "What's actually deployed" |
| Failed apply | Fix with a new PR | Fix in the same PR before merging |
| Concurrency | Main is serialised by the pipeline | Needs PR-level locks (Atlantis locks the project per PR) |
| Typical with | GitHub Actions / Jenkins / GitLab CI | Atlantis, some platforms |

Both are fine. The key rules are the same: **reviewed plan = applied plan**, and **one writer per state at a time**.

### Apply what you reviewed

The plan on the PR was made against the state **at that time**. Between review and apply:

- someone else may have merged and applied (state changed → the old plan is **stale**, topic 09),
- drift may have happened.

So the pipeline on `main` **re-plans**, saves `tfplan`, shows it (and for prod, waits for approval), then runs `terraform apply tfplan`. If anything changed between the plan and the apply, Terraform refuses the stale plan instead of doing something nobody reviewed.

### Rules that keep it safe

| Rule | Why |
|---|---|
| **No humans apply to shared envs from laptops** | One path, one audit trail, the right credentials (topic 57: only CI's role can write). Break-glass exceptions are logged. |
| **Small PRs, one concern each** | Reviewable plans. "Upgrade provider" and "add Redis" are two PRs. |
| **Changes affect few roots** | A PR that plans 12 roots is hard to review; split or sequence it. |
| **`CODEOWNERS` per path** | `envs/prod/` → platform leads; `modules/network/` → network team. |
| **Branch protection** | Required checks (plan, scans, tests), required approvals, no direct pushes to `main`. |
| **Concurrency per state** | One pipeline run per root at a time (topic 36). |
| **Plans and logs retained** | For audits and incident analysis (but plans contain secrets: restricted access, limited retention). |

### Reading a plan in a PR (reviewer checklist)

1. The **summary line**: does `X to add, Y to change, Z to destroy` match the PR description?
2. Every **`-/+`** and **`-`**: why? Look for `# forces replacement` (topics 24, 41).
3. **Stateful resources** (databases, buckets, volumes, KMS keys, DNS zones): any destroy is a red flag.
4. **Security-relevant changes**: IAM policies, security groups, public access, encryption.
5. **`(known after apply)` cascades**: a small change rippling through dependents.
6. **Drift notes** ("changed outside of Terraform"): will this apply revert someone's fix?
7. **Cost**: Infracost comment (topic 54).

### Emergency changes

During an incident, you may need to change something **now**:

1. Prefer a fast PR with an expedited review (keep the pipeline fast so this is realistic).
2. If you must click in the console (break-glass role), **record it**, and the same day open a PR that **adopts or reverts** it (topic 37).
3. Never leave the next unrelated apply to silently undo an emergency fix.

### "Rollback" = roll forward

Terraform has no undo button. To go back, **revert the commit** (`git revert`), open a PR, review the plan, apply. That plan will **recreate**, **modify** or **destroy** things to match the old code, and it **can't** bring back deleted data (a destroyed database stays destroyed unless you restore from backups/snapshots). That's why destroys and replacements of stateful resources get special attention in review and policy (topic 54).

---

## Why it exists

Infrastructure changes are high-impact and often irreversible. The Git workflow brings the same safety software teams rely on (review, automated checks, audit trail, controlled deployment) to infrastructure, and Terraform's **plan** makes review meaningful: reviewers see the **effect**, not just the code.

---

## Analogy

**A building contractor's change orders.**

- The client asks for a change in writing (**PR**).
- The contractor sends an **itemised quote** (plan): "remove wall A, add window B". The quote even lists side-effects ("wall A holds the shelf, so the shelf is removed too").
- The client signs **that** quote (approval).
- The crew builds **exactly** what's on the signed quote. If the site changed since the quote (someone else built something), the crew stops and asks for a new quote (stale plan).
- Undoing means a **new** change order to rebuild the wall. The old plaster is gone (data).

---

## How it works inside

The Git events map to pipeline jobs (implemented in topics 57–58):

```
pull_request opened/updated ──▶ job "plan" (read-only cloud role, -lock-timeout, concurrency per root)
                                  └─ detect changed roots (paths) ─▶ for each: init → plan → comment
push to main                ──▶ job "plan" ─▶ artifact tfplan ─▶ job "apply" (env approval)
                                                                  └─ write role ─▶ apply tfplan
schedule (cron)             ──▶ job "drift" (topic 37)
```

---

## Code / commands

### A PR description template

```markdown
## What
Add a Redis cache (ElastiCache) for the app in dev.

## Why
Session storage moves out of the app instances (ticket APP-123).

## Plan summary
- envs/dev/app: Plan: 4 to add, 1 to change, 0 to destroy
  - + aws_elasticache_subnet_group, aws_elasticache_replication_group, SG + rule
  - ~ aws_security_group_rule app egress (adds 6379)
- No replacements. No changes to prod.

## Risk / rollback
Low. Rollback = revert this PR (destroys the cache; no data to keep).
```

### `CODEOWNERS`

```
# .github/CODEOWNERS
*                       @acme/platform
/modules/network/       @acme/network-team
/envs/prod/             @acme/platform-leads
/policy/                @acme/security
```

### Detecting which roots changed (used by CI)

```bash
git fetch origin main
git diff --name-only origin/main...HEAD \
  | grep -E '^envs/[^/]+/[^/]+/' \
  | cut -d/ -f1-3 | sort -u
```

```
envs/dev/app
envs/staging/app
```

(If a shared module changed, plan every root that uses it: grep for its `source` path.)

### Reverting

```bash
git revert <commit-sha>          # a new commit undoing the change
git push origin HEAD:revert-redis && gh pr create --fill
# review the plan: it shows exactly what the "rollback" will destroy or change
```

---

## Hands-on lab

1. In your `~/tf-lab` repo (push it to a private GitHub repo), protect `main`: require PRs, one approval and status checks (you'll add the checks in topic 57).
2. Add a `CODEOWNERS` file.
3. Make a change in a branch, run `terraform plan` locally, and write a PR description using the template, including the plan summary.
4. Review your own plan with the 7-point checklist; find any replacement and explain it.
5. Practise a revert of a merged lab change and read the resulting plan.

---

## Where you'll use it

- **Every team** using Terraform. Interviewers often ask you to describe this flow end to end.
- **Audits**: "show me who approved this production change" = the PR, its approvals, the plan and the pipeline log.
- **Incident reviews**: "the apply did something unexpected" → was the plan reviewed? Was the applied plan the reviewed one?

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| Apply did something the reviewer didn't see | Apply re-planned instead of using the saved plan, or plan was stale | Save plan → approve → `apply tfplan`; stale plans are rejected |
| Two merges, two pipelines, lock errors | No serialisation per root | Concurrency groups; `-lock-timeout` |
| Huge plans nobody can review | PRs mixing many concerns/roots | Small PRs; split roots |
| An apply undid an emergency fix | Drift not adopted | Same-day follow-up PRs; drift detection |
| "Rollback" didn't restore data | Revert recreated an empty resource | Backups/snapshots; protect stateful resources from deletion |
| Someone applied from a laptop | Humans have write credentials | Read-only humans; write only via the CI role (OIDC, topic 57) |

---

## Quick recap

- Flow: **branch → PR → checks + plan on PR → review code and plan → merge → re-plan → approval → apply the saved plan**.
- Variants: apply after merge (CI) or before merge (Atlantis); both keep "reviewed = applied" and one writer per state.
- Small PRs, `CODEOWNERS`, branch protection, concurrency, retained (restricted) plans.
- Emergency changes get same-day PRs; "rollback" is a revert, and can't restore deleted data.

### Interview questions

1. **Q: Describe your Terraform workflow in a team.**
   A: Changes go through pull requests. CI runs format, validation, linting, security scans and tests, and posts a plan for each affected root. Reviewers check the code and the plan. After merge, the pipeline plans again on main, saves the plan, waits for approval for production, and applies exactly that saved plan, with one run per state at a time.
2. **Q: Why apply a saved plan rather than running `terraform apply` again?**
   A: So what gets applied is exactly what was reviewed. If the state changed in between, Terraform rejects the stale plan rather than applying unreviewed changes.
3. **Q: How do you roll back a bad Terraform change?**
   A: Revert the commit and apply through the normal process, reviewing the plan. It moves infrastructure back to the old configuration, but it can't restore deleted data, which needs backups or snapshots.
