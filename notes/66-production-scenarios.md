# 66 · Production Break-Fix Scenarios

> **Level:** ★★★ advanced
>
> **You will learn:** to handle twelve realistic Terraform incidents end to end. Each has: how to **reproduce** it safely (mostly free, local or the free topic 32 network), the **symptom** you'd see, how to **diagnose** it, the **fix**, and the **prevention**. Together they rehearse everything in Parts 5–10 under pressure.
>
> **You need:** Parts 1–10. Each scenario lists its key topics.
>
> **Part 10 · Running it in production** · Previous: [65 · Error catalogue](65-error-catalogue.md) · Next: [67 · What's new and what's next](67-whats-new.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Real incidents are rarely "a new error message". They're **situations**: a scary plan on Friday afternoon, a pipeline stuck on a lock, a leaked secret. The skill is a calm sequence:

```
STOP (don't apply / don't force anything) → SCOPE (what's affected? is prod at risk?)
→ DIAGNOSE (phase + tool, topic 64) → FIX (smallest safe change, through review)
→ VERIFY (plan shows "No changes") → PREVENT (code, policy, process) → WRITE IT DOWN
```

| # | Scenario | Key topics |
|---|---|---|
| 1 | PR plan wants to replace the database | 24, 40, 54 |
| 2 | Pipeline stuck on a state lock | 36 |
| 3 | Port 22 opened in the console; plan is green | 37, 25 |
| 4 | Apply failed halfway | 13, 41 |
| 5 | State file deleted | 42 |
| 6 | Provider upgrade changes a default | 62 |
| 7 | Removing one server from a list rebuilds three | 21, 22, 40 |
| 8 | State with secrets committed to Git | 12, 35, 51 |
| 9 | Two pipelines applied at the same time | 36, 56, 57 |
| 10 | Destroy fails with `DependencyViolation` | 20, 32 |
| 11 | Dev code applied to the prod account | 30, 49 |
| 12 | A plan that never settles | 18, 24, 37 |

**The one thing to know.** In almost every scenario, the dangerous move is the **fast** one (apply anyway, force-unlock, `-target`, delete state). The plan is your friend: read it, change one thing, read it again.

**Try it now:** do scenarios 7 and 12 (free, 10 minutes each).

**You've got it if you can:** for any scenario, say the first thing you would **not** do, and why.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

Twelve scenarios in a fixed format: **Reproduce → Symptom → Diagnose → Fix → Prevent**.

## Why it exists

Interviews for senior roles and real on-call shifts test judgement, not syntax. Practising incidents in a safe lab builds the instincts.

## Analogy

Fire drills. You practise the evacuation when nothing is burning, so that when something is, you walk calmly to the right exit instead of running into the wrong room.

## How it works inside

All scenarios reduce to the three worlds (config, state, reality; topic 34) disagreeing in a particular way, or to the process around them (locks, pipelines, credentials) failing.

---

## Code / commands: the scenarios

### 1. The PR plan wants to replace the database

- **Reproduce** (free): use topic 44's bucket module as a stand-in for "the database". Rename the module call `module "bucket"` → `module "data_bucket"` without a `moved` block; plan.
- **Symptom:** every resource in the module is destroyed and created again (`N to add, 0 to change, N to destroy`); in real life, `aws_db_instance.main must be replaced` (e.g. after changing `identifier`, `engine` or the subnet group).
- **Diagnose:** find `# forces replacement` (argument change) vs destroy/create at **different addresses** (rename). Ask: what data lives there?
- **Fix:** rename → `moved` block (topic 40). Force-new argument → revert it, or plan a migration (snapshot → new instance → cut-over), never an in-place "just apply".
- **Prevent:** `prevent_destroy` + `deletion_protection` on stateful resources; the **protect** policy (topic 54) blocks deletes in CI; reviewers check the plan summary.

### 2. Pipeline stuck on a state lock

- **Reproduce:** topic 36's double Ctrl-C (or cancel a GitHub Actions apply mid-run).
- **Symptom:** every run fails: `Error acquiring the state lock` with `Who: runner@...` from hours ago.
- **Diagnose:** lock info; is that run/runner really gone? (Actions/Jenkins UI.)
- **Fix:** back up (`state pull`) → `force-unlock <ID>` → `plan` → handle any partial apply (scenario 4).
- **Prevent:** never `cancel-in-progress` for applies; concurrency per state; `-lock-timeout`; runbook with the checklist.

### 3. Port 22 opened in the console; every plan is green

- **Reproduce** (free): topic 37 drift 2 on the topic 32 network.
- **Symptom:** security alert or audit finding, but `terraform plan` says `No changes`.
- **Diagnose:** the rule is a **separate, unmanaged object** next to your separately managed rules: Terraform can't see it.
- **Fix:** revoke it (CLI or console, logged), or if it's needed, add it to code properly (restricted CIDR) through a PR.
- **Prevent:** read-only console, SCP guardrails, AWS Config rule (`restricted-ssh`), a `check` block asserting the rule count.

### 4. Apply failed halfway

- **Reproduce:** topic 33 with the instance profile just created (IAM eventual consistency), or topic 13's permission-denied file.
- **Symptom:** `Error: ...` after `N added`; some resources exist, others don't; one may be **tainted**.
- **Diagnose:** `terraform state list`, `terraform plan` (what's left?), the error's cause.
- **Fix:** fix the cause (wait for IAM, add permission, raise a quota) → plan → apply again. Terraform continues from state.
- **Prevent:** explicit `depends_on` for known eventual-consistency pairs, retries in providers, smaller changes.
- **Don't:** destroy everything "to start clean", or delete the state.

### 5. State file deleted

- **Reproduce** (free): topic 42 drill 2.
- **Symptom:** plan wants to **create everything**; apply would fail with `already exists` or create duplicates.
- **Diagnose:** is it really gone, or are you pointing at the wrong key/workspace/folder? `aws s3api list-object-versions`.
- **Fix:** remove the delete marker / restore the previous version → plan `No changes`. No versioning → freeze applies, rebuild with `import` blocks.
- **Prevent:** versioned bucket with `prevent_destroy`, limited delete permissions on state, backups of the state bucket (replication).

### 6. Provider upgrade changes a default

- **Reproduce:** pick a provider whose changelog mentions a changed default (or simulate by changing an optional argument you didn't set).
- **Symptom:** a bot PR "bump aws 6.70 → 6.73" shows `~ update in-place` on resources nobody touched.
- **Diagnose:** changelog for that range; which attribute changed and why.
- **Fix:** if the new default is fine → accept after review; if not → set the old value explicitly in code; if it's a regression → pin back and report.
- **Prevent:** provider bumps as separate PRs, dev first; read plans of bot PRs like any other.

### 7. Removing one server from a list rebuilds three

- **Reproduce** (free): topic 21 with `count = length(var.names)`; remove the first name.
- **Symptom:** `2 to add, 3 to destroy` for "remove one".
- **Diagnose:** `count` → index shift.
- **Fix:** convert to `for_each` with `moved` blocks (topic 40) in one PR (plan `0/0/0`), **then** remove the item (plan `1 to destroy`).
- **Prevent:** `for_each` for named things; TFLint/code review rule.

### 8. State with secrets committed to Git

- **Reproduce** (free, in a throwaway repo): commit a `terraform.tfstate` containing a `random_password`.
- **Symptom:** secret-scanning alert, or someone notices `terraform.tfstate` in the repo.
- **Diagnose:** which secrets are inside? (`jq` over the file.) Who had access? Is the repo public?
- **Fix (in this order):** 1) **rotate every secret** in the file (the leak already happened; history rewriting doesn't un-leak); 2) remove the file and move state to a backend (topic 35); 3) optionally purge history (`git filter-repo`) and force-push, coordinating with everyone; 4) report per your security process.
- **Prevent:** `.gitignore` (topic 05), pre-commit `detect-private-key` and secret scanners, remote backend from day one, ephemeral/write-only secrets (topic 51).

### 9. Two pipelines applied at the same time

- **Reproduce:** two workflows touching the same root without a shared concurrency group.
- **Symptom:** one fails with a lock error, or (if someone used `-lock=false`) resources missing from state / duplicates.
- **Diagnose:** pipeline logs, state versions in S3 (serials and times), CloudTrail.
- **Fix:** re-plan; import orphans (topic 42 scenario E); restore a state version if needed.
- **Prevent:** one concurrency group **per state** shared by all workflows; never `-lock=false` on apply; apply the saved plan (stale plans are rejected).

### 10. Destroy fails with `DependencyViolation`

- **Reproduce** (free): topic 32 network; create an ENI or security group by hand inside the VPC (`aws ec2 create-security-group --vpc-id ...`); `terraform destroy`.
- **Symptom:** `DependencyViolation: The vpc 'vpc-...' has dependencies and cannot be deleted.`
- **Diagnose:** `aws ec2 describe-network-interfaces --filters Name=vpc-id,Values=...`, `describe-security-groups`, load balancers, endpoints in that VPC.
- **Fix:** delete the unmanaged dependents (or destroy the configuration that owns them first), re-run destroy.
- **Prevent:** everything in Terraform; destroy consumers before producers (topic 38); tag-based inventory.

### 11. Dev code applied to the prod account

- **Reproduce** (safe): in a lab, set `allowed_account_ids` to a fake account and plan with your real profile.
- **Symptom (without protection):** the dev configuration's plan against prod shows creating/destroying unexpected things; worst case it was applied.
- **Diagnose:** `aws sts get-caller-identity`; which profile/role/backend key was used.
- **Fix:** stop; if applied, use the **prod** configuration to plan back to the intended state; restore state versions if states got mixed.
- **Prevent:** `allowed_account_ids` per root, separate accounts/backends/roles per environment, prod roles only for the prod pipeline (topic 57), no workspaces for environments (topic 48).

### 12. A plan that never settles

- **Reproduce** (free): a `local_file` with `content = "built at ${timestamp()}"`; plan twice.
- **Symptom:** every plan shows a change; PRs always "have changes"; drift checks always alert.
- **Diagnose:** which attribute changes each time? Impure function (`timestamp()`, `uuid()`), a value another system rewrites, or API normalisation (JSON order, case, defaults).
- **Fix:** `time_static`/`plantimestamp` only where intended; `ignore_changes` for externally-owned attributes; write values in the API's canonical form; `jsonencode` for policies.
- **Prevent:** review for impure functions; drift checks that alert on repeated identical diffs.

---

## Hands-on lab

1. Do the free ones: 1 (with the bucket module), 3, 5, 7, 8 (throwaway repo), 10, 11, 12.
2. For each, write a 6-line incident note: **What happened / Impact / Detection / Root cause / Fix / Prevention**.
3. Pair with a friend: one breaks something secretly, the other diagnoses using only `terraform` and AWS CLI output.

---

## Where you'll use it

On-call shifts, post-incident reviews, and senior interviews ("tell me about a Terraform incident you handled"). Your incident notes from this lab are good answers.

---

## When it breaks

If a scenario's fix doesn't work, go back to **STOP → SCOPE**: re-read the plan and state, check identity and backend, and don't stack a second risky action on the first. Escalate early when prod data is involved.

---

## Quick recap

- Sequence: **stop → scope → diagnose → fix (smallest safe, reviewed) → verify (plan clean) → prevent → write it down**.
- The fast moves (apply anyway, force-unlock without checking, delete state, `-target` habitually, `-lock=false`) cause the second incident.
- Prevention is mostly design: `for_each`, `moved`, protected stateful resources, policies, versioned backends, separate accounts and roles, concurrency per state, read-only consoles.

### Interview questions

1. **Q: A PR plan shows your production database will be replaced. What do you do?**
   A: Stop and find why: a `# forces replacement` argument or an address change. For an address change, add a `moved` block. For a force-new argument, revert it or plan a proper migration with snapshots. Protect databases with `prevent_destroy`, deletion protection and a CI policy that blocks deletes.
2. **Q: Someone committed a state file with secrets. What are your steps?**
   A: Rotate every secret it contains first, then remove it from the repo and move state to an encrypted remote backend, optionally purge Git history in coordination with the team, report it, and add prevention: `.gitignore`, secret scanning and ephemeral or write-only secrets.
3. **Q: Your apply failed halfway. What state are you in and what do you do?**
   A: Completed resources are in state, the failing one may be tainted, the rest are still planned. Fix the cause (permissions, eventual consistency, quotas) and run plan and apply again; Terraform resumes from state. Don't delete state or destroy everything.
