# 61 · Capstone Part 3: Day-2 Operations

> **Level:** ★★★ advanced
>
> **You will learn:** what happens **after** the first deploy, in seven drills on your capstone: scheduled **drift detection** and response, **importing** a hand-made resource, a **refactor with `moved`** across environments, a **provider upgrade** with bots and lock files, an **AMI rollout** policy, a **state recovery** drill, and a safe **teardown**. Plus an optional, costed **EKS extension** showing the two-state cluster/workloads pattern.
>
> **You need:** topics 37 (drift), 39 (import), 40 (`moved`), 42 (recovery), 59–60. Topic 62 (upgrades) goes deeper on drill 4.
>
> **Part 10 · Running it in production** · Previous: [60 · Capstone part 2: environments and CI/CD](60-capstone-2-environments-and-cicd.md) · Next: [62 · Upgrades: Terraform, providers and modules](62-upgrades.md) · [Back to INDEX](../INDEX.md)
>
> 💰 **Cost:** as in parts 1–2 while the environments run. The optional EKS extension costs ≈ USD 0.10/hour for the control plane plus nodes: do it in one sitting and destroy.

---

## ⚡ Core: the 10-minute version

**Day 1** is building it. **Day 2** is everything after: drift, imports, refactors, upgrades, rollouts, incidents, teardown. Each drill below is a real task you'll get in a DevOps job, done the safe way: **PR → plan → review → apply**, never ad-hoc.

| Drill | Real-world trigger | Tools |
|---|---|---|
| 1. Drift | Someone changed prod in the console | Scheduled `plan -refresh-only`, revert/adopt |
| 2. Import | An SNS topic was created by hand | `import` block |
| 3. Refactor | Rename `module "web"` to `module "frontend"` | `moved` block |
| 4. Provider upgrade | A new AWS provider minor version | Dependabot/Renovate, `init -upgrade`, lock file |
| 5. AMI rollout | Monthly OS patching | Pinned AMI variable + instance refresh |
| 6. State recovery | A bad `state rm` in dev/web | S3 version restore |
| 7. Teardown | End of the project | Reverse dependency order |

**The one thing to know.** Every day-2 change is still **code + plan + review**. The plan tells you whether a "small" operation is actually a destroy.

**Try it now:** do drill 1 on your dev environment.

**You've got it if you can:** explain each drill's risk and how the plan proves it's safe.

> 📖 **That's the core.** Everything below is the full set of drills.

---

## What it is

Seven drills, each with: **trigger → steps → what the plan must show → done when**.

---

## Why it exists

Most Terraform work in a job is not greenfield. It's maintenance on infrastructure that already serves users: keeping code and reality aligned, absorbing changes from other people and tools, upgrading dependencies, and recovering from mistakes, without outages. These drills rehearse exactly that.

---

## Analogy

Owning a building after construction: inspections (drift), registering an extension a tenant built (import), renaming floors on the plans without moving walls (moved), replacing old wiring to code (upgrades), repainting floor by floor while people work (rolling AMI), restoring the plans from the archive after a coffee spill (state recovery), and finally demolition in the right order (teardown).

---

## How it works inside

All drills run through the capstone pipeline (topic 60) except the recovery drill, which is an operator action on state with backups (topic 42).

---

## Code / commands

### Drill 1: drift detection and response

Add `.github/workflows/drift.yml` (topic 37), using a matrix over all roots and the read-only plan role:

```yaml
name: drift
on:
  schedule: [{ cron: "0 */6 * * *" }]
  workflow_dispatch:
permissions: { contents: read, id-token: write, issues: write }
jobs:
  drift:
    strategy:
      fail-fast: false
      matrix:
        root: [envs/dev/network, envs/dev/web, envs/prod/network, envs/prod/web]
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: "${{ matrix.root }}" } }
    steps:
      - uses: actions/checkout@v5
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: 1.17.2, terraform_wrapper: false }
      - uses: aws-actions/configure-aws-credentials@v5
        with: { role-to-assume: "${{ vars.AWS_PLAN_ROLE_ARN }}", aws-region: "${{ vars.AWS_REGION }}" }
      - run: terraform init -lockfile=readonly
      - id: d
        run: |
          set +e
          terraform plan -refresh-only -detailed-exitcode -no-color -lock=false > drift.txt
          echo "code=$?" >> "$GITHUB_OUTPUT"
      - if: steps.d.outputs.code == '2'
        env: { GH_TOKEN: "${{ github.token }}" }
        run: gh issue create --title "Drift in ${{ matrix.root }}" --body-file drift.txt --label drift
```

Create drift in dev: in the console, change the dev ASG's **max size** from 3 to 5, and add a tag `hotfix=true` to the ALB. Run the workflow manually.

- The issue shows `aws_autoscaling_group.web has changed: max_size 3 -> 5` and the tag.
- **Decide**: the max size was an emergency scale-out → **adopt** it (`capacity = { min = 2, max = 5, desired = 2 }` in a PR; plan shows no changes for the ASG). The tag was noise → **revert** (the next apply removes it), or add `ignore_tags` for a tool-owned prefix.
- **Done when** the drift workflow is green for all roots.

### Drill 2: import a hand-made resource

Someone created an SNS topic `acme-dev-alerts` by hand for alarms. Bring it under Terraform in `envs/dev/web`, and add an alarm that uses it:

```bash
aws sns create-topic --name acme-dev-alerts     # simulate the hand-made resource
```

```hcl
# envs/dev/web/alerts.tf
data "aws_caller_identity" "current" {}

import {
  to = aws_sns_topic.alerts
  id = "arn:aws:sns:ap-south-1:${data.aws_caller_identity.current.account_id}:acme-dev-alerts"
}

resource "aws_sns_topic" "alerts" {
  name = "acme-dev-alerts"
}

resource "aws_cloudwatch_metric_alarm" "unhealthy_hosts" {
  alarm_name          = "acme-dev-web-unhealthy-hosts"
  namespace           = "AWS/ApplicationELB"
  metric_name         = "UnHealthyHostCount"
  statistic           = "Maximum"
  period              = 60
  evaluation_periods  = 3
  threshold           = 0
  comparison_operator = "GreaterThanThreshold"
  dimensions = {
    TargetGroup  = split(":", module.web.target_group)[5]      # "targetgroup/web-xxxx/abc..."
    LoadBalancer = module.web.alb_arn_suffix
  }
  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

(Add `output "alb_arn_suffix" { value = aws_lb.web.arn_suffix }` to the web module, and use `aws_lb_target_group.web.arn_suffix` similarly instead of the `split` if you prefer.)

- The plan must show: `aws_sns_topic.alerts will be imported` (0 to change) and `+ aws_cloudwatch_metric_alarm`.
- **Done when** a second plan says `No changes` and the import block can be removed.

### Drill 3: refactor with `moved`

The team wants `module "web"` called `module "frontend"` in both environments. In `envs/dev/web/main.tf` and `envs/prod/web/main.tf`:

```hcl
module "frontend" {                 # renamed
  source = "../../../modules/web"
  # ... same inputs ...
}

moved {
  from = module.web
  to   = module.frontend
}
```

Update every `module.web.*` reference. The PR plans for both roots must show **only** `has moved to` lines: `Plan: 0 to add, 0 to change, 0 to destroy`. Without the `moved` block, the prod plan would destroy the ALB (and the protect policy from topic 60 would block it). **Done when** both environments are applied and plans are clean. Keep the `moved` blocks for a while (other branches may still use the old name).

### Drill 4: provider upgrade

Automate upgrade PRs with Dependabot:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: terraform
    directories: ["/envs/*/*", "/modules/*"]
    schedule: { interval: weekly }
  - package-ecosystem: github-actions
    directory: "/"
    schedule: { interval: weekly }
```

A bot PR bumps the provider in `.terraform.lock.hcl` (within your `~> 6.0` constraints) or a module version. The pipeline plans every affected root. Your job:

1. Read the provider **changelog** between the versions (look for "BREAKING", deprecations, behaviour changes).
2. Read **every plan**: a provider upgrade should usually show `No changes`. Any diff means changed defaults or schema; understand it before merging.
3. Make sure the lock file has hashes for every platform your team and CI use (`terraform providers lock -platform=linux_amd64 -platform=darwin_arm64 -platform=windows_amd64`, topic 10).

**Done when** dev and prod plans are clean after merging. A **major** upgrade (e.g. AWS 6 → 7, one day) gets its own project: topic 62.

### Drill 5: AMI rollout policy

`most_recent = true` in the launch template means "a new AMI whenever AWS publishes one", which triggers an instance refresh at a random time on someone's unrelated apply. Make it explicit:

```hcl
# modules/web/variables.tf
variable "ami_id" {
  type        = string
  description = "AMI to run. Updated deliberately by PR (monthly patching)."
  default     = null              # null → latest AL2023 (dev convenience)
}
```

```hcl
# modules/web/compute.tf
image_id = coalesce(var.ami_id, data.aws_ami.al2023.id)
```

- **dev**: `ami_id = null` (always latest: early warning).
- **prod**: a pinned `ami_id` in `terraform.tfvars`, bumped monthly by a PR after dev has run that AMI for a week.

The prod plan for a bump shows `~ aws_launch_template.web` (new version) and the ASG's `launch_template.version` change → **instance refresh** rolls instances with ≥ 50% healthy. **Done when** prod runs the new AMI without failed health checks (watch the ALB target health during the refresh).

### Drill 6: state recovery

In `envs/dev/web`, simulate a mistake:

```bash
terraform state pull > backup-before-drill.tfstate
terraform state rm module.frontend.aws_lb.web        # oops
terraform plan                                       # wants to CREATE an ALB that exists
```

Recover by restoring the previous S3 version (topic 42, scenario C):

```bash
B=acme-tfstate-123456789012; K=dev/web/terraform.tfstate
aws s3api list-object-versions --bucket $B --prefix $K --max-items 3 \
  --query 'Versions[].{v:VersionId,at:LastModified,latest:IsLatest}' --output table
aws s3api copy-object --bucket $B --key $K --copy-source "$B/$K?versionId=<PREVIOUS>"
terraform plan                                       # No changes
```

**Done when** the plan is clean and you've written a 5-line incident note (what, impact, detection, fix, prevention, e.g. "state rm requires a second person").

### Drill 7: teardown

Order matters (reverse of dependencies):

1. **prod/web**: first apply `deletion_protection = false` (a reviewed PR), then `terraform destroy`.
2. **prod/network**.
3. **dev/web** (also removes the imported SNS topic and the alarm).
4. **dev/network**.
5. Check for leftovers (topic 29's scanner; **EC2 → Load Balancers**, **Auto Scaling groups**, **Elastic IPs**, **SSM parameters** under `/acme/`).
6. Keep **bootstrap** (state bucket + roles) if you'll continue; otherwise remove `prevent_destroy`, empty all object versions, destroy it, and delete the GitHub environments/variables.

### Optional: EKS extension (💰)

To practise the cluster/workloads split from topic 28 on AWS:

```
envs/dev/eks-cluster/   → module "eks" (terraform-aws-modules/eks/aws), node group of 2 × t3.medium
                          in private subnets (needs NAT: enable_nat_gateway = true in dev/network, 💰)
                          outputs: cluster_name, endpoint, CA → published to SSM
envs/dev/eks-addons/    → separate state; provider "kubernetes"/"helm" configured from SSM values
                          with an exec plugin: `aws eks get-token --cluster-name ...`
                          helm_release: aws-load-balancer-controller, metrics-server
```

Two states exist because the Kubernetes provider must be configured from a cluster that already exists (topic 28). Expect ≈ USD 0.10/h (control plane) + nodes + NAT; build, test, destroy in one sitting (destroy addons first, then the cluster, then NAT).

---

## Hands-on lab

Do drills 1–7 in order (the EKS extension is optional). For each, keep a one-paragraph log: trigger, PR link, plan summary, result. That log is excellent interview material.

---

## Where you'll use it

Every week of a DevOps/SRE job: drift tickets, adopting hand-made resources, renames and module refactors, bot PRs for providers, monthly patching, the occasional state incident, and decommissioning.

---

## When it breaks

| Symptom | Drill | Fix |
|---|---|---|
| Drift issue every 6 hours for the same tag | 1 | Adopt or `ignore_tags` for tool-owned tags |
| Import plan shows `1 to change` | 2 | Align config with reality first |
| Refactor plan shows destroy/create | 3 | Missing or wrong `moved` block |
| Provider bump shows unexpected diffs | 4 | Read the changelog; pin back; fix config |
| Instance refresh fails / rolls back | 5 | New AMI broke the app; check target health and cloud-init logs; revert the `ami_id` PR |
| Restore picked the wrong version | 6 | List versions with sizes/times; compare `serial` and resource counts before copying |
| Destroy fails with `DependencyViolation` | 7 | Wrong order or leftover ENIs/LBs; destroy dependents first |

---

## Quick recap

- Day-2 work = drift, imports, refactors, upgrades, rollouts, recovery, teardown, all through **PR → plan → review → apply**.
- Each drill has a **plan signature**: imports show "will be imported, 0 to change"; refactors "has moved to, 0/0/0"; provider bumps ideally "No changes"; AMI bumps "launch template + ASG version".
- Record what you did; it's the history the next engineer (and auditor) needs.

### Interview questions

1. **Q: What do you look for in a plan for a provider upgrade PR?**
   A: Ideally no changes. Any diff means the provider changed defaults, schemas or behaviour; I read the changelog for that version range, understand each diff before merging, and roll through dev first.
2. **Q: How do you patch the OS on instances managed by an Auto Scaling group with Terraform?**
   A: Update the launch template's AMI (pinned variable, bumped by PR after testing in dev); the ASG's instance refresh replaces instances in rolling batches while the load balancer keeps healthy ones serving.
3. **Q: You renamed a module and the plan wants to destroy the load balancer. What happened and how do you fix it?**
   A: The resource addresses changed, so Terraform sees old resources removed and new ones added. Add a `moved` block from the old module address to the new one; the plan should then show only moves.
