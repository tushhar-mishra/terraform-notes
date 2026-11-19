# 53 · Linting and Security Scanning

> **Level:** ★★ intermediate
>
> **You will learn:** the layers of automatic checks every Terraform repo should run before a human reviews it: **`terraform fmt -check`** and **`validate`** (topic 08), **TFLint** (provider-aware linting: invalid instance types, deprecated syntax, naming), **Trivy** (security misconfiguration scanning; ⚠️ tfsec has been folded into it), **Checkov** (security and compliance policies), scanning the **plan JSON** for what will really be created, **suppressions** with reasons, **terraform-docs**, and wiring it all into **pre-commit** hooks and CI.
>
> **You need:** topics 08, 09 (`show -json`), 44 (module READMEs), 46 (public modules).
>
> **Part 8 · Security and quality** · Previous: [52 · Provisioners, and what to use instead](52-provisioners-and-alternatives.md) · Next: [54 · Policy as code](54-policy-as-code.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Machines check the boring and the dangerous, so humans can review the important.

| Tool | Catches | Example finding |
|---|---|---|
| `terraform fmt -check` | Formatting | Unaligned `=` |
| `terraform validate` | Syntax, types, references | `Unsupported argument` |
| **TFLint** | Provider-specific mistakes and best practices | `"t2.micor" is an invalid value as instance_type` |
| **Trivy** (`trivy config`) | Security misconfigurations | S3 bucket without encryption; SG open to `0.0.0.0/0` on 22 |
| **Checkov** | Security/compliance policies (CIS, etc.) | `CKV_AWS_144: Ensure S3 bucket has cross-region replication` |
| **terraform-docs** | Missing/outdated module docs | README out of date |

```bash
terraform fmt -check -recursive && terraform validate
tflint --init && tflint --recursive
trivy config .
checkov -d .
```

**Picture it.** Before a building inspector visits (human reviewer), automatic sensors check the smoke detectors, the wiring and the fire exits. The inspector spends time on the design, not on counting smoke detectors.

**The one thing to know.** Run the checks **locally** (pre-commit) for fast feedback **and** in CI (so nothing skips them). Every suppression needs a **reason** in the comment.

**Try it now:** run `trivy config .` in your topic 33 project and read the first three findings.

**You've got it if you can:** say what TFLint catches that `validate` doesn't, and how to suppress a finding properly.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### TFLint

A linter for Terraform with **plugins (rulesets)** per provider:

- **Core rules** (`terraform` ruleset): unused variables/locals, missing `required_version`/`required_providers`, deprecated syntax, naming conventions, missing descriptions (configurable presets like `recommended` or `all`).
- **AWS ruleset** (`tflint-ruleset-aws`): knows real AWS values: invalid instance types, invalid AMI formats, deprecated instance families, invalid IAM policy JSON...
- Configured by `.tflint.hcl`.

`validate` only knows the provider's **schema** (types, required fields). TFLint knows **values**: `instance_type = "t2.micor"` passes `validate` (it's a string) but fails TFLint.

### Trivy

**Trivy** (by Aqua Security, open source) scans container images, filesystems and **IaC**. `trivy config <dir>` checks Terraform for **misconfigurations** against hundreds of built-in checks (IDs like `AVD-AWS-0086`). **tfsec**, the earlier Terraform scanner from the same company, was merged into Trivy; ⚠️ new projects should use Trivy. You already know Trivy from image scanning in your [GitHub Actions capstone](../../GitHub-Actions-Basics-To-Production/notes/44-capstone-02-trivy-filesystem-scan.md).

### Checkov

**Checkov** (by Prisma Cloud, open source, Python) has thousands of policies (`CKV_AWS_*`, `CKV2_AWS_*` for graph checks across resources) mapped to frameworks like CIS benchmarks. It can scan directories, **plan JSON**, and also Kubernetes, Dockerfiles, CloudFormation...

Trivy and Checkov overlap a lot; many teams pick one. Running both gives more coverage and more noise.

### Scanning code vs scanning the plan

| Scan | Sees | Misses |
|---|---|---|
| **Code** (`trivy config .`, `checkov -d .`) | Your `.tf` as written, fast, no credentials needed | Values only known at plan time; what remote modules actually create (some tools download modules) |
| **Plan JSON** (`terraform show -json tfplan > plan.json`, then `checkov -f plan.json`) | The **real** resources and values about to be created | Needs a plan (credentials, backend) |

Mature pipelines scan code on every commit and the plan before apply.

### Suppressions

Every tool lets you ignore a finding **for a reason**:

```hcl
resource "aws_s3_bucket" "logs" {
  #trivy:ignore:AVD-AWS-0089 This IS the access-log bucket; logging it to itself would loop.
  #checkov:skip=CKV_AWS_18:This IS the access-log bucket.
  bucket = "acme-access-logs"
}

# tflint-ignore: aws_instance_previous_type
resource "aws_instance" "legacy" { instance_type = "t2.micro" }
```

Rules: suppress **inline** (next to the resource), always with a **reason**, review suppressions in PRs, and prefer **fixing** the finding.

### pre-commit

**pre-commit** is a framework that runs checks on `git commit`. The community collection **`antonbabenko/pre-commit-terraform`** wraps all the tools above.

---

## Why it exists

Reviewers are bad at spotting "this bucket isn't encrypted" in a 400-line diff, and great at judging design. Most real incidents from IaC are **misconfigurations** (open security groups, public buckets, unencrypted data, overly broad IAM). Automated scanners catch the common ones every time, in seconds, before anything reaches the cloud ("shift left").

---

## Analogy

**Building inspections.**

- `fmt` = the drawing is neat and uses the standard symbols.
- `validate` = every room on the drawing has walls and a door.
- **TFLint** = the window sizes on the drawing actually exist in the supplier's catalogue.
- **Trivy/Checkov** = the fire-safety checklist: smoke detectors, exits, no locked fire doors.
- **Suppression with a reason** = an inspector's signed waiver: "no sprinkler in the server room: gas suppression installed instead".
- **Human review** = the architect's judgement about whether the building makes sense.

---

## How it works inside

```
git commit
  └─ pre-commit hooks (local, seconds): fmt, validate, tflint, trivy/checkov, terraform-docs
git push → PR
  └─ CI (same checks, cannot be skipped)
       fmt -check → init -backend=false → validate → tflint → trivy config / checkov -d
       → plan → show -json → scan plan JSON (checkov -f) → policy checks (topic 54)
       → results as PR annotations / comments; failures block merge
```

- Static scanners parse HCL themselves (or use Terraform's JSON plan); they don't need cloud credentials for code scans.
- TFLint plugins are separate binaries downloaded by `tflint --init`, configured in `.tflint.hcl`.

---

## Code / commands

### TFLint

Install (Linux): see `github.com/terraform-linters/tflint` (an install script or a single binary). Then in your repo:

```hcl
# .tflint.hcl
config {
  call_module_type = "local"          # also lint local modules
}

plugin "terraform" {
  enabled = true
  preset  = "recommended"
}

plugin "aws" {
  enabled = true
  version = "0.44.0"                   # use the latest release from github.com/terraform-linters/tflint-ruleset-aws
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "terraform_naming_convention" {
  enabled = true                       # snake_case names
}

rule "terraform_documented_variables" {
  enabled = true
}
```

Put a mistake in topic 33's `compute.tf` (`instance_type = "t3.micor"`):

```bash
tflint --init
tflint --recursive
```

```
2 issue(s) found:

Error: "t3.micor" is an invalid value as instance_type (aws_instance_invalid_type)

  on compute.tf line 12:
  12:   instance_type          = "t3.micor"

Warning: variable "page_message" should have a description (terraform_documented_variables)

  on variables.tf line 21:
  21: variable "page_message" {

Reference: https://github.com/terraform-linters/tflint-ruleset-terraform/blob/v0.x/docs/rules/terraform_documented_variables.md
```

`terraform validate` would have said `Success!` for the first one.

### Trivy

```bash
# install: aquasecurity.github.io/trivy (apt repo or binary)
cd ~/tf-lab/33-web
trivy config .
```

```
33-web (terraform)

Tests: 52 (SUCCESSES: 44, FAILURES: 8)
Failures: 8 (UNKNOWN: 0, LOW: 2, MEDIUM: 2, HIGH: 3, CRITICAL: 1)

AVD-AWS-0104 (CRITICAL): Security group rule allows unrestricted egress to any IP address.
═══════════════════════════════════════════════════════════════════════════════
Opening up ports to connect out to the public internet is generally to be avoided...
───────────────────────────────────────────────────────────────────────────────
 security.tf:22
───────────────────────────────────────────────────────────────────────────────
  17   resource "aws_vpc_security_group_egress_rule" "all" {
  ..
  22 [   cidr_ipv4         = "0.0.0.0/0"
───────────────────────────────────────────────────────────────────────────────

AVD-AWS-0107 (HIGH): Security group rule allows unrestricted ingress from any IP address.
 security.tf:12  (port 80 from 0.0.0.0/0)
...
AVD-AWS-0178 (MEDIUM): VPC does not have VPC Flow Logs enabled.
 network.tf:10
```

(IDs and counts vary by Trivy version.) How to read it:

- Each finding: **ID**, **severity**, what's wrong, **file:line**, the code.
- Some findings are **real risks to fix** (flow logs off), some are **accepted by design** (a public web server must allow port 80 from anywhere). Suppress those with a reason:

```hcl
resource "aws_vpc_security_group_ingress_rule" "http" {
  #trivy:ignore:AVD-AWS-0107 Public web server: HTTP from the internet is the point.
  ...
}
```

Fail CI only on serious findings: `trivy config --severity HIGH,CRITICAL --exit-code 1 .`

### Checkov

```bash
pipx install checkov          # or: pip install checkov
checkov -d . --compact --quiet
```

```
Check: CKV_AWS_126: "Ensure that detailed monitoring is enabled for EC2 instances"
	FAILED for resource: aws_instance.web
	File: /compute.tf:10-40
Check: CKV_AWS_135: "Ensure that EC2 is EBS optimized"
	FAILED for resource: aws_instance.web
	File: /compute.tf:10-40
Check: CKV2_AWS_11: "Ensure VPC flow logging is enabled in all VPCs"
	FAILED for resource: aws_vpc.main
...
Passed checks: 31, Failed checks: 9, Skipped checks: 0
```

Scan the **plan**:

```bash
terraform plan -out=tfplan && terraform show -json tfplan > plan.json
checkov -f plan.json --framework terraform_plan
```

Run only chosen checks or skip some globally: `--check CKV_AWS_20,CKV_AWS_57`, `--skip-check CKV_AWS_126`; or a `.checkov.yaml` config file.

### pre-commit

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.99.0                       # pin to the latest release tag
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
        args: ["--args=--config=__GIT_WORKING_DIR__/.tflint.hcl"]
      - id: terraform_trivy
        args: ["--args=--severity=HIGH,CRITICAL"]
      - id: terraform_docs
        args: ["--args=--config=.terraform-docs.yml"]
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: detect-private-key          # stop committing keys
      - id: end-of-file-fixer
```

```bash
pipx install pre-commit
pre-commit install            # adds a git hook
pre-commit run --all-files    # run everything now
```

```
Terraform fmt............................................................Passed
Terraform validate.......................................................Passed
Terraform validate with tflint...........................................Failed
- hook id: terraform_tflint
...
```

The commit is blocked until it passes (or you use `git commit --no-verify`, which CI then catches).

---

## Hands-on lab

1. Install TFLint with the AWS ruleset; lint topic 33's project; break an instance type and an unused variable on purpose.
2. Run `trivy config .` on topics 32 and 33. Fix one real finding (e.g. add VPC flow logs to a log group, or restrict egress to 443 and 80), and suppress one by-design finding with a reason.
3. Run Checkov on the code and on `plan.json`; compare the results.
4. Scan a public module you downloaded in topic 46: `trivy config .terraform/modules/vpc`.
5. Set up pre-commit in your `~/tf-lab` Git repo and make a failing commit.
6. Decide a CI policy: which severities fail the build?

---

## Where you'll use it

- **Every PR** in a mature Terraform repo (topics 57–58 wire these into pipelines).
- **Security/compliance programmes**: Checkov/Trivy reports mapped to CIS benchmarks for audits.
- **Module quality gates**: shared modules must pass scans with zero unexplained suppressions.
- **Interviews**: "How do you prevent insecure infrastructure from being deployed?"

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `Failed to initialize plugins; Plugin "aws" not found` | `tflint --init` not run. | `tflint --init` (cache it in CI). |
| TFLint finds nothing in modules | `call_module_type` not set. | `call_module_type = "local"` (or `"all"`). |
| Hundreds of findings on day one | Legacy code; every default check on. | Baseline: fail on HIGH/CRITICAL only, fix gradually, suppress with reasons. |
| Same issue reported by Trivy and Checkov | Overlap. | Pick one as the gate; use the other occasionally. |
| Suppression ignored | Wrong comment syntax/ID, or comment not directly above the resource/attribute. | Check each tool's exact syntax. |
| Pre-commit hooks slow | Running full scans on every commit. | Keep fast hooks locally; full scans in CI. |
| `--no-verify` used to skip hooks | Local hooks are optional. | CI runs the same checks and blocks merge. |

---

## Quick recap

- Layers: `fmt` → `validate` → **TFLint** (values, conventions) → **Trivy/Checkov** (security) → plan-JSON scans → human review.
- ⚠️ tfsec → use **Trivy** (`trivy config`).
- Suppress only inline, with a reason; prefer fixing.
- Run locally with **pre-commit**, enforce in **CI**.

### Interview questions

1. **Q: What does TFLint catch that `terraform validate` doesn't?**
   A: Provider-specific invalid values (like a non-existent instance type), deprecated syntax, unused declarations and naming/documentation conventions. `validate` only checks syntax, types and references against the schema.
2. **Q: How do you prevent insecure Terraform from being deployed?**
   A: Static scanning (Trivy or Checkov) in pre-commit and CI on code and on the plan JSON, failing on high-severity findings; policy-as-code checks before apply; reviewed suppressions with reasons; and secure-by-default modules.
3. **Q: Why scan the plan JSON as well as the code?**
   A: The plan contains the actual resources and resolved values, including those from variables, functions and remote modules, which static code scanning can't always see.
