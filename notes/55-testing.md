# 55 · Testing Terraform

> **Level:** ★★★ advanced
>
> **You will learn:** the **test pyramid** for infrastructure (static checks → plan-level unit tests → apply-level integration tests → end-to-end), Terraform's built-in **`terraform test`** framework (1.6+): `.tftest.hcl` files, `run` blocks with `command = plan` or `apply`, `assert`, `variables`, `expect_failures` for validation tests, helper `module` runs, **mock providers** and **overrides** (1.7+) so tests run **without cloud credentials or cost**, parallel runs (1.12+), and where **Terratest** fits.
>
> **You need:** topics 14 (validation), 25 (conditions), 43–44 (modules), 53–54.
>
> **Part 8 · Security and quality** · Previous: [54 · Policy as code](54-policy-as-code.md) · Next: [56 · The Git workflow for Terraform](56-git-workflow.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** `terraform test` runs **test files** that plan or apply your module with given inputs and **assert** on the results, then clean up automatically.

```hcl
# modules/app-config/tests/basic.tftest.hcl
variables {
  app_name    = "api"
  environment = "test"
}

run "file_name_follows_convention" {
  command = plan                               # fast: no real resources
  assert {
    condition     = local_file.this.filename == "./out/test-api.conf"
    error_message = "Unexpected filename: ${local_file.this.filename}"
  }
}

run "rejects_bad_port" {
  command = plan
  variables { port = 80 }
  expect_failures = [var.port]                 # the validation SHOULD fail
}
```

```bash
cd modules/app-config && terraform init && terraform test
```

```
tests/basic.tftest.hcl... in progress
  run "file_name_follows_convention"... pass
  run "rejects_bad_port"... pass
tests/basic.tftest.hcl... tearing down
tests/basic.tftest.hcl... pass

Success! 2 passed, 0 failed.
```

**Picture it.** Crash-testing a car design: some tests are computer simulations (plan + mocks: fast, free), some are real crashes in a lab (apply: slow, costs money, but proves it works).

**The one thing to know.** Use `command = plan` and **mock providers** for most tests (fast, free, no credentials); keep a few real `apply` tests for what only the cloud can prove.

**Try it now:** add the test file above to your topic 43 module (after adding a port validation) and run it.

**You've got it if you can:** write a plan test, an apply test and a validation test, and explain when to mock.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### The test pyramid for Terraform

| Level | What | Tools | Speed / cost |
|---|---|---|---|
| **Static** | Format, syntax, lint, security, policy | `fmt`, `validate`, TFLint, Trivy/Checkov, Conftest (topics 53–54) | Seconds, free |
| **Unit (plan)** | "With these inputs, the plan contains X" | `terraform test` with `command = plan`, mocks | Seconds, free |
| **Integration (apply)** | "The module really creates working infrastructure" | `terraform test` with `command = apply`; Terratest | Minutes, costs a little |
| **End-to-end** | "The whole environment works" (HTTP 200, DB reachable) | Smoke tests after deploy; `check` blocks (topic 25) | Minutes, real env |

Most effort should go into the lower levels; a few high-value apply tests at the top.

### Test files

- Files ending in **`.tftest.hcl`** (or `.tftest.json`), in the module's root or a **`tests/`** folder.
- Run with `terraform test` in the module directory (after `terraform init`).
- Each file is an independent scenario; its `run` blocks execute **in order** and share state; at the end, everything applied is **destroyed** (in reverse).

### Blocks in a test file

| Block | Purpose |
|---|---|
| `variables { }` (file level) | Inputs for every run in the file |
| `provider "aws" { }` | Configure providers for the test (e.g. region) |
| `mock_provider "aws" { }` | Replace a provider with a fake that returns generated values (1.7+) |
| `override_resource { }` / `override_data { }` / `override_module { }` | Fix the values a specific resource/data source/module returns (1.7+) |
| `run "name" { }` | One test step |

Inside `run`:

| Setting | Purpose |
|---|---|
| `command = plan` / `apply` | Plan only (default is `apply`) |
| `variables { }` | Run-specific inputs (override file-level ones) |
| `assert { condition, error_message }` | One or more checks; can reference resources, outputs (`output.x`), variables, and earlier runs (`run.setup.some_output`) |
| `expect_failures = [var.x, resource.y, output.z, check.w]` | The run **passes** only if these validations/conditions fail |
| `module { source = "./tests/setup" }` | Run a different module (e.g. create prerequisites) instead of the one under test |
| `plan_options { refresh = false, replace = [...] }` | Plan flags |
| `state_key`, `parallel = true` | Separate state / allow parallel execution (1.12+) |

### Mocks and overrides

```hcl
mock_provider "aws" {
  mock_resource "aws_s3_bucket" {
    defaults = {
      arn = "arn:aws:s3:::mocked"          # computed attributes get these values
    }
  }
}
```

- With a mock provider, **no API calls** happen and **no credentials** are needed. Computed attributes get generated fake values (or your `defaults`).
- Mocks test **your logic** (naming, conditions, `for_each` shapes, outputs), not AWS behaviour.
- Overrides replace the result of one specific resource/data source/module, e.g. `override_data { target = data.aws_availability_zones.available  values = { names = ["a","b","c"] } }`.

### Terratest

**Terratest** (Gruntwork, Go) runs real `terraform init/apply`, then lets you test the result with Go code (HTTP requests, SSH, AWS SDK calls), then `destroy`. More powerful for end-to-end checks; requires Go knowledge. Many teams now use `terraform test` for most cases and Terratest for complex integration suites.

---

## Why it exists

Modules are shared code that creates real infrastructure. A change to a naming expression or a `for_each` key can replace production resources for every caller. Tests catch that in seconds, in CI, before a version is released (topic 45). `terraform test` (1.6+) made testing native, in HCL, with no extra language; mocks (1.7+) made it free and credential-less.

---

## Analogy

**Crash-testing a car.**

- **Static checks** = inspecting the drawings for missing bolts.
- **Plan tests with mocks** = computer simulations: thousands of scenarios, cheap and fast, but only as good as the model.
- **Apply tests** = real crash tests in a lab with a real car: expensive, slower, but they prove it.
- **`expect_failures`** = testing that the seatbelt **locks** when it should: a test that passes when the safety mechanism triggers.
- **Cleanup** = the lab clears away the wreck after every test.

---

## How it works inside

```
terraform test
  for each *.tftest.hcl (sequentially unless parallel allowed):
    for each run block in order:
      build config: the module under test (or run.module), with file + run variables
      providers: real, or mock_provider (no network), plus overrides
      command = plan  → plan in memory → evaluate asserts against planned values
      command = apply → apply into a TEST state (in memory, never your real backend)
                       → evaluate asserts against the applied values
      expect_failures → the run passes only if exactly those checkable objects fail
    teardown: destroy everything the file's apply runs created, in reverse order
  print results; exit code 1 if any run failed
```

Important: tests use a **separate, in-memory state**, never your configured backend, so they can't touch your real state. But apply runs with **real** providers create **real** resources in whatever account your credentials point to: use a sandbox account.

---

## Code / commands

### Tests for the topic 43 module (free, local)

Add validation to `modules/app-config/variables.tf`:

```hcl
variable "port" {
  type    = number
  default = 8080
  validation {
    condition     = var.port >= 1024 && var.port <= 65535
    error_message = "port must be 1024-65535."
  }
}
```

`modules/app-config/tests/app_config.tftest.hcl`:

```hcl
variables {
  app_name    = "api"
  environment = "test"
}

run "plan_naming" {
  command = plan

  assert {
    condition     = endswith(local_file.this.filename, "/out/test-api.conf")
    error_message = "Filename must follow <env>-<app>.conf, got ${local_file.this.filename}"
  }

  assert {
    condition     = strcontains(local_file.this.content, "port=8080")
    error_message = "Default port should be 8080."
  }
}

run "custom_port" {
  command = plan
  variables {
    port = 9090
  }
  assert {
    condition     = output.port == 9090
    error_message = "port output should reflect the input."
  }
}

run "invalid_port_is_rejected" {
  command = plan
  variables {
    port = 80
  }
  expect_failures = [var.port]
}

run "apply_creates_the_file" {
  # command = apply is the default: really writes the file, then cleans it up
  assert {
    condition     = fileexists(local_file.this.filename)
    error_message = "The config file should exist after apply."
  }
}
```

```bash
cd ~/tf-lab/43-modules/modules/app-config
terraform init
terraform test -verbose
```

```
tests/app_config.tftest.hcl... in progress
  run "plan_naming"... pass
  run "custom_port"... pass
  run "invalid_port_is_rejected"... pass
  run "apply_creates_the_file"... pass

# local_file.this:
resource "local_file" "this" {
    content  = <<-EOT
        app=api
        env=test
        port=8080
    EOT
    filename = "./out/test-api.conf"
    ...
}

tests/app_config.tftest.hcl... tearing down
tests/app_config.tftest.hcl... pass

Success! 4 passed, 0 failed.
```

- `-verbose` prints the plan/state of each run.
- `tearing down`: the file created by the apply run was destroyed.

A failing assertion looks like:

```
  run "plan_naming"... fail
╷
│ Error: Test assertion failed
│
│   on tests/app_config.tftest.hcl line 9, in run "plan_naming":
│    9:     condition     = endswith(local_file.this.filename, "/out/test-api.conf")
│     ├────────────────
│     │ local_file.this.filename is "./out/api-test.conf"
│
│ Filename must follow <env>-<app>.conf, got ./out/api-test.conf
╵
Failure! 3 passed, 1 failed.
```

### Tests for the topic 44 S3 module with a mock provider (free, no credentials)

`modules/s3-secure-bucket/tests/unit.tftest.hcl`:

```hcl
mock_provider "aws" {}

variables {
  name = "acme-test-bucket"
}

run "secure_defaults" {
  command = plan

  assert {
    condition     = aws_s3_bucket_public_access_block.this.block_public_acls && aws_s3_bucket_public_access_block.this.restrict_public_buckets
    error_message = "Public access must be blocked."
  }

  assert {
    condition     = one(one(aws_s3_bucket_server_side_encryption_configuration.this.rule).apply_server_side_encryption_by_default).sse_algorithm == "AES256"
    error_message = "Default encryption should be SSE-S3 when no KMS key is given."
  }

  assert {
    condition     = aws_s3_bucket_versioning.this.versioning_configuration[0].status == "Enabled"
    error_message = "Versioning should be on by default."
  }
}

run "kms_when_key_given" {
  command = plan
  variables {
    kms_key_arn = "arn:aws:kms:ap-south-1:123456789012:key/abcd"
  }
  assert {
    condition     = one(one(aws_s3_bucket_server_side_encryption_configuration.this.rule).apply_server_side_encryption_by_default).sse_algorithm == "aws:kms"
    error_message = "KMS key given → aws:kms expected."
  }
}

run "bad_name_rejected" {
  command = plan
  variables {
    name = "Bad_Name"
  }
  expect_failures = [var.name]
}

run "apply_with_mocks" {
  command = apply                           # "applies" against the mock: no real AWS calls
  assert {
    condition     = output.arn != ""
    error_message = "ARN output should be set."
  }
}
```

```bash
unset AWS_PROFILE                           # prove no credentials are needed
terraform init && terraform test
```

```
tests/unit.tftest.hcl... in progress
  run "secure_defaults"... pass
  run "kms_when_key_given"... pass
  run "bad_name_rejected"... pass
  run "apply_with_mocks"... pass
tests/unit.tftest.hcl... tearing down
tests/unit.tftest.hcl... pass

Success! 4 passed, 0 failed.
```

- Nested blocks like `rule {}` are sets/lists in expressions: `one(...)` unwraps a single-element collection.
- With the mock provider, `arn` got a generated fake string.

### A real integration test (sandbox account, cents)

`tests/integration.tftest.hcl` (no `mock_provider`):

```hcl
provider "aws" {
  region = "ap-south-1"
}

run "setup_suffix" {
  module {
    source = "./tests/setup"             # a tiny module that outputs a random suffix
  }
}

run "real_bucket" {
  variables {
    name          = "tf-course-test-${run.setup_suffix.suffix}"
    force_destroy = true
  }
  assert {
    condition     = startswith(output.arn, "arn:aws:s3:::tf-course-test-")
    error_message = "Bucket ARN unexpected."
  }
}
```

Run only the fast tests by default, and the integration ones on demand:

```bash
terraform test -filter=tests/unit.tftest.hcl
terraform test -filter=tests/integration.tftest.hcl     # in a nightly CI job with a sandbox role
```

---

## Hands-on lab

1. Add the validation and the test file to the topic 43 module; run `terraform test -verbose`.
2. Break the naming logic on purpose and read the failure.
3. Add the mock-provider tests to the topic 44 module; run them with **no** AWS credentials.
4. Add an `override_resource` that sets `aws_s3_bucket.this` `arn` to a fixed value, and assert the output equals it.
5. (Optional, cents) Write the integration test with a setup module; run it in your sandbox account.
6. Add `terraform test` to your pre-commit config or CI plan (topic 57).

---

## Where you'll use it

- **Module repositories**: tests in CI before tagging a release (topic 45).
- **Refactors**: prove that a `for_each` change keeps the same addresses (assert on `keys(...)`).
- **Validation rules**: `expect_failures` tests make sure bad inputs are rejected.
- **Nightly integration runs** in a sandbox account, with teardown.

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `No test files found` | Wrong folder or extension. | `*.tftest.hcl` in the module root or `tests/`; run from the module dir. |
| `Unknown condition value` in a plan test | Asserting on a value only known after apply. | Use `command = apply` (with mocks if needed) or assert on known values. |
| Tests leave resources behind | A run failed during teardown (e.g. non-empty bucket). | `force_destroy` in tests; check the output; clean the sandbox. |
| `expect_failures` run fails | The validation didn't fail (or a different object failed). | Check the input really violates the rule; list exactly the expected object. |
| Credentials errors in unit tests | No mock provider. | `mock_provider "aws" {}` for plan-level tests. |
| Slow test suite | Too many apply tests. | Push logic tests down to plan + mocks; run integration nightly; `parallel = true` (1.12+). |

---

## Quick recap

- Pyramid: **static** (fmt/lint/scan/policy) → **plan tests** → **apply tests** → **end-to-end**.
- `terraform test` runs `*.tftest.hcl`: `run` blocks with `command = plan|apply`, `assert`, `variables`, `expect_failures`; state is in-memory; teardown is automatic.
- **Mock providers/overrides** (1.7+) give fast, free, credential-less tests of your logic.
- Real apply tests in a **sandbox** account; Terratest for complex Go-based integration tests.

### Interview questions

1. **Q: How do you test Terraform modules?**
   A: Static checks in CI, then `terraform test` with `.tftest.hcl` files: plan-level runs with mock providers to check logic and validations (`expect_failures`), plus a few apply-level integration tests in a sandbox account that create and destroy real resources. Terratest for complex end-to-end checks.
2. **Q: What do mock providers give you?**
   A: Tests that run without credentials or API calls, with fake computed values, so you can test naming, conditions, loops and outputs quickly and for free on every PR.
3. **Q: How do you test that a variable validation works?**
   A: A run block that passes an invalid value with `expect_failures = [var.name]`; the run passes only if that validation fails.
