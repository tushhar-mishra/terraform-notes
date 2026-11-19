# 09 · The Core Workflow Commands

> **Level:** ★ foundation
>
> **You will learn:** the everyday commands and the flags that matter in real work: `init` (and `-upgrade`, `-backend=false`), `validate`, `fmt`, `plan` (`-out`, `-destroy`, `-detailed-exitcode`, `-input=false`), `apply` (with and without a **saved plan**, `-auto-approve`), `destroy`, and `show`. Also every **plan symbol** (`+ ~ - -/+ +/- <=`), what a **stale plan** is, and the environment variables that change command behaviour in CI.
>
> **You need:** topics 07–08.
>
> **Part 1 · First steps and the core loop** · Previous: [08 · HCL syntax](08-hcl-syntax.md) · Next: [10 · Providers](10-providers.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Five commands do 95% of the work:

```bash
terraform init                    # set up the folder (providers, modules, backend)
terraform fmt && terraform validate
terraform plan -out=tfplan        # preview, and SAVE exactly what will happen
terraform apply tfplan            # do exactly that saved plan (no second prompt)
terraform destroy                 # delete everything in state
```

**Picture it.** A builder's routine: unpack the tools (`init`), check the drawings are tidy and legal (`fmt`/`validate`), write the quote (`plan`), sign the quote (`-out`), build exactly what's on the signed quote (`apply tfplan`).

**The one thing to know.** In real work, **apply the saved plan you reviewed**. `apply tfplan` does exactly what the reviewed plan said. A bare `apply` makes a **new** plan, which may differ if anything changed in between.

**Try it now:** in your topic 08 lab, run `terraform plan -out=tfplan`, then `terraform show tfplan`, then `terraform apply tfplan`.

**You've got it if you can:** name every plan symbol and explain why `apply tfplan` is safer than `apply`.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Command | Changes real infrastructure? | Changes state? | Needs credentials? |
|---|---|---|---|
| `init` | No | No (sets up the backend) | Only for the backend |
| `fmt` | No | No | No |
| `validate` | No | No | No |
| `plan` | **No** | No* | Yes (reads reality) |
| `apply` | **Yes** | Yes | Yes |
| `destroy` | **Yes** (deletes) | Yes | Yes |
| `show` | No | No | No |

\* `plan` never writes state. (`apply -refresh-only` does, topic 37.)

### The plan symbols

| Symbol | Meaning | Example cause |
|---|---|---|
| `+` | **Create** | New block in code |
| `-` | **Destroy** | Block removed from code, or `destroy` |
| `~` | **Update in place** | Changed an argument that the provider can change live (e.g. tags) |
| `-/+` | **Replace**: destroy first, then create | Changed an argument marked `# forces replacement` |
| `+/-` | **Replace**: create the new one first, then destroy the old one | Same, with `create_before_destroy` (topic 24) |
| `<=` | **Read** a data source during apply | A data source that depends on something not created yet (topic 19) |

Other markers inside a plan:

| Marker | Meaning |
|---|---|
| `(known after apply)` | The value will only exist once the provider creates/updates the object. |
| `# forces replacement` | **This** change causes the replacement. |
| `(sensitive value)` | Hidden because it's marked sensitive (topic 14). |
| `# (3 unchanged attributes hidden)` | Unchanged attributes are folded to keep the plan short. |
| `Note: Objects have changed outside of Terraform` | Drift found during refresh (topic 37). |

---

## Why it exists

Running `apply` and reading the prompt works on your laptop. In a team and in CI:

- The person (or bot) who **reviews** the plan must approve **exactly** what will run. That needs a **saved plan**.
- Pipelines can't type `yes`. They need `-input=false` and either a saved plan or `-auto-approve`.
- Pipelines need **exit codes** that mean something ("are there changes?").
- Logs should be machine-readable sometimes (`-json`).

---

## Analogy

**A builder's quote.**

- `plan` = a written quote: "add a window, repaint a room".
- `plan -out=tfplan` = the **signed** quote. `apply tfplan` = "build exactly what's on this signed quote". If the house changed since the quote was written (someone else built something), the builder refuses: "my quote is out of date" (**stale plan**).
- `apply` with no plan file = "come round, look again, tell me what you'll do, and do it if I say yes right now".
- `destroy` = a demolition quote.

---

## How it works inside

### A saved plan file

`terraform plan -out=tfplan` writes a **binary zip file** containing:

- the planned changes,
- a **snapshot of the configuration**,
- the **state it was based on** (including its `serial`, a version counter that goes up each time state changes),
- variable values (so **it can contain secrets**: never commit it, and treat it like state).

`terraform apply tfplan` then:

1. Checks the current state's serial/lineage still matches the one in the plan. If someone applied in between → **"Saved plan is stale"**, and nothing happens.
2. Skips the prompt (you already reviewed it).
3. Performs exactly those actions. It does **not** re-plan.

### `destroy` is just a special plan

`terraform destroy` = `terraform apply -destroy`: make a plan where **every** resource in state is deleted, in reverse dependency order, then apply it. `terraform plan -destroy` previews it.

### Exit codes

| Command | Exit code |
|---|---|
| Any command, success | `0` |
| Any command, error | `1` |
| `plan -detailed-exitcode` | `0` no changes · `1` error · `2` **changes present** |
| `fmt -check` | non-zero if files need formatting |

---

## Code / commands

Set up a lab project (`~/tf-lab/09-workflow`):

```hcl
# versions.tf
terraform {
  required_version = ">= 1.14"
  required_providers {
    local = { source = "hashicorp/local", version = "~> 2.5" }
  }
}

# main.tf
resource "local_file" "app_config" {
  filename        = "${path.module}/app.conf"
  content         = "port=8080\n"
  file_permission = "0644"
}
```

### `init`

```bash
terraform init                    # first time, or after adding providers/modules/backend
terraform init -upgrade           # allow newer provider versions within constraints (topic 10)
terraform init -backend=false     # skip the backend (e.g. for validate in CI)
terraform init -input=false       # never prompt (CI)
```

`init` is **safe to re-run** any time. Re-run it whenever you change providers, modules or the backend.

### `plan` and saving it

```bash
terraform plan -out=tfplan
```

```
Terraform will perform the following actions:

  # local_file.app_config will be created
  + resource "local_file" "app_config" {
      + content              = <<-EOT
            port=8080
        EOT
      ...
      + file_permission      = "0644"
      + filename             = "./app.conf"
      + id                   = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

─────────────────────────────────────────────────────────────────────────────

Saved the plan to: tfplan

To perform exactly these actions, run the following command to apply:
    terraform apply "tfplan"
```

- `Saved the plan to: tfplan`: the signed quote exists.

### `show`: read a saved plan or the state

```bash
terraform show tfplan                          # human-readable plan
terraform show -json tfplan | jq '.resource_changes[] | {address, actions: .change.actions}'
```

```json
{
  "address": "local_file.app_config",
  "actions": [
    "create"
  ]
}
```

- `-json` gives a machine-readable plan. Policy tools (topic 54) and PR comments (topic 57) read this.
- `actions` can be `["create"]`, `["update"]`, `["delete"]`, `["delete","create"]` (replace), `["create","delete"]` (create-before-destroy replace), `["read"]`, `["no-op"]`, or `["forget"]` (remove from state without destroying, topic 40).

`terraform show` with no file shows the **current state** in readable form.

### `apply` a saved plan

```
$ terraform apply tfplan
local_file.app_config: Creating...
local_file.app_config: Creation complete after 0s [id=3f2a...]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

No prompt: the plan was the approval.

### A stale plan

```bash
terraform plan -out=tfplan        # plan to change something...
terraform apply -auto-approve     # ...but someone else applies first
terraform apply tfplan            # now try the old plan
```

```
╷
│ Error: Saved plan is stale
│
│ The given plan file can no longer be applied because the state was changed
│ by another operation after the plan was created.
╵
```

Nothing was changed. Make a new plan and review it again.

### `plan -detailed-exitcode`

```bash
terraform plan -detailed-exitcode -input=false > /dev/null; echo "exit=$?"
```

```
exit=0        # nothing to do
```

Change `port=8080` to `port=9090` in the code, then run it again: `exit=2`. A CI job or drift checker uses this number: `2` = "something differs, notify someone".

### `apply` without a plan, and `-auto-approve`

```bash
terraform apply                      # plans, shows, asks "yes"
terraform apply -auto-approve        # plans and applies with NO review
```

⚠️ `-auto-approve` skips the human check. Fine for throwaway labs and for pipelines that apply a plan someone already approved. **Never** a habit on production from your laptop.

### `destroy` and its preview

```bash
terraform plan -destroy              # preview what destroy would delete
terraform destroy                    # same as: terraform apply -destroy
```

### Flags you'll see in CI

| Flag / variable | Effect |
|---|---|
| `-input=false` or `TF_INPUT=0` | Never prompt. A missing variable becomes an error instead of hanging the job. |
| `-no-color` | No colour codes in logs. |
| `-lock-timeout=5m` | Wait up to 5 minutes for the state lock instead of failing at once (topic 36). |
| `-compact-warnings` | Shorter warnings. |
| `-json` (plan/apply) | Machine-readable streaming output. |
| `TF_IN_AUTOMATION=1` | Removes "now run terraform apply" hints meant for humans. |
| `TF_CLI_ARGS_plan="-lock-timeout=5m"` | Adds flags to every `plan` without editing scripts. |

---

## Hands-on lab

In `~/tf-lab/09-workflow` with the files above:

1. `terraform init`, `terraform fmt -check`, `terraform validate`.
2. `terraform plan -out=tfplan`. Inspect it three ways: `terraform show tfplan`, `terraform show -json tfplan | jq '.resource_changes'`, and `ls -l tfplan` (it's a binary file). Apply it with `terraform apply tfplan`.
3. Change `file_permission` to `"0600"`. Run `terraform plan -detailed-exitcode; echo $?`. What symbol and exit code do you get, and why? (`-/+` and `2`: every `local_file` argument forces replacement.)
4. Reproduce a **stale plan**: plan with `-out`, then make another change and `apply -auto-approve`, then `apply` the old plan file.
5. `terraform plan -destroy`, then `terraform destroy`. Delete `tfplan`.
6. Add `tfplan` and `*.tfplan` to your `.gitignore` if you haven't.

---

## Where you'll use it

- **Every CI pipeline**: `fmt -check` → `init -input=false` → `validate` → `plan -out=tfplan` → (approval) → `apply tfplan` (topics 56–58).
- **PR reviews**: the plan is posted on the PR, and reviewers approve **the plan**, not only the code.
- **Drift detection**: scheduled `plan -detailed-exitcode`, alert on `2` (topic 37).
- **Policy checks**: `show -json tfplan` fed into OPA (Open Policy Agent), a rules engine (topic 54).

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `Error: Saved plan is stale` | State changed after the plan was made. | Re-plan and re-review. This error is protecting you. |
| CI job hangs forever | Terraform is waiting for input (a missing variable or the `yes` prompt). | Add `-input=false`; pass variables; use a saved plan. |
| `Error: Failed to load "tfplan" as a plan file` | Wrong file, or made by a different Terraform version. | Plan and apply with the **same** Terraform version and folder. |
| A plan applied in CI did something different from the reviewed plan | The pipeline ran `apply` without the saved plan, so it re-planned. | Save the plan as a pipeline artifact and apply that file. |
| `Error: Inconsistent dependency lock file` after a teammate's change | Providers changed. | `terraform init` (or `init -upgrade` if versions changed deliberately). |
| `destroy` wants to delete things you didn't expect | It deletes **everything in this state**, maybe more than you think. | Always `plan -destroy` first. Check you're in the right folder and **workspace** (a separate state for the same code, topic 48; check with `terraform workspace show`). |

---

## Quick recap

- `init` (set up) → `fmt`/`validate` (tidy, check) → `plan -out=tfplan` (preview and save) → `apply tfplan` (do exactly that) → `destroy` (delete all).
- Symbols: `+` create, `-` destroy, `~` update, `-/+` replace (destroy first), `+/-` replace (create first), `<=` read.
- A saved plan is **all-or-nothing and can go stale**; it may contain secrets.
- CI flags: `-input=false`, `-detailed-exitcode`, `-lock-timeout`, `TF_IN_AUTOMATION`.

### Interview questions

1. **Q: Why use `terraform plan -out` and `terraform apply <planfile>`?**
   A: So exactly the reviewed changes are applied. A bare `apply` re-plans, and reality or state may have changed since review. A saved plan is rejected as stale if state changed.
2. **Q: What is the difference between `-/+` and `+/-`?**
   A: Both are replacements. `-/+` destroys the old object first, then creates the new one (the default). `+/-` creates the new one first, then destroys the old, which happens with `create_before_destroy` and avoids downtime.
3. **Q: How would you detect drift in a pipeline?**
   A: Run `terraform plan -detailed-exitcode` on a schedule. Exit code 2 means the plan has changes, so alert. Or use `plan -refresh-only` to show only out-of-band changes.
