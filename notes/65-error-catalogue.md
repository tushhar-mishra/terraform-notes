# 65 · Error Catalogue

> **Level:** ★★ intermediate (reference)
>
> **You will learn:** nothing new. This topic collects **every common Terraform error** from the course in one place, grouped by **phase**, each with its meaning, the fix, and the topic that explains it. Use it with topic 64's method: find the phase, find the message, follow the fix.
>
> **You need:** the topics referenced in each row.
>
> **Part 10 · Running it in production** · Previous: [64 · Debugging Terraform](64-debugging-terraform.md) · Next: [66 · Production break-fix scenarios](66-production-scenarios.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Errors fall into a few families. Recognising the family tells you where to look:

| Family | Looks like | First move |
|---|---|---|
| **Init / versions** | `Inconsistent dependency lock file`, `no available releases match` | `terraform init` / `-upgrade`; `terraform providers` |
| **Config** | `Unsupported argument`, `Missing required argument`, `Invalid reference` | Fix the code at the line shown |
| **Values / types** | `Invalid value for variable`, `Inconsistent conditional result types` | `terraform console` |
| **Unknowns** | `Invalid count/for_each argument ... cannot be determined until apply` | Plan-time-known keys/counts |
| **Graph** | `Cycle: ...` | Remove the circular reference |
| **State / lock** | `Error acquiring the state lock`, `Saved plan is stale`, `Failed to persist state` | Topics 36, 09, 42 |
| **Credentials** | `No valid credential sources`, `ExpiredToken`, `AccessDenied` | `sts get-caller-identity` |
| **API** | `already exists`, `DependencyViolation`, `InvalidParameterValue`, `Throttling` | Read the API message |
| **Replacements** | `must be replaced` where you didn't expect it | Find `# forces replacement` |
| **Provider bugs** | `inconsistent final plan`, `inconsistent result after apply`, `crashed` | Upgrade / report |

**The one thing to know.** Search this page for the **first line** of your error (Ctrl-F).

**Try it now:** pick five errors you met during the course and find them below.

**You've got it if you can:** classify an unfamiliar error into a family in under 10 seconds.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

A lookup table. Each section is one phase from topic 13.

## Why it exists

During an incident you don't want to re-read 60 topics. You want the message, its meaning and the fix, with a pointer for the full story.

## Analogy

The troubleshooting page at the back of an appliance manual: "Light blinks red → filter blocked → clean the filter (page 12)".

## How it works inside

Errors come from four sources: **core** (config, graph, state), the **backend** (storage, locks), the **provider** (validation, planning), and the **remote API** (passed through by the provider, usually with `api error <Code>: <message>`). The source tells you whose documentation to read.

---

## Code / commands: the catalogue

### 1. Init, versions, plugins

| Error (first line) | Meaning | Fix | Topic |
|---|---|---|---|
| `Inconsistent dependency lock file` | Providers in config don't match the lock file | `terraform init` (commit the lock file) | 10 |
| `locked provider ... does not match configured version constraint` | Constraint changed | `terraform init -upgrade` | 10 |
| `no available releases match the given constraints` | Constraints conflict (root vs modules) or don't exist | `terraform providers`; fix constraints | 10, 62 |
| `Could not retrieve the list of available versions for provider hashicorp/docker` | Missing `required_providers` source → wrong namespace guessed | Declare `source = "kreuzwerker/docker"` | 10, 26 |
| `the current package ... doesn't match any of the checksums previously recorded` | Platform hashes missing / package differs | `terraform providers lock -platform=...`; investigate if same platform | 10, 62 |
| `Unsupported Terraform Core version` | CLI version outside `required_version` | Install the right version (tenv) | 06, 62 |
| `Module not installed` / `Module source has changed` | New/changed module since init | `terraform init` | 43, 45 |
| `Failed to download module ... Permission denied (publickey)` | No Git access | Deploy key/token; URL scheme | 45 |
| `Failed to query available provider packages ... could not connect` | No network to the registry | Proxy/DNS; provider mirror | 06, 62 |

### 2. Configuration (syntax and schema)

| Error | Meaning | Fix | Topic |
|---|---|---|---|
| `Unsupported argument` (`Did you mean ...?`) | Typo or argument not in this provider version | Fix spelling; check docs for your version | 08 |
| `Missing required argument` | Required field not set | Add it | 08 |
| `Unsupported block type` | Block vs argument confusion (`tags {}` vs `tags = {}`) or removed block | Check the schema | 08, 31 |
| `Duplicate resource "x" "y" configuration` | Same address twice | Rename/delete | 08 |
| `Reference to undeclared resource / input variable / local value` | Typo or wrong scope | Fix the name; across modules use outputs | 11, 16, 43 |
| `Unsupported attribute` | Attribute doesn't exist (or reaching into a module) | Check attribute reference; add a module output | 11, 43 |
| `Invalid reference` | Malformed address (e.g. name starting with a digit) | Fix identifier | 11 |
| `Variables not allowed` | Variable in a backend block (or other literal-only place) | Partial backend config | 35 |
| `Invalid lifecycle` / variables in `lifecycle` | Lifecycle needs literals | Literals; `deletion_protection` instead | 24 |
| `Module is incompatible with count, for_each, and depends_on` | Module contains provider blocks | Pass providers from the root | 44 |
| `Provider configuration not present` | Resources in state need a provider config that was removed | Restore the provider block until resources are gone | 10, 44 |

### 3. Variables and values

| Error | Meaning | Fix | Topic |
|---|---|---|---|
| `No value for required variable` | Not set anywhere (CI with `-input=false`) | tfvars/`-var`/`TF_VAR_` | 14 |
| `Invalid value for variable` | Your `validation` failed | Fix the value | 14, 25 |
| `Invalid value for input variable ... a number is required` | Type conversion failed | Correct type; quote complex CLI values | 14, 17 |
| `Value for undeclared variable` | tfvars sets a name with no `variable` block | Fix/delete | 14 |
| `Output refers to sensitive values` | Root output includes sensitive data | `sensitive = true` | 15 |
| `Inconsistent conditional result types` | `cond ? a : b` with different types | Same type on both sides | 17 |
| `Invalid index` / `The given key does not identify an element` | Out of range / missing key | `try()`, `lookup()`, `one()` | 17, 18, 21 |
| `Error in function call` | Bad function argument | Test in `terraform console` | 18 |
| `Invalid use of ephemeral value` | Ephemeral value in a normal argument | Write-only argument / provider config | 51 |
| `Duplicate object key` | `for` expression produced the same key twice | Unique keys or `...` grouping | 23 |

### 4. Unknown values, count, for_each

| Error | Meaning | Fix | Topic |
|---|---|---|---|
| `Invalid count argument ... cannot be determined until apply` | `count` depends on an apply-time value | Use plan-time-known values (or a one-off `-target`) | 21, 63 |
| `Invalid for_each argument ... cannot be determined until apply` | Keys unknown | Static keys, unknown values | 22 |
| `the "for_each" argument must be a map, or set of strings` | Passed a list | `toset()` or build a map | 22 |
| `Sensitive values ... cannot be used as for_each arguments` | Secret in keys | Non-sensitive keys | 22 |
| `Missing resource instance key` | Referenced a counted/for_each resource without an index/key | `x[0]`, `x["k"]`, `[*]`, `one()` | 21, 22 |

### 5. Graph

| Error | Meaning | Fix | Topic |
|---|---|---|---|
| `Cycle: a, b` | Circular dependency | Remove one reference; separate link resources (SG rules) | 20, 32 |
| `Invalid depends_on reference` | Attribute instead of object in `depends_on` | `[aws_x.y]` | 20 |

### 6. Backend, state, locks

| Error | Meaning | Fix | Topic |
|---|---|---|---|
| `Error acquiring the state lock` | Another run holds the lock (or stale) | Wait / `-lock-timeout` / checklist + `force-unlock` | 36 |
| `Backend configuration changed` | Backend settings differ from `.terraform/` | `init -migrate-state` or `-reconfigure` | 35 |
| `S3 bucket "x" does not exist` | Bootstrap missing/typo | Create bucket; fix name/region | 35 |
| `Saved plan is stale` | State changed after plan | Re-plan, re-review | 09, 56 |
| `Failed to persist state to backend` / `errored.tfstate` | Upload failed after apply | Fix access; `state push errored.tfstate` | 42 |
| `state snapshot was created by Terraform vX, which is newer` | Newer CLI wrote state | Upgrade CLI | 34, 62 |
| `Resource instance managed by newer provider version` | Newer provider wrote state | Upgrade provider | 34, 62 |
| `cannot import state with lineage ... over unrelated state` | Pushing a different state | Wrong file/key: stop | 42 |
| `cannot import state with serial ... over newer state` | Pushing an older state | Restore via S3 versions or `-force` knowingly | 42 |
| `Workspace is not empty` | Deleting a workspace that tracks resources | Destroy first | 48 |
| Plan wants to **create everything** | Empty/wrong state (folder, key, workspace, lost) | Don't apply; find state; restore/import | 34, 42 |

### 7. Credentials and permissions

| Error | Meaning | Fix | Topic |
|---|---|---|---|
| `No valid credential sources found` | Nothing in the credential chain | `AWS_PROFILE`, `aws sso login`, CI role | 29, 30 |
| `The SSO session ... has expired` / `ExpiredToken` | Temporary credentials expired | Re-login / re-assume | 29 |
| `InvalidClientTokenId` / `SignatureDoesNotMatch` | Bad keys, stale env vars, clock skew | `unset AWS_*`; fix keys; sync clock | 29, 30 |
| `AWS account ID not allowed` | `allowed_account_ids` protection | Use the correct profile | 30 |
| `AccessDenied` / `UnauthorizedOperation` / `is not authorized to perform` | Authenticated but not allowed | Grant the named action; check SCPs/boundaries | 29, 64 |
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | OIDC trust conditions don't match | Fix `sub`/`aud` in the trust policy | 57 |
| `Credentials could not be loaded` (GitHub Actions) | Missing `id-token: write` | Add permission | 57 |
| `Required token could not be found` (HCP Terraform) | Not logged in | `terraform login` / `TF_TOKEN_*` | 50 |

### 8. Plans that surprise you

| Symptom | Meaning | Fix | Topic |
|---|---|---|---|
| `must be replaced` + `# forces replacement` | Changed force-new argument | Expected? If not, revert or `create_before_destroy` | 24 |
| `-> (known after apply) # forces replacement` | Upstream replacement cascade | Find the root cause resource | 11, 24 |
| `is tainted, so must be replaced` | Earlier create didn't finish | Let it replace, or `untaint` | 41 |
| `will be replaced due to changes in replace_triggered_by` | A trigger changed | Expected by design | 24 |
| Destroy + create at **different addresses** | Renamed/moved in code | `moved` block | 40 |
| `Objects have changed outside of Terraform` | Drift | Revert/adopt/ignore | 37 |
| Same diff on every plan | Normalisation, impure functions, another system owns it | Match API format; avoid `timestamp()`; `ignore_changes` | 18, 24, 37 |
| Plan replaces instances weekly | `most_recent` AMI | Pin AMI or `ignore_changes = [ami]` | 31, 61 |
| `Instance cannot be destroyed` | `prevent_destroy` caught a destroy/replace | Investigate; deliberate PR if intended | 24 |

### 9. Apply-time API errors (AWS examples)

| Error | Meaning | Fix | Topic |
|---|---|---|---|
| `BucketAlreadyExists` / `EntityAlreadyExists` / `already exists` | Name taken (globally, or by an unmanaged resource) | Unique/prefixed names; import | 31, 39 |
| `DependencyViolation` | Something still uses it | Destroy dependents first; find unmanaged users (ENIs, LBs) | 20, 32 |
| `InvalidParameterValue` / `ValidationError` | The API rejected a value | Read the message; add validation/preconditions | 25, 31 |
| `InvalidParameterValue ... architecture ... does not match` | AMI vs instance type architecture | Matching pair; precondition | 31 |
| `BucketNotEmpty` | Destroying a bucket with objects | Empty it / `force_destroy` (labs) | 31 |
| `ThrottlingException` / `Rate exceeded` | API rate limit | Lower parallelism; stagger; retries | 63 |
| `VcpuLimitExceeded` / `LimitExceeded` | Account quotas | Request increase; smaller resources | 31 |
| `Error: timeout while waiting for state to become ...` | Resource slow or stuck | Check console; increase `timeouts {}`; fix cause | 13 |
| `InvalidParameterValue ... iamInstanceProfile` just after creating it | IAM eventual consistency | Re-run apply | 33 |
| `Deletion protection is enabled` | Protected ALB/RDS | Disable via reviewed change, then destroy | 60 |

### 10. Provider and core bugs

| Error | Meaning | Fix | Topic |
|---|---|---|---|
| `Provider produced inconsistent final plan` | Provider planned differently between plan and apply | Re-run; upgrade provider; report | 13, 64 |
| `Provider produced inconsistent result after apply` | API returned something different from the plan | Usually harmless on re-run; report with `TRACE` | 13, 64 |
| `The plugin ... crashed!` / `Terraform crashed!` | Panic in provider/core | Versions + trace; search issues; upgrade; report | 64 |

### 11. Import, refactor, testing, CI

| Error | Meaning | Fix | Topic |
|---|---|---|---|
| `Cannot import non-existent remote object` | Wrong import ID/region/account | Check docs' import ID format | 39 |
| `Configuration for import target does not exist` | No resource block, no generation | Write it or `-generate-config-out` | 39 |
| `Moved object still exists` | `from` still declared | Remove the old block | 40 |
| `Unsupported cross-resource-type move` | Type can't convert state | `removed` + `import` | 40 |
| `Test assertion failed` | Your `terraform test` assertion | Read the values shown | 55 |
| `No test files found` | Wrong location/extension | `tests/*.tftest.hcl` | 55 |
| CI hangs | Waiting for input | `-input=false` / `TF_INPUT=0` | 09 |
| `Error: Failed to load "tfplan" as a plan file` | Wrong file or different Terraform version | Same version, same folder | 09 |

---

## Hands-on lab

Take every error you saved while doing the course labs. For each, find its row here, check that the "Fix" matches what you did, and add any missing ones to your own copy of this page.

---

## Where you'll use it

During incidents and failed pipelines; as interview revision ("what does `DependencyViolation` mean?").

---

## When it breaks

If an error isn't here: identify its **source** (core, backend, provider, API) from its wording, apply topic 64's method, search the provider's GitHub issues for the exact message, and add it to your copy.

---

## Quick recap

- Errors come from core, backend, provider or the remote API; the wording tells you which.
- Classify by phase, then use the table: meaning → fix → topic.
- The cloud's own message (`api error Code: message`) usually holds the answer.

### Interview questions

1. **Q: What does `Error acquiring the state lock` mean and how do you handle it?**
   A: Another operation holds the state lock, or a crashed run left a stale one. Wait or use `-lock-timeout`; if the holder is confirmed dead, back up the state, `force-unlock` with the lock ID, and run a plan.
2. **Q: What causes "Invalid for_each argument ... cannot be determined until apply"?**
   A: The `for_each` keys depend on values only known after apply. Use keys known at plan time (from variables or static maps) and keep unknown values in the map's values.
3. **Q: What does `Provider produced inconsistent result after apply` indicate?**
   A: A provider bug: the remote API returned values that differ from what the provider planned. Re-running usually reconciles; upgrade the provider and report it with a minimal reproduction and debug logs.
