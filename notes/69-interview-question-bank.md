# 69 · Interview Question Bank

> **Level:** ★★ intermediate (reference)
>
> **You will learn:** nothing new. About **90 interview questions with model answers**, grouped by course part, from fundamentals to production, each linked to the topic with the full explanation. Answers are short on purpose: say them out loud, then add a real example from your labs or capstone.
>
> **You need:** the course. Use topic 70 for senior, open-ended scenarios.
>
> **Part 11 · Interviews and reference** · Previous: [68 · Production-readiness checklist](68-production-checklist.md) · Next: [70 · Senior scenario interviews](70-senior-scenario-interviews.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: how to use this bank

1. Cover the answer, say yours aloud, then compare.
2. Add **one concrete example** to every answer ("in my capstone, the web module...").
3. Questions marked ⭐ are asked in almost every Terraform interview.

**A strong answer** has: the definition in one line → why it matters → how you've used it → a pitfall you know about.

---

## Part 0–1 · Fundamentals

1. ⭐ **What is Infrastructure as Code?** Managing infrastructure through version-controlled definition files applied by a tool, instead of manual changes. Gives repeatability, review, history and fast recovery. ([01](01-what-is-iac.md))
2. **Declarative vs imperative?** Declarative describes the end state and the tool computes steps (Terraform); imperative lists steps (scripts). Declarative is idempotent and can detect drift. ([01](01-what-is-iac.md))
3. ⭐ **Terraform vs Ansible?** Terraform provisions infrastructure via APIs and keeps state; Ansible configures inside servers over SSH. Complementary. ([02](02-iac-tool-landscape.md))
4. **Terraform vs CloudFormation?** CloudFormation is AWS-only with AWS-managed state; Terraform is multi-provider with self-managed state and a plan preview. ([02](02-iac-tool-landscape.md))
5. ⭐ **What are Terraform core and providers?** Core parses config, manages state and the graph, computes plans; providers are plugins (gRPC) translating resource operations into platform APIs. ([03](03-what-is-terraform.md))
6. **What happened with Terraform's licence? What's OpenTofu?** Terraform moved from MPL to BSL in 2023 (restricts competing products, not normal use); OpenTofu is the MPL fork under the Linux Foundation. ([04](04-license-change-and-opentofu.md))
7. **What does `terraform init` do?** Configures the backend, downloads providers and modules, creates/verifies the lock file. ([07](07-first-terraform-project.md), [10](10-providers.md))
8. ⭐ **`plan` vs `apply`?** Plan computes and shows changes without making them; apply executes a plan (new or saved). ([09](09-core-workflow-commands.md))
9. **What do `+`, `~`, `-/+`, `+/-`, `<=` mean in a plan?** Create, update in place, replace (destroy first), replace (create first), read data source during apply. ([09](09-core-workflow-commands.md))
10. **Why use `plan -out` and `apply planfile`?** To apply exactly what was reviewed; stale plans are rejected. ([09](09-core-workflow-commands.md))
11. ⭐ **What is `.terraform.lock.hcl`? Commit it?** Records exact provider versions and checksums; yes, commit it; upgrade with `init -upgrade`. ([10](10-providers.md))
12. **What does `~> 5.40` allow?** `>= 5.40, < 6.0`. ([10](10-providers.md))
13. **How do you use two regions/accounts in one config?** Provider aliases (`alias`, `provider = aws.x`); in AWS v6 also the per-resource `region` argument. ([10](10-providers.md), [30](30-aws-provider-and-auth.md))
14. **Arguments vs attributes?** Arguments are set by you; attributes are readable values, including computed ones like IDs. ([11](11-resources-and-references.md))
15. **What is `(known after apply)`?** A value only available after the provider creates/updates the object; things derived from it are unknown too. ([11](11-resources-and-references.md))
16. ⭐ **What is state and why is it needed?** A record mapping config to real objects with attributes and dependencies; needed to update/delete, order destroys, detect drift and plan efficiently. ([12](12-state-basics.md))
17. ⭐ **Walk me through `terraform apply`.** Load config → lock and read state → start/configure providers → refresh → build graph → plan walk → approve → apply walk in parallel calling APIs → write state → outputs → unlock. ([13](13-inside-plan-and-apply.md))
18. **What happens if apply fails halfway?** Finished resources are in state, failed ones may be tainted; fix the cause and re-run. ([13](13-inside-plan-and-apply.md))

## Part 2 · The language

19. ⭐ **Variable precedence?** Default < `TF_VAR_` < `terraform.tfvars` < `*.auto.tfvars` < `-var`/`-var-file` (last wins). ([14](14-input-variables.md))
20. **Does `sensitive` protect secrets?** Only from CLI output; still in state and plans. ([14](14-input-variables.md), [51](51-secrets.md))
21. **Variables vs locals vs outputs?** Inputs from outside; internal named expressions; values exposed to callers/CLI. ([14](14-input-variables.md)–[16](16-locals.md))
22. **list vs set vs map vs object?** Ordered; unique unordered; keyed same-type; named fields of different types. ([17](17-types-and-expressions.md))
23. **How do you make an argument conditional?** A conditional returning `null` when unset. ([17](17-types-and-expressions.md))
24. **How do you test expressions?** `terraform console`. ([18](18-functions-and-console.md))
25. **Why `jsonencode` for policies?** Always valid JSON, readable diffs. ([18](18-functions-and-console.md))
26. ⭐ **Resource vs data source?** Managed vs read-only lookup. ([19](19-data-sources.md))
27. **Why is a data source sometimes read during apply?** Its inputs are unknown or it depends on pending changes (often `depends_on`). ([19](19-data-sources.md))
28. ⭐ **Implicit vs explicit dependencies?** References vs `depends_on` for hidden dependencies. ([20](20-dependencies-and-the-graph.md))
29. **How do you fix a dependency cycle?** Remove one reference or add a separate link resource (e.g. SG rule resources). ([20](20-dependencies-and-the-graph.md))
30. ⭐ **`count` vs `for_each`?** `count` for identical copies or 0/1 toggles; `for_each` for named items, stable keys, no index shift. ([21](21-count.md), [22](22-for-each.md))
31. **Why "Invalid for_each argument ... until apply"?** Keys depend on unknown values; use static keys. ([22](22-for-each.md))
32. **What's a `dynamic` block?** Generates repeated nested blocks from a collection. ([23](23-for-expressions-and-dynamic-blocks.md))
33. ⭐ **`lifecycle` options?** `create_before_destroy`, `prevent_destroy`, `ignore_changes`, `replace_triggered_by`, `destroy = false` (1.16+), conditions. ([24](24-lifecycle.md))
34. **Why does Terraform replace a resource?** Force-new argument change, unknown upstream value, taint, `-replace`, `replace_triggered_by`. ([24](24-lifecycle.md))
35. **Precondition vs postcondition vs check block?** Before (inputs) / after (self) errors; check blocks are warnings at the end. ([25](25-conditions-and-checks.md))

## Part 3–4 · Providers in practice

36. **How does the Docker provider talk to Docker?** Docker Engine API on the socket (or SSH/TCP). ([26](26-docker-provider.md))
37. **Would you deploy apps to Kubernetes with Terraform?** Cluster and add-ons yes; apps via GitOps/Helm. ([28](28-terraform-and-kubernetes.md))
38. **Why split cluster and Kubernetes resources into two configurations?** Providers are configured before resources exist. ([28](28-terraform-and-kubernetes.md))
39. ⭐ **How does the AWS provider find credentials?** Provider args → env vars → shared config/profile (SSO, roles) → web identity → container → instance role; optional `assume_role`. ([30](30-aws-provider-and-auth.md))
40. **How do you prevent applying to the wrong AWS account?** `allowed_account_ids`, separate roles/backends per environment. ([30](30-aws-provider-and-auth.md))
41. **Why are S3 settings separate resources?** Provider v4+ splits each bucket sub-configuration into its own resource. ([31](31-first-aws-resources.md))
42. **How do you avoid hard-coded AMIs?** `aws_ami` data source (and pin for prod rollouts). ([31](31-first-aws-resources.md), [61](61-capstone-3-day-2.md))
43. **What makes a subnet public?** A route to an internet gateway. ([32](32-aws-networking.md))
44. **Why separate security group rule resources?** Clean diffs, no cycles, no inline/separate conflicts. ([32](32-aws-networking.md))
45. **`user_data` changed but nothing happened: why?** Runs only at first boot; replace the instance. ([33](33-project-2-web-server-on-aws.md))
46. **How do you access instances without SSH?** SSM Session Manager with an instance role. ([33](33-project-2-web-server-on-aws.md))

## Part 5 · State

47. ⭐ **What does refresh do? Why is `terraform refresh` deprecated?** Re-reads real objects; the command wrote state without review → use `apply -refresh-only`. ([34](34-state-deep-dive.md))
48. **Resource in AWS and in code but not in state: what happens?** Plan creates → conflict; import it. ([34](34-state-deep-dive.md))
49. **What are `serial` and `lineage`?** Version counter and history ID; protect against stale/unrelated overwrites. ([34](34-state-deep-dive.md))
50. ⭐ **Why a remote backend?** Sharing, locking, encryption, access control, versioning. ([35](35-remote-backends.md))
51. **How do you configure different state per environment?** Partial backend config (`-backend-config`) or folders with their own backend blocks. ([35](35-remote-backends.md), [49](49-environments-and-layout.md))
52. **How is the state bucket created?** A bootstrap config with local state (or CLI), hardened. ([35](35-remote-backends.md))
53. ⭐ **What is state locking? How on S3 today?** One operation per state; `use_lockfile = true` (conditional writes); DynamoDB deprecated. ([36](36-state-locking.md))
54. **When is `force-unlock` safe?** When the holder is confirmed dead; back up, unlock, plan. ([36](36-state-locking.md))
55. ⭐ **What is drift? How do you detect and handle it?** Out-of-band changes; scheduled `plan -refresh-only -detailed-exitcode`; revert, adopt, accept, or ignore. ([37](37-drift.md))
56. **Why might a plan be green although someone opened a port?** Unmanaged separate rule objects are invisible. ([37](37-drift.md))
57. **How do configurations share values?** `terraform_remote_state`, lookups by tag, SSM parameters, platform features; remote state exposes the whole state. ([38](38-sharing-state.md))
58. ⭐ **How do you import existing resources?** `import` blocks (+ `-generate-config-out`), plan to "import, 0 to change", apply. ([39](39-import.md))
59. ⭐ **How do you rename a resource without destroying it?** `moved` block. ([40](40-moved-and-removed.md))
60. **How do you stop managing a resource without deleting it?** `removed` with `destroy = false`. ([40](40-moved-and-removed.md))
61. **`-replace` vs `taint`?** Same effect; `-replace` is in a reviewed plan, `taint` changed state immediately (deprecated). ([41](41-replace.md))
62. ⭐ **You lost the state file. What now?** Freeze; check location; restore S3 version/delete marker; else rebuild via imports. ([42](42-state-recovery.md))
63. **"Failed to persist state": what do you do?** Don't re-apply; fix access; `state push errored.tfstate`. ([42](42-state-recovery.md))
64. **How do you move resources between states?** `removed` (forget) in source + `import` in destination. ([42](42-state-recovery.md))

## Part 6–7 · Modules and environments

65. **Root vs child module?** Root has state/backend/providers; children are called and share the root's state. ([43](43-modules-basics.md))
66. ⭐ **How do you design a reusable module?** Focused, few validated inputs, secure defaults, clear outputs, no provider blocks, versioned, tested, documented. ([44](44-writing-modules.md))
67. **Why no provider blocks in modules?** Breaks `count`/`for_each`/removal and forces one config on callers. ([44](44-writing-modules.md))
68. **How do you version modules?** SemVer tags; Git `?ref=` or registry `version`; modules not in the lock file. ([45](45-module-sources-and-versioning.md))
69. **Evaluating a public module?** Maintainer, activity, versioning, tests, scope, security, licence; pin and read the plan. ([46](46-public-modules.md))
70. **What is dependency inversion in modules?** Pass dependencies (IDs) in instead of creating/looking up inside. ([47](47-module-patterns.md))
71. ⭐ **Workspaces for environments?** Generally no: shared backend/credentials, invisible selection. ([48](48-workspaces.md))
72. ⭐ **How do you manage dev/staging/prod?** Separate roots/states/accounts, same modules, different tfvars, promotion through the pipeline. ([49](49-environments-and-layout.md))
73. **What does Terragrunt add?** DRY backends/providers, dependencies between units, `run --all`. ([49](49-environments-and-layout.md))
74. **What does HCP Terraform add?** Remote runs, workspaces with vars/creds, VCS workflows, policies, drift detection, private registry, RBAC, audit. ([50](50-hcp-terraform-and-platforms.md))

## Part 8 · Security and quality

75. ⭐ **How do you handle secrets in Terraform?** Cloud-managed secrets; ephemeral resources/variables + write-only arguments; secret stores; protect state. ([51](51-secrets.md))
76. ⭐ **Why avoid provisioners? Alternatives?** Not in plan/state, not idempotent, taint on failure; use cloud-init, Packer, Ansible/SSM, actions. ([52](52-provisioners-and-alternatives.md))
77. **What replaced `null_resource`?** `terraform_data`. ([52](52-provisioners-and-alternatives.md))
78. **TFLint vs validate?** TFLint knows provider values and conventions; validate only schema. ([53](53-linting-and-scanning.md))
79. **How do you prevent insecure infrastructure?** Trivy/Checkov in pre-commit and CI, plan scanning, policies, secure modules. ([53](53-linting-and-scanning.md))
80. ⭐ **What is policy as code?** Rules (Rego/Sentinel) evaluated on plan JSON before apply. ([54](54-policy-as-code.md))
81. **How do you test Terraform?** Static checks, `terraform test` plan-level with mocks, a few apply-level tests, end-to-end checks. ([55](55-testing.md))

## Part 9–10 · CI/CD and operations

82. ⭐ **Describe your Terraform CI/CD workflow.** PR → checks + plan comment → review → merge → re-plan → approval → apply saved plan; one run per state. ([56](56-git-workflow.md)–[58](58-jenkins.md))
83. ⭐ **How does CI authenticate to AWS without keys?** OIDC → `AssumeRoleWithWebIdentity` with trust conditions on repo/branch/environment. ([57](57-github-actions.md))
84. **Why separate plan and apply roles?** PR code is untrusted; only approved prod jobs can write. ([57](57-github-actions.md))
85. **How do you roll back a Terraform change?** Revert commit and apply; can't restore deleted data. ([56](56-git-workflow.md))
86. **How do you upgrade a provider major version?** Latest minor first, fix deprecations, upgrade guide, plan all roots, dev → prod. ([62](62-upgrades.md))
87. **Plans take 15 minutes. What do you do?** Measure; split state; cache providers; tune parallelism; careful refresh options. ([63](63-terraform-at-scale.md))
88. **When is `-target` OK?** Emergencies/recovery, followed by a full plan. ([63](63-terraform-at-scale.md))
89. ⭐ **How do you debug a Terraform failure?** Read the error, identify the phase, use the phase's tool (`console`, `sts get-caller-identity`, `TF_LOG`, CloudTrail, `graph`). ([64](64-debugging-terraform.md))
90. **What's new in Terraform recently?** Import/check blocks, tests + mocks, removed, ephemeral + write-only, S3 lockfile, query + actions. ([67](67-whats-new.md))

---

## Rapid-fire (one-line answers)

| Question | Answer |
|---|---|
| File for exact provider versions? | `.terraform.lock.hcl` |
| Command to format code? | `terraform fmt` |
| Exit code for "changes present" with `-detailed-exitcode`? | `2` |
| Current workspace in code? | `terraform.workspace` |
| Read one output for scripts? | `terraform output -raw NAME` |
| Upgrade providers within constraints? | `terraform init -upgrade` |
| Switch backend without copying state? | `terraform init -reconfigure` |
| Copy state to a new backend? | `terraform init -migrate-state` |
| Remove a stale lock? | `terraform force-unlock ID` |
| Show plan as JSON? | `terraform show -json tfplan` |
| Debug logs to a file? | `TF_LOG=DEBUG TF_LOG_PATH=file` |
| Built-in resource replacing `null_resource`? | `terraform_data` |
| Block to adopt existing resources? | `import` |
| Block to rename addresses? | `moved` |
| Block to forget a resource? | `removed` (`destroy = false`) |
| S3 native locking argument? | `use_lockfile = true` |

---

## What it is / Why it exists / Analogy / How it works inside / Hands-on lab / Where you'll use it / When it breaks

This is a revision tool: interviews reward **fluent, example-backed** answers. Lab: record yourself answering the ⭐ questions in under 60 seconds each; re-read the linked topic for any answer you hesitate on. If an interviewer asks something not here, use the course's mental model: **config, state, reality; providers and APIs; plan before apply**.

## Quick recap

Know the ⭐ questions cold; back every answer with a lab/capstone example; finish with a pitfall you've seen.

### Interview questions

The whole page is the interview questions. For senior, open-ended ones, continue with [topic 70](70-senior-scenario-interviews.md).
