# 70 · Senior Scenario Interviews

> **Level:** ★★★ advanced
>
> **You will learn:** how to answer **open-ended design and incident questions** that senior DevOps/SRE/platform interviews use: a structure for any answer, twelve worked scenarios (design a Terraform platform for many teams, migrate ClickOps to IaC, recover lost state, secure the pipeline, handle a monorepo at scale, adopt OpenTofu or not, and more), and the trade-offs interviewers want to hear.
>
> **You need:** the whole course; topic 66 (incidents) and 69 (fact questions).
>
> **Part 11 · Interviews and reference** · Previous: [69 · Interview question bank](69-interview-question-bank.md) · Next: [71 · Cheat sheet](71-cheat-sheet.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Senior questions have no single right answer. Interviewers judge **how you think**: do you clarify, consider risk, make trade-offs, and land on something practical?

A structure that works for every scenario (**C-R-O-P-S**):

1. **Clarify**: scale, constraints, team skills, compliance, what exists today.
2. **Risks**: what could go badly wrong (data loss, outage, security, cost)?
3. **Options**: two or three approaches with trade-offs.
4. **Pick and plan**: a recommendation with phases (start small, prove it, expand).
5. **Safeguards and signals**: guardrails, rollback, how you'll know it works.

**Picture it.** An architect asked "design a house": a good one first asks how many people live there, the budget and the plot, then sketches options, then picks one and explains why.

**The one thing to know.** Always mention **state safety**, **the plan-review workflow**, **least-privilege credentials** and **blast radius**. These four separate senior answers from junior ones.

**Try it now:** answer scenario 1 aloud in five minutes using C-R-O-P-S, then compare with the model answer.

**You've got it if you can:** answer any scenario below in 5 minutes with clear trade-offs.

> 📖 **That's the core.** The scenarios follow.

---

## What it is / Why it exists / Analogy / How it works inside

A practice set. Each scenario: **the question → what they're testing → a model answer (in C-R-O-P-S shape) → follow-ups to expect**. Senior interviews test judgement under uncertainty, because that's the job. The model answers reuse ideas from across the course, so each points back to the topics behind it.

---

## Code / commands: the scenarios

### 1. "Design how 10 product teams should use Terraform on AWS."

**Testing:** structure, ownership, safety at scale.

- **Clarify:** number of accounts/environments, existing CI, compliance, team Terraform skill, HCP budget.
- **Risks:** shared god-state, teams breaking each other, inconsistent security, secrets in state, wrong-account applies.
- **Options:** (a) one central repo run by the platform team; (b) a platform-maintained **module catalogue + golden pipeline**, teams own their roots; (c) a platform (HCP Terraform/Spacelift) with workspaces per team/environment.
- **Pick:** (b), optionally on (c): a multi-account landing zone (account per team per environment), a state bucket and OIDC plan/apply roles per account, versioned modules with tests (topics 44–45, 55), a reusable pipeline with scans/policies/approvals (60), `CODEOWNERS`, and teams owning their roots (49).
- **Safeguards:** SCPs and policy as code (54), drift detection routed to owners (37), protected stateful resources (24), golden-path docs and templates.
- **Follow-ups:** "How do teams consume network outputs?" (SSM contract, 38). "How do you roll out a module fix to everyone?" (versioned release + bot PRs, 45, 61).

### 2. "We have 300 resources created by hand in the console. Bring them under Terraform."

**Testing:** brownfield adoption without outages.

- **Clarify:** criticality, how often things change, who clicks today, which accounts/regions.
- **Risks:** accidental replacement during import, configs that don't match reality, people still clicking.
- **Plan:** inventory by service and tag (AWS Config, `terraform query` where supported, 67); prioritise (network → shared → data → apps); for each slice, write config + `import` blocks (`-generate-config-out` as a draft, 39), iterate until **"N to import, 0 to change"**, apply via PR; then lock the door: read-only console, SCPs, drift detection (37).
- **Safeguards:** `prevent_destroy` on data, policy blocking deletes, small slices, a freeze per slice.
- **Follow-up:** "The plan after import wants to replace an RDS instance." Stop, find the force-new attribute, match config to reality (66 #1).

### 3. "An engineer deleted the prod state file. Go."

**Testing:** calm incident handling.

- **Immediate:** freeze pipelines/applies (an apply now would duplicate or conflict); confirm it's really gone (key, workspace, backend) (42).
- **Recover:** versioned bucket → remove the delete marker or copy the previous version → plan "No changes". No versioning → rebuild via `import` blocks, slice by slice.
- **After:** post-mortem; restrict `s3:DeleteObject` on state, enable versioning/replication, `prevent_destroy` on the bucket, rehearse restores.
- **Follow-up:** "How would you know which resources belonged to that state?" (tags like `stack`/`repo` from `default_tags`, CloudTrail, the code itself.)

### 4. "Make our Terraform pipeline secure."

**Testing:** supply chain and credentials.

- No long-lived keys: **OIDC** with trust conditions on repo/branch/environment (57).
- **Split roles**: read-only plan (PRs), write apply (protected environment only).
- **Reviewed saved plans** with approvals; branch protection; `CODEOWNERS` (56).
- Pin Terraform, providers (lock file with checksums), modules (versions/SHAs), CI actions (SHAs) (10, 45, 57).
- Static scans + policy on the plan (53, 54); secrets out of state (51); short-lived restricted plan artifacts; protect the state bucket (35).
- **Follow-up:** "A malicious PR adds a `local-exec` that curls the credentials." Plan runs with read-only, short-lived creds; no secrets in env; code review; consider blocking provisioners by policy, and running PR plans in isolated runners.

### 5. "Our monorepo has 150 roots and PR plans take 40 minutes."

**Testing:** scale and CI engineering.

- Measure (63): which roots, refresh vs walk time, API throttling.
- Plan only **affected roots** (path-based detection + module→root mapping) (60).
- Cache providers; parallelise per root (matrix) with per-state concurrency; split oversized states.
- Consider orchestration tooling (Terragrunt, Terramate, Stacks, a platform) for dependency-aware runs (49, 50).
- **Follow-up:** "A module used by all roots changes." Plan all, but batch and summarise; or version the module and roll out gradually per root.

### 6. "Should we move from Terraform to OpenTofu?"

**Testing:** balanced, fact-based decision.

- **Clarify:** licence policy, HCP Terraform usage, needed features, support contracts.
- **Facts:** same HCL/providers/workflow; OpenTofu adds client-side state encryption, provider `for_each`, `-exclude`, OCI registries; Terraform has HCP Terraform, Stacks, Sentinel/Terraform Policy, query and actions (04, 67).
- **Plan if yes:** pilot one non-critical root with a state copy; CI with both binaries; avoid tool-specific features during the transition; migrate root by root.
- **Answer shape:** "It depends on X and Y; here's how I'd decide and de-risk it."

### 7. "A Terraform apply caused an outage. What would you change?"

**Testing:** post-incident thinking, not blame.

- Find where the process allowed it: was the plan reviewed? Did it show the destroy/replacement? Was it the reviewed plan? Did policy exist? Was the resource protected?
- Fixes: `create_before_destroy` + name prefixes for zero-downtime replacements (24); `prevent_destroy`/deletion protection; a policy denying deletes of critical types (54); clearer plan summaries in PRs; smaller PRs; dev → staging soak time.
- **Follow-up:** "The plan was reviewed and approved." → The reviewer couldn't see the impact: improve plan readability (summaries, risk labels, Infracost), training, and automated checks for risky actions.

### 8. "How do you manage secrets for 50 RDS databases with Terraform?"

**Testing:** modern secret handling, and knowing what ends up in state.

- Prefer `manage_master_user_password = true` (RDS + Secrets Manager rotation; Terraform never sees passwords) (51).
- If Terraform must supply them: ephemeral generation + `password_wo` with versioned rotation.
- App access via IAM roles to Secrets Manager; no secrets in outputs or tfvars.
- Audit: grep state for secrets; restrict state access; KMS on the state bucket.

### 9. "Design state layout for a multi-region, multi-account product."

**Testing:** blast radius thinking and cross-state wiring.

- **Clarify:** how many accounts and regions, active-active or primary/DR, data-residency rules, which resources are global (IAM, Route 53, CloudFront), how often each layer changes.
- **Risks:** one state spanning regions (a single bad apply or regional API outage blocks everything), slow plans, tangled cross-state references, a region's state stored only in that region (lost with the region).
- **Options:** (a) one state per account with provider aliases for every region: simple, but a huge blast radius; (b) one state per **account × region × component**: small, isolated, more wiring; (c) (b) orchestrated by Terragrunt/Stacks to cut the repetition (49, 50).
- **Pick:** (b). State buckets per account (replicated to a second region, 35), keys like `<region>/<component>/terraform.tfstate`; components such as network, data, platform, apps; global resources in their own "global" states. Cross-state values via SSM in each account/region (38). Provider aliases or per-resource `region` (AWS v6) only where a component truly spans regions (30).
- **Safeguards:** promote region by region with a canary region first; `allowed_account_ids` in every provider; policy checks on region and account (54); per-state concurrency in CI (57).
- **Follow-ups:** "Where does the state for the state bucket live?" (a small bootstrap root that creates the bucket with local state, then migrates its own state into that bucket; or the landing-zone tooling creates it, 35). "A region goes down: can you still plan the others?" (yes, that's why states are per region; the state bucket is replicated).

### 10. "Kubernetes: what should Terraform own and what should Argo CD own?"

**Testing:** clear ownership boundaries between tools.

- **Clarify:** EKS or another platform, how many clusters, who deploys apps (platform team or product teams), existing GitOps use.
- **Risks:** two tools fighting over the same object (endless diffs or reverts), the cluster and in-cluster resources in one state (the provider can't configure itself before the cluster exists, 28), Terraform becoming the app deploy tool for daily releases.
- **Options:** (a) Terraform for everything including app manifests; (b) Terraform for infrastructure and bootstrap, Argo CD for apps; (c) Terraform for the cluster only, everything in-cluster via Argo CD (including add-ons).
- **Pick:** (b). Terraform owns VPC, EKS cluster, node groups, IAM (IRSA/Pod Identity) and bootstrap add-ons (ingress controller, cert-manager, external-dns, Argo CD itself), in a **separate state** from the cluster (28). Argo CD owns application manifests, versions, app config and anything that changes daily. The boundary: Terraform creates Argo CD's root Application; after that, Git drives apps.
- **Safeguards:** `ignore_changes` on fields controllers own (replica counts under an autoscaler); one owner per object, written down; drift alerts on both sides.
- **Follow-ups:** "Why not install add-ons with Argo CD too?" (fine, option (c), if the team is GitOps-mature; Terraform still needs the IAM roles they use). "How do you rebuild a cluster?" (Terraform recreates the cluster and Argo CD; Argo CD re-syncs apps from Git.)

### 11. "How do you keep 200 provider/module dependencies up to date safely?"

**Testing:** maintenance as a routine, not a crisis.

- **Clarify:** how far behind you are today, how many roots, whether modules are internal or public, test coverage.
- **Risks:** big-bang upgrades years later (many breaking changes at once), unreviewed bot merges changing infrastructure, supply-chain risk from unpinned third-party code.
- **Options:** (a) manual upgrades when something breaks; (b) bots with auto-merge; (c) bots with grouped PRs, full plans and human review of anything non-trivial.
- **Pick:** (c). Dependabot/Renovate for providers, modules and CI actions; grouped weekly PRs per ecosystem (61, 62). Every bot PR gets full plans; "No changes" merges quickly; real diffs need review. Majors treated as small projects with the upgrade guide; deprecation warnings fixed continuously. Multi-platform lock files; the pinned CLI upgraded together across roots.
- **Safeguards:** upgrade dev first and let it soak; module tests in CI (55); a dashboard of versions per root so laggards are visible.
- **Follow-ups:** "A provider minor version changes a default and the plan shows 40 updates." (Don't merge; read the changelog, pin back or set the argument explicitly, then upgrade deliberately.)

### 12. "A teammate wants to use `-target` in the pipeline to speed things up."

**Testing:** pushing back with reasons and a better fix.

- **Clarify:** what's slow (refresh, a huge state, API throttling, the runner), how slow, and which changes they want to target.
- **Risks:** partial applies, config and state drifting apart unnoticed, surprise diffs for the next person, dependencies skipped (63).
- **Options:** (a) allow `-target` in CI; (b) fix the cause: split the state, plan only affected roots, cache providers, raise parallelism carefully; (c) Terraform 1.17's `-minimal-refresh` for faster plans, with full plans still run on a schedule.
- **Pick:** (b), with (c) as a quick win where it fits. Explain kindly that `-target` treats the symptom and hides the real problem.
- **Safeguards:** `-target` only for documented emergencies, always followed by a full plan/apply; a CI check that rejects `-target` in normal pipelines.
- **Follow-ups:** "It's an incident and the full plan is blocked by an unrelated broken resource." (Targeted apply is acceptable here; record it and run a full plan as soon as the blocker is fixed.)

---

## Hands-on lab

1. Answer each scenario aloud in five minutes with C-R-O-P-S. Record yourself.
2. For three scenarios, draw the architecture on paper (accounts, states, pipelines, roles).
3. Turn your capstone (59–61) into a 3-minute story: problem, design, safeguards, one incident drill and what you learned.

---

## Where you'll use it

Senior/lead interviews, design reviews, and real planning meetings. These are the same conversations you'll have on the job.

---

## When it breaks

Common answer mistakes: jumping to tools before clarifying, ignoring state safety and credentials, no rollback/safeguard story, one-option answers, buzzwords without trade-offs. Fix with C-R-O-P-S and one concrete example from your labs.

---

## Quick recap

- **C-R-O-P-S**: Clarify → Risks → Options → Pick and plan → Safeguards and signals.
- Always cover **state safety, plan review, least-privilege credentials, blast radius**.
- Back answers with your capstone and drills.

### Interview questions

The twelve scenarios above. Practise until each takes five calm minutes.
