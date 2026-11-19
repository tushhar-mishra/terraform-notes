# 64 · Debugging Terraform

> **Level:** ★★ intermediate
>
> **You will learn:** a repeatable **debugging method** (symptom → phase → tool), how to read **any** Terraform error, the debugging tools: **`TF_LOG`** levels and split core/provider logs, `TF_LOG_PATH`, JSON logs, **`terraform console`**, `graph`, `providers`, `validate -json`, `show -json`; cloud-side tools (**CloudTrail**, `sts get-caller-identity`, `sts decode-authorization-message`); what a **crash** (`crash.log`) is and how to report provider bugs; and five broken labs to practise on.
>
> **You need:** topic 13 (the phases), 18 (console), 20 (graph), 30 (credentials), 34.
>
> **Part 10 · Running it in production** · Previous: [63 · Terraform at scale](63-terraform-at-scale.md) · Next: [65 · Error catalogue](65-error-catalogue.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The method:**

1. **Read the whole error**: summary, `on file.tf line N`, the values shown, the **provider's original error** at the bottom.
2. **Which phase failed?** (topic 13): config load · init · backend/lock · provider configure · refresh · graph · plan · apply.
3. **Pick the tool for that phase:**

| Phase | Tool |
|---|---|
| Config / expressions | `terraform validate`, `terraform console` |
| Init / providers | `terraform providers`, `terraform version`, the lock file |
| Backend / lock | `terraform state pull`, backend IAM, lock info (topic 36) |
| Credentials / permissions | `aws sts get-caller-identity`, `env | grep AWS_`, `aws sts decode-authorization-message` |
| Refresh / plan / apply (API) | `TF_LOG=DEBUG`, CloudTrail, the provider's docs and GitHub issues |
| Ordering / cycles | `terraform graph` |
| Crash | `crash.log`, report a bug |

```bash
TF_LOG=DEBUG TF_LOG_PATH=./tf-debug.log terraform plan
grep -nE 'ERROR|error|HTTP/1.1 [45]' tf-debug.log | head
```

**Picture it.** A doctor's diagnosis: listen to the symptom (error), find which body system is involved (phase), then order the right test (tool). Don't order every test at once.

**The one thing to know.** Most errors are fully explained in the message itself, especially the **"Original Error"** / API error at the bottom. Read it before reaching for debug logs.

**Try it now:** do broken lab 1 below.

**You've got it if you can:** debug an `AccessDenied` from the error to the exact missing IAM action.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Anatomy of an error

```
╷
│ Error: creating EC2 Instance: operation error EC2: RunInstances, https response error
│ StatusCode: 400, RequestID: 6c1e..., api error InvalidParameterValue: Value (t3.micor)
│ for parameter instanceType is invalid.
│
│   with aws_instance.web,
│   on compute.tf line 10, in resource "aws_instance" "web":
│   10: resource "aws_instance" "web" {
│
╵
```

| Part | Tells you |
|---|---|
| `Error: creating EC2 Instance` | The provider's summary: what it was doing |
| `operation error EC2: RunInstances` | The exact **API call** |
| `StatusCode: 400`, `RequestID` | HTTP status (topic 05); the request ID to find it in CloudTrail/support |
| `api error InvalidParameterValue: ...` | The **cloud's own message**: usually the actual answer |
| `with aws_instance.web, on compute.tf line 10` | Which resource and where in your code |

For config errors, the box shows the **expression and its values** (`├──── │ var.x is "..."`), which is often all you need.

### `TF_LOG`

| Variable | Meaning |
|---|---|
| `TF_LOG=TRACE\|DEBUG\|INFO\|WARN\|ERROR` | Log level for core **and** providers (TRACE = everything, including gRPC calls) |
| `TF_LOG=JSON` | TRACE level, in JSON lines (for tools) |
| `TF_LOG_CORE=...` / `TF_LOG_PROVIDER=...` | Separate levels: e.g. `TF_LOG_PROVIDER=DEBUG` alone to see only API traffic |
| `TF_LOG_PATH=file` | Write logs to a file (they go to stderr otherwise) |

The AWS provider at `DEBUG` logs each HTTP request/response (with credentials redacted, but **bodies can contain secrets**: don't paste logs publicly without checking).

### Other tools

| Tool | Use |
|---|---|
| `terraform console` | Evaluate any expression with real variables and state (topic 18) |
| `terraform validate -json` | Machine-readable diagnostics (editor integrations, CI annotations) |
| `terraform plan -json` / `show -json tfplan` | Machine-readable plans; `jq` for "which resources change and why" |
| `terraform graph` | Dependency problems, cycles (topic 20) |
| `terraform providers` | Which provider versions are required by whom (version conflicts) |
| `terraform version` | CLI + provider versions (always include in bug reports) |
| `terraform state show/list/pull` | What Terraform believes (topic 34) |
| `aws sts get-caller-identity` | Which identity is actually used |
| `aws sts decode-authorization-message --encoded-message ...` | Decodes AWS's encoded "Encoded authorization failure message" into the exact denied action/resource/condition |
| **CloudTrail** (Event history) | Every AWS API call: who, what, error code, request parameters |

### Crashes

A **crash** is a bug (a "panic" in Go) in Terraform or a provider: the process dies with `Terraform crashed!` or `The plugin ... crashed!` and writes `crash.log` (or the provider's panic output). It's not your config's fault (even if your config triggered it). Steps:

1. Note `terraform version`, the provider version, and the stack trace's first lines.
2. Search the provider's / Terraform's GitHub issues for the panic message.
3. Try the latest patch version of the provider.
4. Report with a **minimal** config reproducing it, plus the trace (redacted).
5. Check state afterwards (a crash mid-apply = partial apply, topic 13; maybe a stale lock, topic 36).

---

## Why it exists

Terraform sits between your code, its state, provider plugins and remote APIs; a failure can come from any of them. Without a method, people guess ("re-run it", "delete `.terraform`", "force-unlock") and sometimes make things worse. A phase-based approach finds the cause quickly and safely.

---

## Analogy

**A doctor's diagnosis.** The symptom (error text) points to a body system (phase). The doctor orders the right test for that system (tool): a blood test, not an MRI, for most things. Detailed scans (`TRACE` logs) are for when simpler tests don't explain it. And a hospital error (crash) gets reported to the hospital, not treated as the patient's fault.

---

## How it works inside

`TF_LOG` turns on Go's `hclog` logging in core; core passes the level to each provider plugin it starts, and provider logs come back over the plugin channel, prefixed with the provider's binary name:

```
2026-11-19T18:01:02.123+0530 [DEBUG] provider.terraform-provider-aws_v6.73.0_x5: HTTP Request Sent: aws.region=ap-south-1 aws.service=EC2 aws.operation=RunInstances http.method=POST ...
2026-11-19T18:01:02.456+0530 [DEBUG] provider.terraform-provider-aws_v6.73.0_x5: HTTP Response Received: ... http.status_code=400 ...
```

---

## Code / commands

### The five most useful one-liners

```bash
# 1. Who am I, really?
env | grep -E '^(AWS_|TF_)'; aws sts get-caller-identity

# 2. What API call failed? (DEBUG to a file, then search)
TF_LOG_PROVIDER=DEBUG TF_LOG_PATH=dbg.log terraform apply; grep -nE 'operation=|status_code=[45]' dbg.log | tail

# 3. Why was it denied? (when AWS returns an encoded message)
aws sts decode-authorization-message --encoded-message "<long string>" --query DecodedMessage --output text | jq .

# 4. What value does this expression really have?
echo 'module.web.url' | terraform console

# 5. What will change, as data?
terraform plan -out=tfplan >/dev/null && terraform show -json tfplan \
  | jq -r '.resource_changes[] | select(.change.actions != ["no-op"]) | "\(.change.actions|join(",")) \(.address)"'
```

### Broken labs (do them in order)

Use a scratch copy of topic 31 (S3 + EC2) or topic 33.

**Broken lab 1: `AccessDenied`.** Create an IAM user/role that only has `ReadOnlyAccess`, use it as your profile, and `terraform apply`:

```
│ Error: creating S3 Bucket (tf-course-lab-...): operation error S3: CreateBucket,
│ https response error StatusCode: 403, RequestID: ..., api error AccessDenied:
│ User: arn:aws:iam::123456789012:user/readonly is not authorized to perform:
│ s3:CreateBucket on resource: "arn:aws:s3:::tf-course-lab-..." because no
│ identity-based policy allows the s3:CreateBucket action
```

Phase: apply (API). The message names the **identity**, the **action** and the **reason**. Fix: right profile or policy. (CloudTrail shows the same event with `errorCode: AccessDenied`.)

**Broken lab 2: wrong account/region.** Run with a profile for another region set via `AWS_REGION=us-east-1` and a provider without `region`:

```
│ Error: reading S3 Bucket ... : couldn't find resource
```

Or a plan that wants to **create everything** (topic 34). Phase: refresh. Tools: `env | grep AWS_`, `get-caller-identity`, `allowed_account_ids`.

**Broken lab 3: an expression bug.** In a `for_each`, use `toset(var.names)` where `var.names` is a map:

```
│ Error: Invalid function argument
│
│   on main.tf line 12, in resource "local_file" "x":
│   12:   for_each = toset(var.names)
│     ├────────────────
│     │ var.names is map of string with 2 elements
│
│ Invalid value for "v" parameter: cannot convert map of string to set of any single type.
```

Phase: plan (expressions). Tool: `terraform console`, `type(var.names)`, then fix (`keys(var.names)` or use the map directly).

**Broken lab 4: a version conflict.** A local module requiring `aws < 6.0` used from a root with `~> 6.0`:

```
│ Error: Failed to query available provider packages
│
│ Could not retrieve the list of available versions for provider hashicorp/aws:
│ no available releases match the given constraints >= 4.0.0, < 6.0.0, ~> 6.0
```

Phase: init. Tool: `terraform providers` (shows which module requires what). Fix the module's constraint.

**Broken lab 5: a slow/hanging apply.** Create an instance with a `user_data` that never finishes, behind a `check` block: the apply finishes but the check warns forever. Or set `-parallelism=50` on many resources in a small account and watch for `Throttling`. Tools: `TF_LOG=DEBUG` (retries), the console (instance status, system log), SSM (cloud-init logs, topic 33).

### Reporting a provider bug (template)

~~~markdown
### Terraform and provider versions
Terraform v1.17.2 on linux_amd64
+ provider registry.terraform.io/hashicorp/aws v6.73.0

### Minimal configuration
```hcl
resource "aws_xxx" "repro" { ... }
```

### Expected vs actual
Expected: in-place update. Actual: "Provider produced inconsistent final plan".

### Debug output
Gist link (TF_LOG=DEBUG, secrets removed)

### Steps
1. terraform apply
2. change X
3. terraform apply → error
~~~

---

## Hands-on lab

1. Do broken labs 1–5. For each, write: symptom → phase → tool → cause → fix.
2. Find one of your own past errors in CloudTrail (Event history, filter by `Error code`).
3. Run a plan with `TF_LOG=JSON TF_LOG_PATH=log.json` and use `jq` to count log lines per provider.
4. Practise `decode-authorization-message` if you can trigger an encoded failure (e.g. starting an instance with a denied `ec2:RunInstances` condition).

---

## Where you'll use it

- **Daily**: failed pipeline runs, red PR checks, "why does the plan want to replace this?"
- **On-call**: permission errors after IAM changes, throttling during big applies, stale locks after crashes.
- **Interviews**: "How do you debug a Terraform failure?": describe the method, not just `TF_LOG`.

---

## When it breaks

(The debugging tools themselves.)

| Symptom | Cause | Fix |
|---|---|---|
| `TF_LOG` output floods the terminal | Logs go to stderr | `TF_LOG_PATH=file`; filter with `grep`/`jq` |
| No provider logs | `TF_LOG_CORE` set but not `TF_LOG_PROVIDER`, or vice versa | Use `TF_LOG` for both |
| Debug log contains secrets | Request/response bodies | Never share raw logs; redact; delete files after use |
| `console` hangs | State lock held by another process | Close other runs (topic 18) |
| CloudTrail shows nothing | Wrong region, or data events (e.g. S3 object reads) not enabled | Check the region; data events are off by default |

---

## Quick recap

- Method: **read the whole error → identify the phase → use that phase's tool**.
- The provider's API error (operation, status code, message) usually contains the answer.
- Tools: `validate`, `console`, `graph`, `providers`, `state`, `show -json`, `TF_LOG` (+ `TF_LOG_PATH`, split core/provider), `sts get-caller-identity`, `decode-authorization-message`, CloudTrail.
- Crashes are bugs: collect versions + trace, check issues, upgrade, report minimally.

### Interview questions

1. **Q: How do you debug a failing Terraform apply?**
   A: Read the full error, including the provider's API error and the resource location; identify the phase (init, credentials, refresh, plan or apply); then use the matching tool: `validate`/`console` for config, `sts get-caller-identity` and IAM/CloudTrail for permissions, `TF_LOG=DEBUG` with `TF_LOG_PATH` for API-level detail, `graph` for ordering. Fix the cause and re-run; Terraform resumes from state.
2. **Q: What do `TF_LOG` and `TF_LOG_PATH` do?**
   A: `TF_LOG` sets the log level (TRACE to ERROR, or JSON) for Terraform core and providers; `TF_LOG_CORE`/`TF_LOG_PROVIDER` set them separately. `TF_LOG_PATH` writes the logs to a file instead of stderr.
3. **Q: You get AccessDenied in CI but not locally. How do you investigate?**
   A: Compare identities (`aws sts get-caller-identity` in both), check which role CI assumed and its policies, look at the denied action and resource in the error or CloudTrail, decode any encoded authorization message, and check for SCPs or permission boundaries that apply only to the CI role.
