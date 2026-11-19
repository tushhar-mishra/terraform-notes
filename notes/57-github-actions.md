# 57 · Terraform in GitHub Actions

> **Level:** ★★★ advanced
>
> **You will learn:** a production-grade Terraform pipeline in **GitHub Actions**: **OIDC** (OpenID Connect) authentication to AWS with **no stored keys** (and the Terraform code that creates the OIDC provider and two IAM roles: read-only **plan** and write **apply**), a **PR workflow** (fmt, validate, TFLint, Trivy, test, plan, plan posted as a PR comment), a **main-branch workflow** (plan → upload the plan → **environment approval** → apply the same plan), **concurrency** per state, least-privilege `permissions`, and pinning actions.
>
> **You need:** topics 29–30 (IAM roles, credential chain), 35–36 (backend, locking), 53–56. GitHub Actions background: [topics 17](../../GitHub-Actions-Basics-To-Production/notes/17-secrets-and-variables.md), [18](../../GitHub-Actions-Basics-To-Production/notes/18-github-token-and-permissions.md), [24](../../GitHub-Actions-Basics-To-Production/notes/24-artifacts.md), [28](../../GitHub-Actions-Basics-To-Production/notes/28-environments.md), [29](../../GitHub-Actions-Basics-To-Production/notes/29-concurrency.md), [36](../../GitHub-Actions-Basics-To-Production/notes/36-oidc-cloud-authentication.md), [37](../../GitHub-Actions-Basics-To-Production/notes/37-security-hardening.md).
>
> **Part 9 · CI/CD** · Previous: [56 · The Git workflow for Terraform](56-git-workflow.md) · Next: [58 · Terraform in Jenkins](58-jenkins.md) · [Back to INDEX](../INDEX.md)
>
> 💰 **Cost:** GitHub Actions minutes (free tier for public/private within limits); AWS: IAM is free; the lab deploys topic 32's free network.

---

## ⚡ Core: the 10-minute version

**The idea.** Two workflows, two AWS roles, zero stored keys:

```
PR  ──▶ terraform-pr.yml   ──▶ assume ROLE "tf-plan"  (read-only + state lock) ──▶ checks + plan ──▶ PR comment
main ──▶ terraform-main.yml ──▶ assume ROLE "tf-plan"  ──▶ plan -out=tfplan ──▶ artifact
                              └▶ environment "production" (required reviewer approves)
                                 ──▶ assume ROLE "tf-apply" (write) ──▶ apply tfplan
```

The key OIDC step:

```yaml
permissions:
  id-token: write          # allow the job to request an OIDC token from GitHub
  contents: read
steps:
  - uses: aws-actions/configure-aws-credentials@v5
    with:
      role-to-assume: arn:aws:iam::123456789012:role/tf-plan
      aws-region: ap-south-1
```

**Picture it.** Instead of giving the pipeline a permanent key to the building (access keys in secrets), the building's reception (AWS STS) checks the pipeline's **ID card** issued by GitHub (OIDC token: "I'm workflow X in repo Y on branch Z") and hands out a **visitor badge** valid for one hour (temporary credentials).

**The one thing to know.** Separate **plan** (read-only, usable by PRs) and **apply** (write, usable **only** by the protected `production` environment) roles. Then even a malicious PR can't change infrastructure.

**Try it now:** run the bootstrap Terraform below to create the OIDC provider and roles, add the two workflows to your lab repo, open a PR.

**You've got it if you can:** explain how GitHub OIDC replaces stored AWS keys, and how the trust policy limits which workflows can assume the apply role.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### OIDC from GitHub Actions to AWS (recap of GitHub Actions topic 36, applied to Terraform)

1. You register GitHub's OIDC issuer (`https://token.actions.githubusercontent.com`) in your AWS account as an **identity provider**.
2. You create IAM roles whose **trust policy** says: "tokens from that issuer, for audience `sts.amazonaws.com`, whose **`sub`** (subject) claim matches `repo:ORG/REPO:...`, may assume me".
3. In a workflow with `permissions: id-token: write`, the `configure-aws-credentials` action requests a token from GitHub and calls `sts:AssumeRoleWithWebIdentity`.
4. AWS checks the token signature and the conditions, and returns **temporary credentials** as environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`), which Terraform's provider and backend pick up (topic 30's credential chain, step 2).

The `sub` claim tells you **what** is running:

| Trigger | `sub` |
|---|---|
| Pull request | `repo:acme/infra:pull_request` |
| Push to a branch | `repo:acme/infra:ref:refs/heads/main` |
| Job with `environment: production` | `repo:acme/infra:environment:production` |

### Two roles

| Role | Trusted `sub` | Permissions |
|---|---|---|
| `tf-plan` | `repo:ORG/REPO:pull_request`, `repo:ORG/REPO:ref:refs/heads/main` | `ReadOnlyAccess` + read/write on the state **lock** objects + read on state |
| `tf-apply` | **only** `repo:ORG/REPO:environment:production` | What the configuration needs to manage (admin in a sandbox; scoped in real life) + state read/write |

The `production` **environment** in GitHub has **required reviewers** and a **deployment branch rule** (`main` only). So assuming `tf-apply` requires: running from `main`, in the `production` environment, after a human approved. A PR can't get it.

### Workflow design

| Concern | How |
|---|---|
| Least privilege for `GITHUB_TOKEN` | `permissions:` at workflow/job level ([GHA topic 18](../../GitHub-Actions-Basics-To-Production/notes/18-github-token-and-permissions.md)) |
| One run per state | `concurrency: { group: tf-${{ env.ROOT }}, cancel-in-progress: false }` (never cancel an apply) |
| Reviewed plan = applied plan | Plan job uploads `tfplan` as an **artifact**; apply job downloads it and runs `terraform apply tfplan` |
| Same Terraform version everywhere | `hashicorp/setup-terraform` with an exact `terraform_version` |
| Readable logs, no prompts | `TF_IN_AUTOMATION=1`, `TF_INPUT=0`, `-no-color` for comments |
| Supply chain | Pin third-party actions to a **commit SHA** ([GHA topic 37](../../GitHub-Actions-Basics-To-Production/notes/37-security-hardening.md)); versions shown below are tags for readability |
| Fork PRs | No OIDC token / secrets for forks by default: plans on fork PRs can't access AWS (intended) |

---

## Why it exists

Storing long-lived AWS keys in CI secrets is the classic way infrastructure accounts get compromised: keys leak through logs, forks or third-party actions, and they work forever. OIDC gives short-lived, narrowly scoped credentials tied to **which repo, branch and environment** is running. Combined with plan/apply separation and environment approvals, the pipeline becomes the only, auditable path to production.

---

## Analogy

**A building with reception and visitor badges.**

- **Old way**: give the delivery company a **permanent master key** (access keys). If it's copied, anyone can get in, forever.
- **OIDC**: the delivery driver shows a **company ID card** (GitHub OIDC token) at reception. Reception (AWS STS) checks it was issued by the right company, for the right job (`sub`), and issues a **1-hour visitor badge** (temporary credentials).
- **Two badges**: "look around" (plan role) for anyone with a valid ID; "maintenance access" (apply role) only for drivers on the **approved production job** with a manager's signature (environment reviewers).

---

## How it works inside

```
PR opened
 └─ terraform-pr.yml
     job plan (permissions: id-token write, contents read, pull-requests write)
       1. checkout
       2. configure-aws-credentials → GitHub OIDC token (sub = repo:acme/infra:pull_request)
          → sts:AssumeRoleWithWebIdentity(tf-plan) → temp creds in env
       3. setup-terraform 1.17.2
       4. fmt -check · init · validate · tflint · trivy · terraform test
       5. plan -out=tfplan -lock-timeout=5m → show -no-color → comment on PR
push to main (after merge)
 └─ terraform-main.yml
     job plan  → (tf-plan role) → plan -out=tfplan → upload artifact (tfplan + .terraform.lock.hcl)
     job apply (needs: plan, environment: production)
       ⏸ waits for a required reviewer to approve in the GitHub UI (they read the plan in job 1's log/summary)
       → OIDC (sub = repo:acme/infra:environment:production) → tf-apply role
       → download artifact → init → apply tfplan   (stale → rejected, re-run)
```

---

## Code / commands

### Step 1: bootstrap the OIDC provider and roles (Terraform, run once with admin credentials)

`~/tf-lab/57-gha-bootstrap/main.tf` (with `aws ~> 6.0`, your provider block):

```hcl
variable "github_repo" {
  type        = string
  description = "owner/repo allowed to use these roles"
  default     = "your-github-user/tf-lab"
}

variable "state_bucket" {
  type    = string
  default = "tf-course-state-123456789012"
}

resource "aws_iam_openid_connect_provider" "github" {
  url            = "https://token.actions.githubusercontent.com"
  client_id_list = ["sts.amazonaws.com"]
  # thumbprint_list is no longer needed for GitHub's issuer (AWS validates it with trusted CAs)
}

# ── Trust policies ───────────────────────────────────────────────────
data "aws_iam_policy_document" "plan_trust" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }
    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }
    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      values = [
        "repo:${var.github_repo}:pull_request",
        "repo:${var.github_repo}:ref:refs/heads/main",
      ]
    }
  }
}

data "aws_iam_policy_document" "apply_trust" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }
    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }
    condition {
      test     = "StringEquals"                                  # exact match: environment only
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:${var.github_repo}:environment:production"]
    }
  }
}

# ── State access (both roles need the lock; plan only reads state) ──
data "aws_iam_policy_document" "state_rw" {
  statement {
    actions   = ["s3:ListBucket"]
    resources = ["arn:aws:s3:::${var.state_bucket}"]
  }
  statement {
    actions   = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"]
    resources = ["arn:aws:s3:::${var.state_bucket}/*"]
  }
}

resource "aws_iam_policy" "state_rw" {
  name   = "tf-state-access"
  policy = data.aws_iam_policy_document.state_rw.json
}

# ── Roles ────────────────────────────────────────────────────────────
resource "aws_iam_role" "plan" {
  name                 = "tf-plan"
  assume_role_policy   = data.aws_iam_policy_document.plan_trust.json
  max_session_duration = 3600
}

resource "aws_iam_role_policy_attachment" "plan_readonly" {
  role       = aws_iam_role.plan.name
  policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
}

resource "aws_iam_role_policy_attachment" "plan_state" {
  role       = aws_iam_role.plan.name
  policy_arn = aws_iam_policy.state_rw.arn
}

resource "aws_iam_role" "apply" {
  name                 = "tf-apply"
  assume_role_policy   = data.aws_iam_policy_document.apply_trust.json
  max_session_duration = 3600
}

resource "aws_iam_role_policy_attachment" "apply_admin" {
  role       = aws_iam_role.apply.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"   # ⚠️ sandbox only; scope it in real accounts
}

resource "aws_iam_role_policy_attachment" "apply_state" {
  role       = aws_iam_role.apply.name
  policy_arn = aws_iam_policy.state_rw.arn
}

output "plan_role_arn"  { value = aws_iam_role.plan.arn }
output "apply_role_arn" { value = aws_iam_role.apply.arn }
```

- The plan role's `state_rw` lets it write the **lock** object (`plan` takes the lock, topic 36) and, technically, state. Tighter: allow `PutObject`/`DeleteObject` only on `*.tflock` for the plan role.
- `ReadOnlyAccess` includes reading many data planes (e.g. S3 objects). Some teams use `ViewOnlyAccess` plus specific reads; balance with what `plan` needs to refresh.

```bash
cd ~/tf-lab/57-gha-bootstrap && terraform init && terraform apply
```

### Step 2: GitHub settings

1. Repo **Settings → Secrets and variables → Actions → Variables**: `AWS_PLAN_ROLE_ARN`, `AWS_APPLY_ROLE_ARN`, `AWS_REGION=ap-south-1`. (ARNs aren't secrets.)
2. **Settings → Environments → New environment `production`**: required reviewers (you), deployment branches: `main` only.
3. Branch protection on `main`: require the `terraform-pr / plan` check (topic 56).

The Terraform root used here: `envs/dev/network` = topic 32's network with the S3 backend (`key = "gha/dev/network/terraform.tfstate"`).

### Step 3: the PR workflow

`.github/workflows/terraform-pr.yml`:

```yaml
name: terraform-pr
on:
  pull_request:
    paths: ["envs/**", "modules/**", ".github/workflows/terraform-*.yml"]

permissions:
  contents: read
  id-token: write
  pull-requests: write

env:
  TF_IN_AUTOMATION: "1"
  TF_INPUT: "0"
  ROOT: envs/dev/network

concurrency:
  group: tf-${{ github.workflow }}-${{ github.event.pull_request.number }}
  cancel-in-progress: true          # safe for PR plans: newer pushes supersede older plans

jobs:
  plan:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ${{ env.ROOT }}
    steps:
      - uses: actions/checkout@v5

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.17.2
          terraform_wrapper: false

      - uses: aws-actions/configure-aws-credentials@v5
        with:
          role-to-assume: ${{ vars.AWS_PLAN_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: fmt
        run: terraform fmt -check -recursive "$GITHUB_WORKSPACE"

      - name: init
        run: terraform init -lockfile=readonly

      - name: validate
        run: terraform validate

      - uses: terraform-linters/setup-tflint@v4
      - name: tflint
        run: tflint --init && tflint --recursive

      - name: trivy
        uses: aquasecurity/trivy-action@0.33.1
        with:
          scan-type: config
          scan-ref: ${{ env.ROOT }}
          severity: HIGH,CRITICAL
          exit-code: "1"

      - name: plan
        id: plan
        run: |
          set +e
          terraform plan -out=tfplan -lock-timeout=5m -no-color -detailed-exitcode > plan.txt 2>&1
          code=$?
          echo "exitcode=$code" >> "$GITHUB_OUTPUT"
          cat plan.txt
          [ "$code" -eq 1 ] && exit 1 || exit 0

      - name: comment plan on PR
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          {
            echo "### Terraform plan: \`${ROOT}\` (exit code ${{ steps.plan.outputs.exitcode }})"
            echo '<details><summary>Show plan</summary>'
            echo
            echo '```'
            tail -c 60000 plan.txt          # PR comments have a size limit
            echo '```'
            echo '</details>'
          } > comment.md
          gh pr comment ${{ github.event.pull_request.number }} --body-file comment.md
```

- `-lockfile=readonly`: fail if `.terraform.lock.hcl` would change (provider changes must come from a deliberate commit).
- `-detailed-exitcode`: `0` no changes, `2` changes, `1` error (topic 09). The step fails only on `1`.
- Shell is bash with `-e` by default in Actions; `set +e` lets us capture the exit code ([GHA topic 04](../../GitHub-Actions-Basics-To-Production/notes/04-linux-shell-recap-for-runners.md)).

### Step 4: the main workflow (plan → approval → apply)

`.github/workflows/terraform-main.yml`:

```yaml
name: terraform-main
on:
  push:
    branches: [main]
    paths: ["envs/**", "modules/**"]
  workflow_dispatch:

permissions:
  contents: read
  id-token: write

env:
  TF_IN_AUTOMATION: "1"
  TF_INPUT: "0"
  ROOT: envs/dev/network

concurrency:
  group: tf-state-envs-dev-network      # one run per STATE, across all workflows that touch it
  cancel-in-progress: false             # never cancel an apply mid-way

jobs:
  plan:
    runs-on: ubuntu-latest
    outputs:
      changes: ${{ steps.plan.outputs.changes }}
    defaults:
      run:
        working-directory: ${{ env.ROOT }}
    steps:
      - uses: actions/checkout@v5
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.17.2
          terraform_wrapper: false
      - uses: aws-actions/configure-aws-credentials@v5
        with:
          role-to-assume: ${{ vars.AWS_PLAN_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}
      - run: terraform init -lockfile=readonly
      - id: plan
        run: |
          set +e
          terraform plan -out=tfplan -lock-timeout=5m -detailed-exitcode -no-color | tee plan.txt
          code=${PIPESTATUS[0]}
          [ "$code" -eq 1 ] && exit 1
          echo "changes=$([ "$code" -eq 2 ] && echo true || echo false)" >> "$GITHUB_OUTPUT"
          { echo '```'; cat plan.txt; echo '```'; } >> "$GITHUB_STEP_SUMMARY"
      - uses: actions/upload-artifact@v4
        if: steps.plan.outputs.changes == 'true'
        with:
          name: tfplan
          path: ${{ env.ROOT }}/tfplan
          retention-days: 3                # plans contain secrets: keep briefly

  apply:
    needs: plan
    if: needs.plan.outputs.changes == 'true'
    runs-on: ubuntu-latest
    environment: production              # ⏸ required reviewers approve here
    defaults:
      run:
        working-directory: ${{ env.ROOT }}
    steps:
      - uses: actions/checkout@v5
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.17.2
          terraform_wrapper: false
      - uses: aws-actions/configure-aws-credentials@v5
        with:
          role-to-assume: ${{ vars.AWS_APPLY_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}
      - uses: actions/download-artifact@v4
        with:
          name: tfplan
          path: ${{ env.ROOT }}
      - run: terraform init -lockfile=readonly
      - run: terraform apply -lock-timeout=5m tfplan
```

- The **plan is shown in the job summary** so the approver reads exactly what will be applied.
- The apply job runs `terraform apply tfplan`: no re-plan. If the state changed since (another run), Terraform says **"Saved plan is stale"** and you re-run the workflow.
- `if: changes == 'true'`: no approval request when there's nothing to do.
- A real repo has one plan/apply pair **per root** (a matrix over changed roots, or one workflow per environment).

### What it looks like

- PR: a check `terraform-pr / plan` and a comment with the collapsible plan.
- After merge: `terraform-main` → `plan` succeeds → `apply` shows **"Waiting for review: production needs approval"** → you click **Review deployments → Approve** → apply runs.

Apply log excerpt:

```
Run aws-actions/configure-aws-credentials@v5
  Assuming role with OIDC
  Authenticated as assumedRoleId AROA...:GitHubActions
Run terraform apply -lock-timeout=5m tfplan
aws_vpc_security_group_ingress_rule.web_http: Modifying... [id=sgr-0a1b...]
aws_vpc_security_group_ingress_rule.web_http: Modifications complete after 1s [id=sgr-0a1b...]

Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
```

---

## Hands-on lab

1. Apply the bootstrap (OIDC provider + two roles) in your sandbox account.
2. Push your `~/tf-lab` repo to GitHub with `envs/dev/network` (topic 32 code + S3 backend) and both workflows. Set the variables and the `production` environment.
3. Open a PR changing a tag. Check the plan comment. Make `fmt` fail on purpose and see the check fail.
4. Merge. Approve the deployment. Confirm the change in AWS.
5. Security test: in a PR, add a step that tries `role-to-assume: ${{ vars.AWS_APPLY_ROLE_ARN }}`. It fails with `Not authorized to perform sts:AssumeRoleWithWebIdentity`: the trust policy only allows the `production` environment.
6. Trigger two runs quickly and watch concurrency queue the second.
7. Clean up: `terraform destroy` the network (locally with admin credentials, or via a destroy workflow), and destroy the bootstrap roles when you finish the course.

---

## Where you'll use it

- **Most GitHub-hosted companies**: this exact pattern (OIDC, plan on PR, approval-gated apply of the saved plan).
- **Multi-account**: one plan/apply role pair per AWS account, trust conditions per environment.
- **Platform templates**: reusable workflows ([GHA topic 22](../../GitHub-Actions-Basics-To-Production/notes/22-reusable-workflows.md)) so every infra repo gets the same pipeline.

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity` | Trust policy `sub`/`aud` doesn't match this run (wrong repo, branch, environment) or the provider isn't set up. | Compare the run's `sub` with the policy; check the OIDC provider ARN. |
| `Credentials could not be loaded, please check your action inputs: Could not load credentials from any providers` | Missing `permissions: id-token: write`. | Add it to the workflow/job. |
| `Error acquiring the state lock` in CI | Overlapping runs. | Concurrency per state; `-lock-timeout`. Don't `cancel-in-progress` applies. |
| `Saved plan is stale` in the apply job | State changed between plan and apply. | Re-run the workflow (new plan, new approval). |
| `Error: Failed to install provider ... locked` with `-lockfile=readonly` | The lock file doesn't match required providers. | Commit the updated `.terraform.lock.hcl` (with all platforms, topic 10). |
| PR comment step fails `Resource not accessible by integration` | Missing `pull-requests: write`, or a fork PR (read-only token). | Add the permission; for forks, skip the comment. |
| Plan works locally, `AccessDenied` in CI | The plan role lacks a read permission needed for refresh. | Add the specific read action to the plan role. |

---

## Quick recap

- **OIDC**: GitHub token → `AssumeRoleWithWebIdentity` → 1-hour credentials; no stored AWS keys.
- **Two roles**: `tf-plan` (read-only + lock; PRs and main) and `tf-apply` (write; **only** `environment:production`).
- **PR workflow**: fmt → init → validate → TFLint → Trivy → plan → comment.
- **Main workflow**: plan → artifact → **environment approval** → `apply tfplan`.
- Concurrency per state (never cancel applies), exact Terraform version, `-lockfile=readonly`, pinned actions, short artifact retention.

### Interview questions

1. **Q: How does your pipeline authenticate to AWS without access keys?**
   A: GitHub Actions OIDC: the job requests a signed token from GitHub, and `configure-aws-credentials` exchanges it via `sts:AssumeRoleWithWebIdentity` for temporary credentials. The IAM role's trust policy only accepts tokens from our repo and, for the apply role, only from the protected production environment.
2. **Q: How do you make sure the applied plan is the reviewed one in GitHub Actions?**
   A: The plan job saves `tfplan` as an artifact and shows it in the job summary; the apply job, gated by an environment approval, downloads that file and runs `terraform apply tfplan`. Terraform rejects it if it's stale.
3. **Q: Why separate plan and apply roles?**
   A: Plans run on untrusted code from pull requests, so they get read-only access. Only the approved production job on main can assume the write role, so a malicious or careless PR can't change infrastructure.
