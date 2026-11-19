# 54 · Policy as Code

> **Level:** ★★★ advanced
>
> **You will learn:** how to turn your organisation's rules ("every bucket has an owner tag", "no instance bigger than xlarge in dev", "never delete a database in an automated run") into **code that checks every plan** before apply: **OPA** (Open Policy Agent) with **Rego**, run by **Conftest** against `terraform show -json` output; testing policies with `opa test`; how **Sentinel** (on HCP Terraform) and **Terraform Policy** (built into the Terraform CLI since 1.17) do the same in HashiCorp's tooling; and **cost policies** with **Infracost**.
>
> **You need:** topics 09 (`show -json`, actions), 30 (`tags_all`), 53 (scanning, which uses built-in rules; this topic is about **your** rules).
>
> **Part 8 · Security and quality** · Previous: [53 · Linting and security scanning](53-linting-and-scanning.md) · Next: [55 · Testing Terraform](55-testing.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Scanners (topic 53) check **generic** best practices. **Policy as code** checks **your** rules, written as code, versioned and tested, evaluated against the **plan** before anything changes:

```rego
# policy/tags.rego
package main

deny contains msg if {
  some rc in input.resource_changes
  rc.mode == "managed"
  "create" in rc.change.actions
  not rc.change.after.tags_all.owner
  msg := sprintf("%s: every resource must have an 'owner' tag", [rc.address])
}
```

```bash
terraform plan -out=tfplan && terraform show -json tfplan > plan.json
conftest test plan.json            # exit code 1 if any deny matches
```

**Picture it.** A bank's rulebook turned into an automatic checker: every transfer request (plan) is checked against the rules ("no transfer above X without two signatures") **before** the money moves, not audited afterwards.

**The one thing to know.** Policies read the **plan JSON**: `resource_changes[]` with `address`, `type`, `change.actions` and `change.after` values. Learn that structure, and you can write any rule.

**Try it now:** write the tag policy above, run it against a plan from topic 31, see it fail, add the tag via `default_tags`, see it pass.

**You've got it if you can:** explain what Conftest reads, and write a rule that blocks deleting a database.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Where policies run

```
code ─▶ fmt/validate/tflint/trivy (topic 53) ─▶ plan ─▶ show -json ─▶ POLICIES ─▶ approval ─▶ apply
                                                                       │
              OPA/Conftest in CI, Terraform Policy (-policies, 1.17+), or Sentinel/OPA in HCP Terraform
```

### The plan JSON (what policies read)

```json
{
  "format_version": "1.2",
  "terraform_version": "1.17.2",
  "resource_changes": [
    {
      "address": "aws_s3_bucket.lab",
      "mode": "managed",
      "type": "aws_s3_bucket",
      "name": "lab",
      "provider_name": "registry.terraform.io/hashicorp/aws",
      "change": {
        "actions": ["create"],
        "before": null,
        "after": { "bucket": "tf-course-lab-9372b1c3", "force_destroy": true, "tags_all": { "project": "tf-course" } },
        "after_unknown": { "arn": true, "id": true }
      }
    }
  ],
  "configuration": { ... },
  "prior_state": { ... }
}
```

| Field | Use in policies |
|---|---|
| `resource_changes[].type`, `.address` | Which resource |
| `change.actions` | `["create"]`, `["update"]`, `["delete"]`, `["delete","create"]` (replace)... |
| `change.after` | Planned values (unknown ones are absent; `after_unknown` marks them) |
| `change.before` | Current values (for updates/deletes) |
| `configuration` | The config as written (e.g. to check module sources or provider settings) |

### OPA, Rego and Conftest

- **OPA** (Open Policy Agent, a CNCF project) evaluates policies written in **Rego**, a declarative query language. You've seen OPA in Kubernetes admission control (Gatekeeper, [Kubernetes topic 58](../../CKA-Certification-Course-2025/notes/58-admission-control.md)); the same language works on Terraform plans.
- **Conftest** is a CLI that runs Rego policies against files (JSON, YAML, HCL...). By default it looks for rules named `deny`, `violation` and `warn` in `package main` under `./policy`.
- Since **OPA 1.0** (Dec 2024), Rego uses the `if` and `contains` keywords by default (the syntax used here).

Rego in five lines:

```rego
deny contains msg if {        # "deny" is a set; add msg to it IF all lines below are true
  some rc in input.resource_changes   # loop: for some element rc of the list
  rc.type == "aws_instance"           # each line is a condition (AND)
  not allowed[rc.change.after.instance_type]   # "not" = the expression is false/undefined
  msg := sprintf("%s: type %s not allowed", [rc.address, rc.change.after.instance_type])
}
```

### Sentinel and Terraform Policy (HashiCorp platform)

- **Sentinel**: HashiCorp's policy language, built into HCP Terraform/Enterprise, with Terraform-specific imports (`tfplan/v2`, `tfconfig/v2`, `tfstate/v2`) and **enforcement levels**: `advisory` (warn), `soft-mandatory` (can be overridden by authorised users), `hard-mandatory` (blocks).
- HCP Terraform also runs **OPA** policies natively.
- Terraform **1.17** made HashiCorp's newer **Terraform Policy** generally available, **in the CLI itself**: `plan`, `apply` and `query` accept a `-policies` flag pointing at your policy files, so policies run without HCP Terraform or Conftest. It's a newer, Terraform-only option (OpenTofu doesn't have it); check HashiCorp's Terraform Policy docs for the policy file syntax before adopting it. OPA/Conftest (above) stays the portable choice.

```sentinel
import "tfplan/v2" as tfplan

buckets = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_s3_bucket" and rc.change.actions contains "create"
}

main = rule {
  all buckets as _, b { "owner" in keys(b.change.after.tags_all) }
}
```

### Cost policies: Infracost

**Infracost** estimates the monthly cost of a plan from cloud price lists, and can comment cost diffs on PRs and enforce limits:

```bash
infracost breakdown --path plan.json
```

```
 Name                                          Monthly Qty  Unit    Monthly Cost
 aws_instance.web
 ├─ Instance usage (Linux/UNIX, on-demand, t3.micro)   730  hours         $8.18
 └─ root_block_device
    └─ Storage (general purpose SSD, gp3)                8  GB            $0.73
 OVERALL TOTAL                                                             $8.91
```

(Prices illustrative.) Its JSON output can be fed into OPA ("deny if the monthly increase > $500").

---

## Why it exists

Rules written in a wiki get forgotten. Reviewers miss things in big plans. Some rules are about **context** that scanners can't know: your tag standard, allowed regions, instance sizes per environment, which resources must never be deleted by a pipeline. Policy as code makes these rules **automatic, consistent, versioned and testable**, and gives clear error messages that teach people the rules.

---

## Analogy

**A bank's automatic rule checker.** The rulebook ("transfers over £10,000 need two signatures", "no transfers to blocked countries") is turned into software that checks **every** transfer request **before** it's executed. It explains why a request is refused. Some rules are warnings, some are hard stops, and a manager can override soft ones. The rulebook itself is reviewed and tested before changes (policy tests).

---

## How it works inside

```
terraform show -json tfplan  → plan.json  (the input document)
conftest test plan.json
   loads ./policy/*.rego
   for each rule named deny/violation/warn in package main:
       evaluate with input = plan.json → a set of messages
   any deny/violation message → FAIL, exit code 1
   warn messages              → WARN, exit code 0 (unless --fail-on-warn)
```

Rego is evaluated like a database query: each rule body is a set of conditions; Rego finds every combination of values (`some rc in ...`) that makes them all true, and collects the results.

---

## Code / commands

Use a plan from topic 31 or 33 (planning is free; you don't need to apply). Install Conftest (`github.com/open-policy-agent/conftest` releases) and optionally OPA (`openpolicyagent.org`).

### Policies

`policy/tags.rego`:

```rego
package main

required_tags := {"owner", "project"}

deny contains msg if {
  some rc in input.resource_changes
  rc.mode == "managed"
  some action in rc.change.actions
  action in {"create", "update"}
  tags := object.get(rc.change.after, "tags_all", null)
  tags != null                                   # only taggable resources have tags_all
  some t in required_tags
  not tags[t]
  msg := sprintf("%s: missing required tag %q", [rc.address, t])
}
```

`policy/instances.rego`:

```rego
package main

allowed_instance_types := {"t3.micro", "t3.small", "t4g.micro", "t4g.small"}

deny contains msg if {
  some rc in input.resource_changes
  rc.type == "aws_instance"
  "create" in rc.change.actions
  itype := rc.change.after.instance_type
  not itype in allowed_instance_types
  msg := sprintf("%s: instance type %s is not allowed (allowed: %v)", [rc.address, itype, allowed_instance_types])
}

warn contains msg if {
  some rc in input.resource_changes
  rc.type == "aws_instance"
  rc.change.after.metadata_options[0].http_tokens != "required"
  msg := sprintf("%s: IMDSv2 should be required", [rc.address])
}
```

`policy/protect.rego`:

```rego
package main

protected_types := {"aws_db_instance", "aws_rds_cluster", "aws_s3_bucket", "aws_kms_key"}

deny contains msg if {
  some rc in input.resource_changes
  rc.type in protected_types
  "delete" in rc.change.actions                 # plain delete OR the delete half of a replace
  msg := sprintf("%s: deleting/replacing a %s needs a manual, approved change", [rc.address, rc.type])
}
```

### Run it

```bash
cd ~/tf-lab/31-first-aws
terraform plan -out=tfplan -var="instance_type=m5.large"
terraform show -json tfplan > plan.json
conftest test plan.json -p ~/tf-lab/54-policy/policy
```

```
WARN - plan.json - main - aws_instance.lab: IMDSv2 should be required
FAIL - plan.json - main - aws_instance.lab: instance type m5.large is not allowed (allowed: {"t3.micro", "t3.small", "t4g.micro", "t4g.small"})
FAIL - plan.json - main - aws_s3_bucket.lab: missing required tag "owner"
FAIL - plan.json - main - aws_instance.lab: missing required tag "owner"

12 tests, 8 passed, 1 warning, 3 failures, 0 exceptions
```

(The IMDSv2 warning appears only if you removed `metadata_options`; counts vary.) Fix: `instance_type` back to `t3.micro`, and add `owner = "you"` to the provider's `default_tags` (topic 30). Re-plan and re-test:

```
12 tests, 12 passed, 0 warnings, 0 failures, 0 exceptions
```

### Test the policies themselves

`policy/protect_test.rego`:

```rego
package main

test_deny_db_delete if {
  count(deny) > 0 with input as {"resource_changes": [
    {"address": "aws_db_instance.main", "type": "aws_db_instance", "mode": "managed",
     "change": {"actions": ["delete"], "after": null}}
  ]}
}

test_allow_db_update if {
  count(deny) == 0 with input as {"resource_changes": [
    {"address": "aws_db_instance.main", "type": "aws_db_instance", "mode": "managed",
     "change": {"actions": ["update"], "after": {"tags_all": {"owner": "a", "project": "b"}}}}
  ]}
}
```

```bash
opa test policy/ -v        # or: conftest verify -p policy/
```

```
policy/protect_test.rego:
data.main.test_deny_db_delete: PASS (1.2ms)
data.main.test_allow_db_update: PASS (0.9ms)
--------------------------------------------------------------------------------
PASS: 2/2
```

Policies are code: they get tests and PR reviews.

### In a pipeline

```bash
terraform plan -out=tfplan -input=false
terraform show -json tfplan > plan.json
conftest test plan.json -p policy/ --output github      # annotations in GitHub Actions
```

Exceptions: give the protected-delete rule an escape hatch that needs a **human** (e.g. a separate pipeline with an extra approval, or Conftest's `exception` rules keyed to a reviewed list), never a flag anyone can set.

---

## Hands-on lab

1. Write the three policies and run them against a topic 31 plan with a disallowed instance type and missing tags. Fix the configuration until all pass.
2. Plan a replacement of the S3 bucket (`-replace=aws_s3_bucket.lab`), test: the protect rule fires.
3. Write and run the policy unit tests.
4. Add a rule: "no security group rule may allow port 22 from `0.0.0.0/0`" (look for `aws_vpc_security_group_ingress_rule` with `from_port <= 22 <= to_port` and `cidr_ipv4 == "0.0.0.0/0"`). Test it against topic 32's plan.
5. (Optional) Run `infracost breakdown --path plan.json` with a free Infracost API key.

---

## Where you'll use it

- **Governance at scale**: tagging, regions, sizes, encryption, approved module sources, provider versions.
- **Safety rails for automation**: no automated deletes of databases/buckets/keys.
- **HCP Terraform/Enterprise**: Sentinel, OPA or Terraform Policy sets attached to workspaces.
- **Cost control**: Infracost comments and budget policies on PRs.
- **Kubernetes**: the same OPA/Rego skills apply to Gatekeeper admission policies.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| Rule never fires | Wrong field path; value unknown at plan time (absent from `after`); wrong package name. | Inspect `plan.json` with `jq`; check `after_unknown`; `package main` for Conftest defaults. |
| `rego_parse_error: ... if keyword is required` | Old Rego syntax with OPA 1.x. | Use `if`/`contains` (OPA 1.0 syntax). |
| False positives on resources without tags | Not every resource supports tags. | Check `tags_all` exists before requiring tags (as above). |
| Policies pass locally, fail in CI | Different plan (variables, env) or Conftest version. | Same inputs; pin tool versions. |
| People bypass policies | They're advisory or skippable. | Enforce in CI as required checks; platform enforcement levels. |
| Unreadable failures | Messages like "policy failed". | Include the address, the offending value and the rule's intent. |

---

## Quick recap

- Policy as code = **your** rules, as versioned and tested code, checked against the **plan JSON** before apply.
- OPA/Rego with **Conftest**: `deny`/`warn` rules over `input.resource_changes[]` (`type`, `address`, `change.actions`, `change.after`).
- Test policies with `opa test`; give clear messages; enforce in CI.
- HashiCorp tooling: Sentinel and OPA on HCP Terraform (advisory / soft- / hard-mandatory), and Terraform Policy in the CLI (`-policies`, GA in 1.17). Infracost adds cost checks.

### Interview questions

1. **Q: What is policy as code for Terraform?**
   A: Organisation rules written as code (e.g. Rego or Sentinel) and evaluated automatically against Terraform plans before apply, such as required tags, allowed instance types or no deletion of databases, failing the pipeline with a clear message when violated.
2. **Q: How would you enforce a tagging standard?**
   A: Default tags in the provider for the common ones, plus a policy over the plan JSON that denies any created or updated taggable resource whose `tags_all` lacks the required keys, run with Conftest in CI (or Sentinel/OPA in HCP Terraform).
3. **Q: How do you stop an automated pipeline from deleting a production database?**
   A: `prevent_destroy` and provider deletion protection on the resource, plus a policy that denies any plan containing a `delete` action on database types, with exceptions only through a separate human-approved path.
