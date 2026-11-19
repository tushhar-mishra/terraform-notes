# 60 · Capstone Part 2: Environments and CI/CD

> **Level:** ★★★ advanced
>
> **You will learn:** to turn the part 1 codebase into a **team-ready system**: add a **prod** environment (bigger, protected), build a **reusable GitHub Actions workflow** that checks, scans, tests, policy-checks and plans **one root**, then call it to **promote** changes in order (dev network → dev web → prod network → prod web), with **environment approvals** and per-environment **OIDC roles**, and quality gates on every PR.
>
> **You need:** topics 49 (environments), 53–57, 59.
>
> **Part 10 · Running it in production** · Previous: [59 · Capstone part 1: foundation](59-capstone-1-foundation.md) · Next: [61 · Capstone part 3: day-2 operations](61-capstone-3-day-2.md) · [Back to INDEX](../INDEX.md)
>
> 💰 **Cost:** two environments ≈ two ALBs + four small instances while running. Work in sessions and destroy at the end of each (part 3 covers teardown).

---

## ⚡ Core: the 10-minute version

**The pipeline:**

```
PR ──▶ terraform-pr.yml: for each changed root → fmt · validate · tflint · trivy · tests · plan · conftest · comment
merge ──▶ terraform-main.yml:
          dev/network ─▶ dev/web ─▶ prod/network ─▶ prod/web
          (auto apply)  (auto apply)  (⏸ approval)    (⏸ approval)
          each step = reusable workflow: plan → artifact → [environment gate] → apply tfplan
```

**Prod differs from dev only in inputs:**

| | dev | prod |
|---|---|---|
| CIDR | `10.30.0.0/16` | `10.40.0.0/16` |
| Capacity | 2–3 | 2–6 |
| ALB deletion protection | off | **on** |
| State key | `dev/...` | `prod/...` (real life: a **separate account and bucket**) |
| Apply | automatic after merge | **human approval** |
| AWS role | `tf-apply-dev` | `tf-apply-prod` (only from environment `production`) |

**The one thing to know.** One reusable workflow encodes your standards **once**; every root and environment uses it. Promotion order is just `needs:` between calls.

**Try it now:** add `envs/prod`, push the workflows, open a PR that changes `page_message` in both environments, and follow it from PR to prod.

**You've got it if you can:** explain how a single merge reaches prod, and where a human must approve.

> 📖 **That's the core.** Everything below is the full build.

---

## What it is

### Prod environment

`envs/prod/network/main.tf` and `envs/prod/web/main.tf` are copies of the dev roots with:

- backend keys `prod/network/...` and `prod/web/...`,
- `environment = "prod"`, `cidr = "10.40.0.0/16"`,
- `capacity = { min = 2, max = 6, desired = 2 }`,
- `default_tags` with `environment = "prod"`,
- and a new module input for ALB protection.

Add to `modules/web`:

```hcl
variable "deletion_protection" {
  type    = bool
  default = false
}
```

```hcl
resource "aws_lb" "web" {
  # ...
  enable_deletion_protection = var.deletion_protection
}
```

And in `envs/prod/web/main.tf`: `deletion_protection = true`.

> In a real company, prod lives in **another AWS account**, with its own state bucket and roles. In this lab, both environments share one account; separate keys and roles give you the same **workflow**, and `allowed_account_ids` per root shows where the account boundary would be.

### Roles per environment (update the bootstrap)

Extend topic 57's bootstrap: keep `tf-plan`; replace `tf-apply` with two roles:

| Role | Trust `sub` |
|---|---|
| `tf-apply-dev` | `repo:ORG/acme-infra:environment:dev` |
| `tf-apply-prod` | `repo:ORG/acme-infra:environment:production` |

In GitHub: environment **`dev`** (deployment branch `main`, no reviewers) and **`production`** (deployment branch `main`, **required reviewers**). Repository variables: `AWS_PLAN_ROLE_ARN`, `AWS_APPLY_ROLE_ARN_DEV`, `AWS_APPLY_ROLE_ARN_PROD`, `AWS_REGION`.

---

## Why it exists

Part 1 worked from a laptop. Teams need the same result **without** laptops: every change reviewed with its plan, scanned, tested and policy-checked, applied by a pipeline in a safe order, with humans approving production. Building it once as a reusable workflow keeps every root consistent.

---

## Analogy

A **factory production line** for infrastructure changes: every change goes down the same belt (reusable workflow) with the same inspection stations (fmt, lint, scan, test, policy, plan), first into the test kitchen (dev), and only after a supervisor signs (approval) into the flagship restaurant (prod).

---

## How it works inside

```
terraform-main.yml (on push to main)
  call tf-root.yml (root=envs/dev/network,  environment=dev)        plan → apply (auto)
     needs ▼
  call tf-root.yml (root=envs/dev/web,      environment=dev)        plan → apply (auto)
     needs ▼
  call tf-root.yml (root=envs/prod/network, environment=production) plan → ⏸ approve → apply
     needs ▼
  call tf-root.yml (root=envs/prod/web,     environment=production) plan → ⏸ approve → apply
Each call: if plan shows no changes, the apply job is skipped (no approval requested).
A failure stops everything after it (dev must succeed before prod starts).
```

---

## Code / commands

### The reusable workflow: `.github/workflows/tf-root.yml`

```yaml
name: tf-root
on:
  workflow_call:
    inputs:
      root:
        type: string
        required: true
      environment:
        type: string
        required: true          # "dev" or "production" (GitHub environment name)
      apply:
        type: boolean
        default: false          # PRs: plan only
      apply_role_var:
        type: string
        default: ""             # name of the repo variable with the apply role ARN
      artifact_name:
        type: string
        default: "plan"         # unique per root, e.g. "dev-network" (artifact names can't contain "/")

env:
  TF_IN_AUTOMATION: "1"
  TF_INPUT: "0"
  TF_VERSION: "1.17.2"

jobs:
  plan:
    runs-on: ubuntu-latest
    outputs:
      changes: ${{ steps.plan.outputs.changes }}
    defaults:
      run:
        working-directory: ${{ inputs.root }}
    steps:
      - uses: actions/checkout@v5

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false

      - uses: aws-actions/configure-aws-credentials@v5
        with:
          role-to-assume: ${{ vars.AWS_PLAN_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Static checks
        run: |
          terraform fmt -check -recursive "$GITHUB_WORKSPACE"
          terraform init -lockfile=readonly
          terraform validate

      - uses: terraform-linters/setup-tflint@v4
      - name: TFLint
        run: tflint --init && tflint --recursive --config "$GITHUB_WORKSPACE/.tflint.hcl"

      - name: Trivy
        uses: aquasecurity/trivy-action@0.33.1
        with:
          scan-type: config
          scan-ref: ${{ inputs.root }}
          severity: HIGH,CRITICAL
          exit-code: "1"

      - name: Plan
        id: plan
        run: |
          set +e
          terraform plan -out=tfplan -lock-timeout=10m -detailed-exitcode -no-color > plan.txt 2>&1
          code=$?
          cat plan.txt
          [ "$code" -eq 1 ] && exit 1
          echo "changes=$([ "$code" -eq 2 ] && echo true || echo false)" >> "$GITHUB_OUTPUT"
          {
            echo "### ${{ inputs.root }}"
            echo '```'
            grep -E '^(Plan:|No changes|  #)' plan.txt || true
            echo '```'
          } >> "$GITHUB_STEP_SUMMARY"
          terraform show -json tfplan > plan.json

      - name: Policies
        run: |
          curl -fsSL https://github.com/open-policy-agent/conftest/releases/download/v0.62.0/conftest_0.62.0_Linux_x86_64.tar.gz | tar xz conftest
          ./conftest test plan.json -p "$GITHUB_WORKSPACE/policy"

      - uses: actions/upload-artifact@v4
        if: inputs.apply && steps.plan.outputs.changes == 'true'
        with:
          name: tfplan-${{ inputs.artifact_name }}
          path: |
            ${{ inputs.root }}/tfplan
            ${{ inputs.root }}/plan.txt
          retention-days: 3

  apply:
    needs: plan
    if: inputs.apply && needs.plan.outputs.changes == 'true'
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}          # ⏸ production has required reviewers
    concurrency:
      group: tf-state-${{ inputs.root }}
      cancel-in-progress: false
    defaults:
      run:
        working-directory: ${{ inputs.root }}
    steps:
      - uses: actions/checkout@v5
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false
      - uses: aws-actions/configure-aws-credentials@v5
        with:
          role-to-assume: ${{ vars[inputs.apply_role_var] }}
          aws-region: ${{ vars.AWS_REGION }}
      - uses: actions/download-artifact@v4
        with:
          name: tfplan-${{ inputs.artifact_name }}      # exactly THIS root's plan
          path: ${{ inputs.root }}
      - run: terraform init -lockfile=readonly
      - run: terraform apply -lock-timeout=10m tfplan
```

- One place for **all** checks: every root gets identical treatment.
- `vars[inputs.apply_role_var]`: picks the right apply role per environment.
- The approver of `production` sees each root's plan summary in the run summary and the full `plan.txt` in the artifact.
- Conftest version and URL: pin to a real release (check `github.com/open-policy-agent/conftest/releases`).
- Artifacts are shared by **all jobs in the workflow run**, including every reusable-workflow call. That's why each root gets its **own artifact name** (`artifact_name`), and the apply job downloads exactly that one. Downloading by a wildcard pattern would mix plans from different roots, and you could apply the wrong plan.

### PR workflow: `.github/workflows/terraform-pr.yml`

```yaml
name: terraform-pr
on:
  pull_request:

permissions:
  contents: read
  id-token: write
  pull-requests: write

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      roots: ${{ steps.detect.outputs.roots }}
    steps:
      - uses: actions/checkout@v5
        with:
          fetch-depth: 0
      - id: detect
        run: |
          base=${{ github.event.pull_request.base.sha }}
          changed=$(git diff --name-only "$base"...HEAD)
          roots=$(printf '%s\n' "$changed" | grep -oE '^envs/[^/]+/[^/]+' | sort -u)
          # a module change → plan every root
          if printf '%s\n' "$changed" | grep -q '^modules/'; then
            roots=$(ls -d envs/*/* )
          fi
          echo "roots=$(printf '%s\n' $roots | jq -R . | jq -cs .)" >> "$GITHUB_OUTPUT"

  module-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.17.2
      - run: |
          for m in modules/*/; do
            [ -d "$m/tests" ] || continue
            (cd "$m" && terraform init -backend=false && terraform test)   # mock providers: no AWS needed
          done

  plan:
    needs: changes
    if: needs.changes.outputs.roots != '[]'
    strategy:
      fail-fast: false
      matrix:
        root: ${{ fromJSON(needs.changes.outputs.roots) }}
    uses: ./.github/workflows/tf-root.yml
    with:
      root: ${{ matrix.root }}
      environment: ${{ contains(matrix.root, '/prod/') && 'production' || 'dev' }}
      apply: false
```

(Add a step in `tf-root.yml`'s plan job, or a separate job, to post `plan.txt` as a PR comment, as in topic 57.)

### Main workflow: `.github/workflows/terraform-main.yml`

```yaml
name: terraform-main
on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  id-token: write

jobs:
  dev-network:
    uses: ./.github/workflows/tf-root.yml
    with: { root: envs/dev/network, environment: dev, apply: true, apply_role_var: AWS_APPLY_ROLE_ARN_DEV, artifact_name: dev-network }

  dev-web:
    needs: dev-network
    uses: ./.github/workflows/tf-root.yml
    with: { root: envs/dev/web, environment: dev, apply: true, apply_role_var: AWS_APPLY_ROLE_ARN_DEV, artifact_name: dev-web }

  prod-network:
    needs: dev-web
    uses: ./.github/workflows/tf-root.yml
    with: { root: envs/prod/network, environment: production, apply: true, apply_role_var: AWS_APPLY_ROLE_ARN_PROD, artifact_name: prod-network }

  prod-web:
    needs: prod-network
    uses: ./.github/workflows/tf-root.yml
    with: { root: envs/prod/web, environment: production, apply: true, apply_role_var: AWS_APPLY_ROLE_ARN_PROD, artifact_name: prod-web }
```

### Policies used by the pipeline (`policy/`)

Reuse topic 54's `tags.rego` (requires `owner` and `project`, which your `default_tags` provide), `instances.rego` (allowed types), and `protect.rego` (no deletes of protected types). Add the ALB:

```rego
package main

deny contains msg if {
  some rc in input.resource_changes
  rc.type == "aws_lb"
  "delete" in rc.change.actions
  contains(rc.address, "prod")      # simple example; better: check tags_all.environment == "prod"
  msg := sprintf("%s: deleting a production load balancer needs a manual change", [rc.address])
}
```

### Walk a change through

1. `git switch -c feat/new-banner`; change `page_message` defaults in `envs/dev/web` and `envs/prod/web`.
2. Push, open a PR → `terraform-pr` runs: module tests, then a matrix plan for `envs/dev/web` and `envs/prod/web` → both show the launch template and ASG changes.
3. Review the plans (topic 56 checklist): `~ aws_launch_template.web` (new version), `~ aws_autoscaling_group.web` (version) → instance refresh. No destroys.
4. Merge → `terraform-main`: dev-network (no changes → no apply) → dev-web applies automatically → prod-network (no changes) → prod-web **waits for approval** → approve → applied.
5. `curl` both URLs.

---

## Hands-on lab

1. Add `envs/prod/*`, the `deletion_protection` input and the prod role/environments.
2. Push `acme-infra` to GitHub with the three workflows, `.tflint.hcl`, `policy/` and module tests.
3. Walk a change from PR to prod as above.
4. Open a PR that sets `instance_type = "m5.large"` in prod: the policy check fails the PR.
5. Open a PR that would destroy the prod ALB (e.g. rename the module call without a `moved` block): the plan shows the delete; the policy blocks it. Fix it with a `moved` block.
6. Check that a PR job **cannot** assume `tf-apply-prod` (topic 57 lab step 5).
7. **End of session:** destroy prod web → prod network → dev web → dev network (prod ALB first needs `deletion_protection = false` applied, then destroy).

---

## Where you'll use it

This is a complete, interview-ready story: "PRs get checks and plans for exactly the roots they touch; merges promote through dev to prod with approvals; per-environment OIDC roles; policies and tests gate everything; one reusable workflow keeps it consistent."

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` for prod apply | Environment name mismatch with the trust `sub` | `environment: production` ↔ `...:environment:production` |
| Prod waits for approval with no changes | `changes` output not wired | Check `-detailed-exitcode` handling and the `if:` |
| Matrix plans run for every root on every PR | Module change detection too broad, or detection failed | Refine the `changes` job; map modules → roots |
| `Saved plan is stale` in prod | A concurrent run changed prod state | Re-run; concurrency per state |
| Destroy fails: `Deletion protection is enabled` | ALB protection on | Apply `deletion_protection = false` first (a reviewed change) |
| Policy failures on resources without tags | Policy too strict | The `tags_all != null` guard (topic 54) |

---

## Quick recap

- Prod = same modules, different inputs, separate state/roles (separate account in real life), deletion protection, approvals.
- One **reusable workflow** = all checks + plan + gated apply for one root.
- PRs: detect changed roots → test modules → plan matrix → policies → comments.
- Main: promote in order with `needs:` (dev → prod), auto in dev, **approval** in prod, apply the saved plan.

### Interview questions

1. **Q: How do changes get from a PR to production in your setup?**
   A: The PR runs checks, module tests, scans, policies and plans for the affected roots. After review and merge, the main workflow runs the same reusable workflow per root in order: dev network, dev web, then prod network and prod web. Dev applies automatically; prod jobs pause for a required reviewer, then apply the saved plan with a prod-only OIDC role.
2. **Q: Why a reusable workflow?**
   A: To define the standard pipeline (checks, plan, policy, gated apply) once and apply it identically to every root and environment, so improvements and fixes happen in one place.
3. **Q: How do you stop a PR from deploying to prod?**
   A: PR jobs only get the read-only plan role; the prod apply role's trust policy only accepts tokens from the protected `production` environment, which is limited to the main branch and requires approval.
