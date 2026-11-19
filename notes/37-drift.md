# 37 · Drift

> **Level:** ★★ intermediate
>
> **You will learn:** what **drift** is and where it comes from (emergency console fixes, other tools, AWS itself, deletions), how Terraform **detects** it (and what it **can't** see), the four ways to **respond** (revert it, adopt it into code, accept it into state, ignore it), how to **detect drift on a schedule** in CI and alert on it, and how to **prevent** it (read-only console access, guardrails, ownership tags). You'll create drift in your free topic 32 network and handle each case.
>
> **You need:** topics 09 (`-detailed-exitcode`), 24 (`ignore_changes`), 30 (`ignore_tags`), 32 (network), 34 (refresh modes), 35–36.
>
> **Part 5 · State in depth** · Previous: [36 · State locking](36-state-locking.md) · Next: [38 · Sharing state between configurations](38-sharing-state.md) · [Back to INDEX](../INDEX.md)
>
> 💰 **Cost:** the topic 32 network is free.

---

## ⚡ Core: the 10-minute version

**The idea.** **Drift** = real infrastructure no longer matches what Terraform last recorded (and usually what the code says). Terraform finds it during **refresh**, so every `plan` shows it as `Note: Objects have changed outside of Terraform`.

For each drift, decide:

| Decision | When | How |
|---|---|---|
| **Revert** | The change was wrong or unauthorised | `terraform apply` (puts reality back to the code) |
| **Adopt** | The change was right | Update the **code** to match, then `plan` → `No changes` |
| **Accept into state** | Only state is stale (no config difference) | `terraform apply -refresh-only` |
| **Ignore** | Another system legitimately owns that attribute | `lifecycle { ignore_changes = [...] }` / provider `ignore_tags` |

**Picture it.** Your house plans say "the kitchen wall is white". Someone painted it red. You either repaint it white (revert), change the plans to say red (adopt), or decide the plans don't care about paint colour (ignore).

**The one thing to know.** Terraform only sees drift in **attributes of resources it manages**. Things created by hand that it doesn't manage are **invisible** to it, including an extra security group rule added next to your separately managed rules.

**Try it now:** `terraform plan -refresh-only` in your topic 32 project after adding a tag to the VPC with the AWS CLI.

**You've got it if you can:** name the four responses and pick the right one for "someone opened port 22 to the world during an incident".

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Where drift comes from

| Source | Example |
|---|---|
| **Emergency fixes in the console** | An engineer widens a security group during an outage and forgets to update code |
| **Other automation** | Autoscalers change counts; a security scanner adds tags; a Kubernetes controller edits a load balancer |
| **The platform itself** | AWS adds default values, rotates something, or upgrades an engine version |
| **Deletion** | Someone deletes a resource, or a cleanup script runs |
| **Another Terraform configuration** | Two configurations manage the same object (a bug: one owner per resource) |

### What Terraform can and can't see

| Situation | Seen? |
|---|---|
| An **attribute** of a managed resource changed (instance type, a tag, an inline SG rule) | ✅ Yes, on refresh |
| A managed resource was **deleted** | ✅ Yes ("has been deleted") |
| A **new object** created by hand (a new VM, a new bucket) | ❌ No: not in state |
| A new SG rule added by hand to a group whose rules are **separate resources** | ❌ No: the extra rule is its own object, not in state |
| The same rule added to a group with **inline** rules | ✅ Yes: the group's rule set changed (Terraform will remove it) |
| A tag added by hand to a managed resource | ✅ Yes (and Terraform removes it), unless `ignore_tags` |

So "No changes" means "**what Terraform manages** matches", not "nobody touched the account". Finding unmanaged objects needs other tools: AWS Config (an AWS service that records resource configurations and checks them against rules), tag-based inventory ("everything without `managed_by=terraform`"), or `terraform query` / list resources (Terraform 1.14+, topic 67) to list what exists and generate import blocks.

---

## Why it exists

Drift is inevitable in real organisations. The danger isn't drift itself; it's **unnoticed** drift:

- The next unrelated apply silently **reverts** an emergency fix → the outage comes back.
- Code no longer describes production → rebuilding (DR, a new region) produces something different.
- Security changes (open ports, public buckets) hide until an audit or a breach.

Detecting drift early and deciding consciously keeps code and reality honest.

---

## Analogy

**House plans and a renovation.**

- The **plans** (code) say the kitchen wall is white with one window.
- A builder, under pressure, painted it red and added a second window (console changes).
- The **inspector** (refresh) compares the house with the logbook (state) and reports both.
- You decide: repaint and brick up the window (**revert**), update the plans to red and two windows (**adopt**), or mark paint colour as "tenant's choice" in the plans (**ignore**).
- A brand-new **shed** someone built in the garden isn't in the logbook, so the inspector never checks it (unmanaged resources).

---

## How it works inside

Detection is just refresh (topic 34):

```
plan → for each resource in state: Read(id) → compare with last recorded attributes
     → differences = drift → printed under "Objects have changed outside of Terraform"
     → then config vs refreshed state = the actions (which may undo the drift)
```

`plan -refresh-only` prints **only** the drift section. `-detailed-exitcode` returns `2` when there's anything to do:

| Command | Exit 2 means |
|---|---|
| `terraform plan -detailed-exitcode` | Config and reality differ (drift **or** unapplied code changes) |
| `terraform plan -refresh-only -detailed-exitcode` | Reality differs from **state** (pure drift) |

For scheduled drift checks on the main branch (where code == what was last applied), either works; `-refresh-only` isolates drift from code changes that haven't been applied yet.

---

## Code / commands

Re-apply your topic 32 network if you destroyed it (`~/tf-lab/32-network`, free). Then create drift with the AWS CLI.

### Drift 1: a tag added by hand

```bash
VPC=$(terraform output -raw vpc_id)
aws ec2 create-tags --resources "$VPC" --tags Key=owner,Value=someone-else
terraform plan
```

```
Note: Objects have changed outside of Terraform

Terraform detected the following changes made outside of Terraform since the
last "terraform apply" which may have affected this plan:

  # aws_vpc.main has changed
  ~ resource "aws_vpc" "main" {
        id                                   = "vpc-0a1b2c3d4e5f6a7b8"
      ~ tags                                 = {
          + "owner" = "someone-else"
            # (1 unchanged element hidden)
        }
      ~ tags_all                             = {
          + "owner"      = "someone-else"
            # (4 unchanged elements hidden)
        }
        # (16 unchanged attributes hidden)
    }

Unless you have made equivalent changes to your configuration, or ignored the
relevant attributes using ignore_changes, the following plan may include
actions to undo or respond to these changes.

─────────────────────────────────────────────────────────────────────────────

Terraform will perform the following actions:

  # aws_vpc.main will be updated in-place
  ~ resource "aws_vpc" "main" {
        id                                   = "vpc-0a1b2c3d4e5f6a7b8"
      ~ tags                                 = {
          - "owner" = "someone-else" -> null
            # (1 unchanged element hidden)
        }
      ~ tags_all                             = {
          - "owner"      = "someone-else" -> null
            # (4 unchanged elements hidden)
        }
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

Read it in two halves:

1. **"has changed"**: the drift report (what happened outside Terraform).
2. **"will be updated in-place"**: what Terraform will do about it (remove the tag).

Responses:

- **Revert**: `terraform apply` → tag removed.
- **Adopt**: add `owner = "someone-else"` to the VPC's `tags` in code → plan says `No changes`.
- **Ignore (tags written by other tools)**: in the provider, `ignore_tags { keys = ["owner"] }` (or `key_prefixes = ["scanner:"]`) → plan says `No changes` and Terraform never fights over that tag.

### Drift 2: a security group rule changed by hand

Add an SSH rule from anywhere to the web SG, as someone might during an incident:

```bash
SG=$(terraform output -raw web_sg_id)
aws ec2 authorize-security-group-ingress --group-id "$SG" --protocol tcp --port 22 --cidr 0.0.0.0/0
terraform plan
```

```
No changes. Your infrastructure matches the configuration.
```

**Nothing.** Your rules are **separate resources** (`aws_vpc_security_group_ingress_rule`), and this new rule is a separate object Terraform doesn't manage, so it's invisible. Port 22 is open to the world and every plan is green.

Compare: if the group used **inline** `ingress {}` blocks, the plan would show the group "has changed" and would remove the rule. Separate rules are still the better design (topic 32); the lesson is that **"No changes" is not a security audit**. Catch this with:

- AWS Config managed rules (e.g. `restricted-ssh`) or Security Hub,
- a `check` block that lists the group's rules with a data source and asserts (topic 25):

```hcl
data "aws_vpc_security_group_rules" "web" {
  filter {
    name   = "group-id"
    values = [aws_security_group.web.id]
  }
}

check "web_sg_only_expected_rules" {
  assert {
    condition     = length(data.aws_vpc_security_group_rules.web.ids) == 3   # http, ssh-admin, egress
    error_message = "Unexpected rules on ${aws_security_group.web.id}: someone changed it outside Terraform."
  }
}
```

Now the plan **warns**. Remove the rule (`aws ec2 revoke-security-group-ingress --group-id "$SG" --protocol tcp --port 22 --cidr 0.0.0.0/0`) and the warning goes away.

### Drift 3: a managed rule edited by hand

```bash
RULE=$(aws ec2 describe-security-group-rules --filters Name=group-id,Values="$SG" \
  --query 'SecurityGroupRules[?FromPort==`80` && !IsEgress].SecurityGroupRuleId' --output text)
aws ec2 modify-security-group-rules --group-id "$SG" --security-group-rules \
  "SecurityGroupRuleId=$RULE,SecurityGroupRule={IpProtocol=tcp,FromPort=8080,ToPort=8080,CidrIpv4=0.0.0.0/0}"
terraform plan
```

```
  # aws_vpc_security_group_ingress_rule.web_http has changed
  ~ resource "aws_vpc_security_group_ingress_rule" "web_http" {
      ~ from_port = 80 -> 8080
      ~ to_port   = 80 -> 8080
...
  # aws_vpc_security_group_ingress_rule.web_http will be updated in-place
  ~ resource "aws_vpc_security_group_ingress_rule" "web_http" {
      ~ from_port = 8080 -> 80
      ~ to_port   = 8080 -> 80
...
```

The managed rule is seen and will be **reverted**. `terraform apply`. (Whether a change like this is in place or a replacement is decided by the provider, topic 13. Read the symbol in your own plan.)

### Drift 4: deleted

```bash
aws ec2 delete-route --route-table-id "$(aws ec2 describe-route-tables \
  --filters Name=tag:Name,Values=tf-course-public-rt --query 'RouteTables[0].RouteTableId' --output text)" \
  --destination-cidr-block 0.0.0.0/0
terraform plan
```

```
  # aws_route.public_internet has been deleted
...
  # aws_route.public_internet will be created
```

Public subnets just lost internet access; Terraform puts the route back on apply.

### Scheduled drift detection (GitHub Actions)

```yaml
# .github/workflows/drift.yml
name: drift-check
on:
  schedule:
    - cron: "0 */6 * * *"          # every 6 hours (UTC)
  workflow_dispatch:

permissions:
  id-token: write                  # OIDC to AWS (topic 57)
  contents: read
  issues: write

jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.17.2
          terraform_wrapper: false
      - uses: aws-actions/configure-aws-credentials@v5
        with:
          role-to-assume: ${{ vars.TF_READONLY_ROLE_ARN }}   # read-only role is enough for plans
          aws-region: ap-south-1
      - run: terraform init -input=false -backend-config=backends/prod.s3.tfbackend
      - id: plan
        run: |
          set +e
          terraform plan -refresh-only -detailed-exitcode -input=false -no-color -lock=false > drift.txt
          echo "exit=$?" >> "$GITHUB_OUTPUT"
      - if: steps.plan.outputs.exit == '2'
        env:
          GH_TOKEN: ${{ github.token }}
        run: gh issue create --title "Drift detected in prod $(date -u +%F)" --body-file drift.txt
      - if: steps.plan.outputs.exit == '1'
        run: exit 1
```

- `-refresh-only -detailed-exitcode`: exit `2` = drift.
- `-lock=false` is acceptable **here**: it's a read-only check that must not block real applies (topic 36).
- Action versions here are illustrative; topic 57 explains each piece and pins them properly.
- HCP Terraform has built-in **drift detection** and health checks that do the same (topic 50).

### Preventing drift

| Control | How |
|---|---|
| **Read-only console by default** | Humans get a `ReadOnly` permission set; write access only through pipelines, plus a break-glass role for emergencies (logged). |
| **Guardrails** | AWS Organizations **SCPs** (Service Control Policies) deny changes to resources tagged `managed_by=terraform` except from the pipeline role. |
| **Fast path to change code** | If a code change takes a day, people will click. A 10-minute PR → plan → apply pipeline is the best prevention. |
| **Post-incident rule** | Every emergency console change gets a follow-up PR the same day (adopt or revert). |
| **Tag everything** | `managed_by = "terraform"`, `repo`, `stack` via `default_tags`, so anyone in the console knows not to touch it and where its code lives. |

---

## Hands-on lab

1. In the topic 32 network, create **Drift 1** (tag) and handle it three ways: revert, adopt, `ignore_tags`.
2. Create **Drift 2** (extra SSH rule). See `No changes`. Add the `check` block, see the warning, revoke the rule.
3. Create **Drift 3** (edit a managed rule) and **Drift 4** (delete the route); apply to fix both.
4. Run `terraform plan -refresh-only -detailed-exitcode; echo $?` before and after each change.
5. Write the drift workflow in a test repo (topic 57 completes the OIDC part).
6. `terraform destroy` if you don't need the network for topic 38 (you will, so maybe keep it).

---

## Where you'll use it

- **Every on-call week**: "the plan shows drift on prod". Decide revert / adopt / ignore, and fix the process that let it happen.
- **Scheduled checks** in CI or HCP Terraform, with alerts to Slack or issues.
- **Security**: drift on security groups, bucket policies and IAM is a real incident signal.
- **Interviews**: "How do you handle configuration drift?" is a standard Terraform question.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| An apply undid someone's emergency fix | Drift wasn't noticed or adopted before an unrelated apply. | Read the drift section of every plan; adopt fixes in code the same day. |
| Endless drift on tags | Another tool writes tags. | `ignore_tags` in the provider. |
| Endless drift on one attribute (e.g. `desired_count`, `replicas`) | Another system owns it. | `ignore_changes` on that attribute. |
| Endless drift that comes back after apply | The provider normalises a value differently (case, order, default). | Write the value as the API returns it; check for a known provider issue. |
| Plan is green but something is clearly wrong | Unmanaged objects (separate rules, new resources) are invisible. | Guardrails, AWS Config, checks, inventory by tags; import what should be managed. |
| Drift check pipeline blocks real applies | It took the lock. | `-lock=false` for read-only drift checks only. |

---

## Quick recap

- **Drift** = reality changed outside Terraform. Refresh detects it in **managed** attributes; unmanaged objects are invisible.
- Respond by **reverting** (apply), **adopting** (change code), **accepting** (`apply -refresh-only`), or **ignoring** (`ignore_changes`, `ignore_tags`).
- Detect on a schedule: `plan -refresh-only -detailed-exitcode` → exit 2 → alert.
- Prevent with read-only consoles, SCP guardrails, a fast PR pipeline and same-day follow-up of emergency changes.

### Interview questions

1. **Q: What is drift and how do you detect it?**
   A: Differences between real infrastructure and Terraform's state/config caused by changes outside Terraform. Every plan refreshes and reports them; for continuous detection, run `terraform plan -refresh-only -detailed-exitcode` on a schedule (exit code 2 = drift), or use HCP Terraform's drift detection.
2. **Q: Someone fixed production in the console during an incident. What do you do?**
   A: Look at the drift in a plan, decide whether the change should stay. If yes, update the code to match and apply through the normal PR process (plan should show no changes); if not, revert it with an apply. Then fix the process, e.g. read-only console access and a faster pipeline.
3. **Q: Your plan says "No changes" but someone added an SSH rule to a managed security group. How?**
   A: The group's rules are managed as separate resources, so a new hand-made rule is a separate unmanaged object that Terraform doesn't track. Catch it with AWS Config rules, guardrails or a check block that asserts the group's rule list.
