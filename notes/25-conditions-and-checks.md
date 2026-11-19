# 25 · Custom Conditions and `check` Blocks

> **Level:** ★★ intermediate
>
> **You will learn:** the four ways to make Terraform **verify assumptions** and fail (or warn) with **your own clear message**: variable `validation` (recap), **`precondition`** (check inputs before a resource/data source/output is planned), **`postcondition`** (check the result, using `self`), and **`check` blocks** (continuous health assertions that only **warn**, with an optional scoped data source). You'll learn when each runs, which ones block an apply, and how to choose.
>
> **You need:** topics 14 (validation), 15 (outputs), 19 (data sources), 24 (`lifecycle`).
>
> **Part 2 · Making configuration flexible** · Previous: [24 · The `lifecycle` meta-argument](24-lifecycle.md) · Next: [26 · Docker recap and the Docker provider](26-docker-provider.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** You can attach checks with your own error messages:

| Check | Where | When it runs | Failure = |
|---|---|---|---|
| `validation` | `variable` block | Before planning | **Error** |
| `precondition` | `lifecycle` of a resource/data source, or an `output` | Before that object is planned | **Error** |
| `postcondition` | `lifecycle` of a resource/data source | After it's planned/read/applied | **Error** |
| `check` block | Top level | End of every plan and apply | **Warning** only |

```hcl
data "http" "app" {
  url = "http://localhost:8080/health"
  lifecycle {
    postcondition {
      condition     = self.status_code == 200                # self = this object
      error_message = "App health endpoint returned ${self.status_code}."
    }
  }
}

check "website_up" {                                         # warns, never blocks
  data "http" "site" { url = "https://example.com" }
  assert {
    condition     = data.http.site.status_code == 200
    error_message = "example.com is down."
  }
}
```

**Picture it.** A pilot's checklist: **preconditions** are pre-flight checks (don't take off if the fuel is low), **postconditions** are post-landing checks (is the gear down and locked?), and a **`check` block** is the dashboard warning light: it shows a problem, but the plane keeps flying.

**The one thing to know.** Use `precondition`/`postcondition` when continuing would be **wrong** (stop the run). Use `check` for "**is everything healthy?**" signals that shouldn't block deployments.

**Try it now:** add the `check` block above to any lab and run `terraform plan`. Change the URL to a broken one and see a **warning**, not an error.

**You've got it if you can:** choose the right kind of check for "the AMI must be x86_64", "the subnet must be private", and "the website should return 200".

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### `precondition`

```hcl
resource "local_file" "deploy" {
  # ...
  lifecycle {
    precondition {
      condition     = var.environment != "prod" || var.replicas >= 2
      error_message = "prod needs at least 2 replicas."
    }
  }
}
```

- Checks **assumptions about the inputs** of this object: variables, other resources, data sources.
- Runs **before** Terraform plans this object. If it fails, nothing is created or changed.
- Unlike a variable `validation`, it sits **next to the resource that cares**, and can refer to anything (other resources, data sources), not only the variable.
- Also allowed in `output` blocks: check something before exposing an output (useful for module outputs).

### `postcondition`

```hcl
data "local_file" "config" {
  filename = "${path.module}/config.json"
  lifecycle {
    postcondition {
      condition     = can(jsondecode(self.content))
      error_message = "config.json is not valid JSON."
    }
  }
}
```

- Checks **the object itself** after Terraform knows its values: after reading a data source, after planning a resource (if values are known), or after applying it.
- **`self`** refers to the object's own attributes. (Inside a resource you can't reference the resource by its own address.)
- If the values are `(known after apply)` during plan, the check waits until apply.

### `check` blocks (Terraform 1.5+)

```hcl
check "<NAME>" {
  data "TYPE" "NAME" { ... }        # optional "scoped" data source: only usable inside this check

  assert {                          # one or more
    condition     = <bool expression>
    error_message = "..."
  }
}
```

- Run at the **end** of every `plan` and `apply`, after everything else.
- A failing assertion produces a **warning**. The plan/apply still succeeds.
- A **scoped data source** inside the check is read as part of the check. If **it** fails (e.g. the URL doesn't respond), that's also just a warning, unlike a normal data source, which would fail the whole plan.
- Can reference anything in the configuration.
- In HCP Terraform (topic 50), checks also run on a schedule for continuous health monitoring.

### Choosing

| Question | Use |
|---|---|
| Is this input value acceptable on its own? | `validation` on the variable |
| Are this resource's inputs (from several places) consistent before I build it? | `precondition` |
| Did the data source/resource come back the way I assumed? | `postcondition` |
| Is the system healthy overall? Should a human be told? | `check` block |

---

## Why it exists

Without them, wrong assumptions surface as **confusing provider errors** halfway through an apply ("InvalidParameterCombination"), or worse, as **successful** applies that build the wrong thing (a database in a public subnet, an ARM image on an x86 instance type). Conditions:

- **Fail early**, at plan time where possible, before anything changes.
- Produce **your** message, pointing at the real cause.
- **Document** assumptions in code: the next reader sees "this subnet must be private" right on the resource.

`check` blocks exist because some things should be **visible** but not **blocking**: a health endpoint down during a deploy shouldn't stop you from deploying the fix.

---

## Analogy

**A pilot's checklists.**

- **Variable validation**: the booking system refuses a ticket with an impossible date.
- **Precondition**: pre-flight: "fuel ≥ required for this route" (needs both the fuel gauge and the route, two different inputs). Fails → don't take off.
- **Postcondition**: after landing gear is lowered: "three green lights". `self` = this plane's own gauges. Fails → go around.
- **`check`**: a cockpit warning light: "cabin temperature high". It's noted and reported, but the flight continues.

---

## How it works inside

```
plan:
  1. variable validations                                  → error stops everything
  2. graph walk; for each resource/data/output node:
       preconditions evaluated                             → error stops the plan
       (data source read / resource planned)
       postconditions evaluated if values known            → error stops the plan
                                       (unknown → deferred to apply)
  3. after the walk: check blocks evaluated (scoped data sources read here)
                                                           → WARNINGS
apply:
  same, plus postconditions deferred from plan are evaluated right after
  each object is applied → error stops the apply (objects already applied stay applied)
  check blocks evaluated again at the end                  → WARNINGS
```

Conditions are evaluated with the same rules as any expression: `can()` and `try()` are useful to guard against errors inside the condition itself.

---

## Code / commands

`~/tf-lab/25-checks/versions.tf` with `local` and `http`.

```hcl
variable "environment" {
  type    = string
  default = "dev"
  validation {
    condition     = contains(["dev", "prod"], var.environment)
    error_message = "environment must be dev or prod."
  }
}

variable "replicas" {
  type    = number
  default = 1
}

# A config file maintained by another team
data "local_file" "settings" {
  filename = "${path.module}/settings.json"
  lifecycle {
    postcondition {
      condition     = can(jsondecode(self.content).region)
      error_message = "settings.json must be valid JSON with a \"region\" key."
    }
  }
}

locals {
  settings = jsondecode(data.local_file.settings.content)
}

resource "local_file" "deploy" {
  filename = "${path.module}/deploy-${var.environment}.txt"
  content  = "replicas=${var.replicas} region=${local.settings.region}\n"

  lifecycle {
    precondition {
      condition     = var.environment != "prod" || var.replicas >= 2
      error_message = "prod needs at least 2 replicas (got ${var.replicas})."
    }
    postcondition {
      condition     = length(self.content) < 200
      error_message = "deploy file unexpectedly large."
    }
  }
}

output "deploy_file" {
  value = local_file.deploy.filename
  precondition {
    condition     = local.settings.region != ""
    error_message = "Refusing to output a deploy file without a region."
  }
}

check "public_site" {
  data "http" "site" {
    url = "https://example.com"
  }
  assert {
    condition     = data.http.site.status_code == 200
    error_message = "example.com returned ${data.http.site.status_code}."
  }
}
```

Create the input file:

```bash
echo '{"region": "ap-south-1"}' > settings.json
terraform init && terraform apply -auto-approve
```

### A failing precondition

```
$ terraform plan -var="environment=prod" -var="replicas=1"
╷
│ Error: Resource precondition failed
│
│   on main.tf line 33, in resource "local_file" "deploy":
│   33:       condition     = var.environment != "prod" || var.replicas >= 2
│     ├────────────────
│     │ var.environment is "prod"
│     │ var.replicas is 1
│
│ prod needs at least 2 replicas (got 1).
╵
```

- Terraform shows the condition **and the values** that made it false, then your message.
- Nothing was planned.

### A failing postcondition on a data source

```bash
echo '{"zone": "a"}' > settings.json
terraform plan
```

```
╷
│ Error: Resource postcondition failed
│
│   on main.tf line 21, in data "local_file" "settings":
│   21:       condition     = can(jsondecode(self.content).region)
│     ├────────────────
│     │ self.content is "{\"zone\": \"a\"}\n"
│
│ settings.json must be valid JSON with a "region" key.
╵
```

Without it, you'd get a vaguer `Unsupported attribute ... This object does not have an attribute named "region"` from the `locals` block, pointing at the wrong place. Restore the file.

### A failing `check`: a warning

Change the check's URL to `https://example.com/does-not-exist`:

```
$ terraform plan
data.local_file.settings: Reading...
...
No changes. Your infrastructure matches the configuration.
╷
│ Warning: Check block assertion failed
│
│   on main.tf line 54, in check "public_site":
│   54:     condition     = data.http.site.status_code == 200
│     ├────────────────
│     │ data.http.site.status_code is 404
│
│ example.com returned 404.
╵
```

- `No changes.` + `Warning:`: the run **succeeds**. The exit code is 0.
- Change the URL to a host that doesn't exist (`https://nope.invalid`): the scoped data source itself fails, and you get a **warning** (`Error making request ... no such host` reported as a warning in the check), still not an error.

---

## Hands-on lab

1. Build the project. Apply with defaults.
2. Trigger each failure: invalid `environment` (validation), `prod` with 1 replica (precondition), broken `settings.json` (postcondition), bad URL (check). For each, note: error or warning? Did anything change?
3. Run `terraform plan -detailed-exitcode; echo $?` with only the check failing. What's the exit code? (`0` if no changes: checks don't affect it.)
4. Move the "prod needs 2 replicas" rule into the `replicas` variable's `validation` (since 1.9, validation can reference `var.environment`). Which place reads better? (Both are fine; put rules where a reader will look.)
5. Add a `check` that warns if `deploy-${var.environment}.txt` contains `replicas=1` in prod.
6. `terraform destroy`.

---

## Where you'll use it

- **Preconditions** on AWS resources: "the AMI architecture matches the instance type", "the subnet is private", "the KMS key is in the same region".
- **Postconditions** on data sources: "the looked-up VPC has DNS support enabled", "exactly 3 AZs were found", "the AMI is less than 90 days old".
- **Output preconditions** in modules: guarantee to callers that outputs are valid.
- **Check blocks**: "the load balancer answers 200", "the TLS certificate expires in more than 30 days", "the bucket is not public". Shown on every plan; scheduled in HCP Terraform.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `Resource precondition failed` | Your assumption about the inputs was false. | Read the values shown; fix the inputs (or the rule). |
| `Resource postcondition failed` during **apply**, after some resources were created | The values were unknown at plan time; the check ran after apply. | The object exists now; fix the cause and re-apply (or replace it). Prefer conditions that can run at plan time. |
| `Invalid reference ... self` | `self` used outside a postcondition (or in a precondition). | `self` is only for postconditions. |
| Check never shows | You expected an error; checks only warn. | Use pre/postconditions to block. |
| `Error in function call` inside a condition | The condition itself errors (e.g. `jsondecode` of bad JSON). | Wrap with `can()` / `try()`. |
| Condition messages unhelpful | Generic text like "invalid input". | Say what's required and include the value: `"got ${var.x}"`. |

---

## Quick recap

- `validation` (variables), `precondition` (before an object), `postcondition` (after, with `self`) → **errors** that stop the run.
- `check` blocks → **warnings** at the end of every plan/apply; can include a scoped data source.
- Conditions show the values that failed plus your message: write messages that say what's required.
- Use blocking conditions when continuing would be wrong; `check` for health signals.

### Interview questions

1. **Q: Precondition vs postcondition?**
   A: A precondition checks assumptions about an object's inputs before Terraform plans it. A postcondition checks the object's own resulting values (via `self`) after it's read, planned or applied. Both fail the run with your message.
2. **Q: What's a `check` block and how is it different?**
   A: A top-level block with assertions (and optionally a scoped data source) evaluated at the end of every plan and apply. Failures are warnings, not errors, so it reports health problems without blocking changes.
3. **Q: Where would you validate that a variable is a valid CIDR?**
   A: In the variable's `validation` block, e.g. `condition = can(cidrhost(var.cidr, 0))`, so bad input fails before planning.
