# 13 · What Happens Inside `plan` and `apply`

> **Level:** ★★ intermediate
>
> **You will learn:** every step from typing `terraform apply` to `Apply complete!`: read CLI settings → load and check the configuration → open the backend and **lock** state → start providers and fetch their **schemas** → **configure** providers → **refresh** → build the **dependency graph** → **walk** the graph to compute the plan (diff) → show / save the plan → (apply) walk the graph again calling **provider APIs** in parallel → check results → **write state** as it goes → outputs → unlock. You'll also see what a **partial apply** leaves behind. This is Terraform's most-asked "explain the internals" interview question, like `kubectl apply` is for Kubernetes.
>
> **You need:** topics 03 (core, providers, gRPC), 09 (commands, saved plans), 10 (providers, lock file), 11 (references, unknown values), 12 (state, serial).
>
> **Part 1 · First steps and the core loop** · Previous: [12 · State: the basics](12-state-basics.md) · Next: [14 · Input variables](14-input-variables.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** `terraform apply` is a relay race inside one process:

1. **Load** the `.tf` files (and modules), check versions and the lock file.
2. **Lock** the state and **read** it.
3. **Start provider plugins** and ask each for its schema; **configure** them (credentials, region).
4. **Refresh**: ask providers to re-read every object in state → spot drift.
5. **Build the graph** of everything (resources, data sources, variables, outputs, providers) from references.
6. **Plan**: walk the graph; for each resource, ask its provider "if the config is this and reality is that, what will change?" → `+ ~ - -/+`.
7. **Show** the plan, wait for `yes` (or use a saved plan).
8. **Apply**: walk the graph again, **10 at a time** in parallel, calling `Create/Update/Delete` through providers, which call the real APIs.
9. **Save state** after each change; finally write outputs and **unlock**.

**Picture it.** A wedding planner: reads the brief (config), checks the notebook (state), phones every supplier to confirm what's really booked (refresh), draws the schedule (graph), writes the to-do list (plan), gets your signature, then calls suppliers in the right order, ticking the notebook after each call.

**The one thing to know.** Watch it happen:

```bash
TF_LOG=DEBUG terraform plan 2>&1 | grep -E 'starting plugin|Starting graph walk|references:' | head
```

**Try it now:** in your topic 11 lab, run that command and match the lines to steps 3, 5 and 6.

**You've got it if you can:** tell the whole story in order, and explain what happens to state if `apply` fails halfway.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

A **trace** of one `terraform plan` and one `terraform apply`. Knowing the steps tells you **where** a failure happened, which tells you **what kind** of problem it is:

| Where it failed | Kind of problem | Example errors |
|---|---|---|
| Loading config | Your HCL | `Unsupported argument`, `Duplicate resource` |
| Init checks | Providers/lock file | `Inconsistent dependency lock file` |
| Backend / lock | State storage | `Error acquiring the state lock`, `AccessDenied` on the state bucket |
| Provider configure | Credentials/region | `No valid credential sources found` |
| Refresh | Reading reality | `403` reading a resource, `resource not found` → drift |
| Graph | Your references | `Cycle: a, b` |
| Plan walk | Values / provider validation | `Invalid value`, `Invalid count argument` |
| Apply walk | The real API said no | `409 already exists`, `InvalidParameterValue`, timeouts |
| After apply | Provider bug | `Provider produced inconsistent result after apply` |

---

## Why it exists

Because "Terraform failed" isn't a diagnosis. Interviewers ask "what happens when you run `terraform apply`?" to see if you understand that Terraform is **state + graph + providers**, not magic. And in production, knowing that state is written **during** apply explains why a failed apply leaves some resources created and some not.

---

## Analogy

**A wedding planner.**

1. Reads your **brief** (config): 100 guests, a venue, flowers, a band.
2. Opens the **notebook** (state) and puts a "**busy, don't touch**" sign on it (lock).
3. Picks up the **phone list** of suppliers (providers) and checks each one's catalogue (schema).
4. **Phones every supplier** already booked to confirm what's really reserved (refresh). The florist says "you're booked for roses, but someone changed it to tulips" (drift).
5. Draws the **schedule** (graph): the venue must be booked before the flowers are delivered *to the venue*; the band and the cake are independent, so they can be arranged at the same time.
6. Writes the **to-do list** (plan): book band (+), change flowers back to roses (~), cancel the photographer you removed from the brief (−).
7. Gets **your signature** (`yes`).
8. Makes the calls, several at once where the schedule allows (parallel apply), **ticking the notebook after each call** (state written as it goes).
9. If the band's line is dead (API error), the venue and flowers are still booked and ticked. Next time, the planner only needs to retry the band (partial apply).
10. Removes the "busy" sign (unlock).

---

## How it works inside

### The full trace

```
terraform apply
  │
  ├─ 1. CLI start
  │     read ~/.terraformrc, TF_* env vars, parse flags
  │
  ├─ 2. Load configuration
  │     parse all *.tf (+ modules from .terraform/modules) → one config tree
  │     check required_version · check providers installed & match the lock file
  │
  ├─ 3. Backend + state
  │     open backend (local file / S3 ...) → ACQUIRE LOCK → read state ("prior state")
  │
  ├─ 4. Providers
  │     start each provider binary (child process, gRPC)
  │     GetProviderSchema → validate every resource/data block against it
  │     evaluate provider blocks → ConfigureProvider (region, credentials)
  │
  ├─ 5. Build the PLAN graph
  │     one node per resource, data source, variable, local, output, provider
  │     edges from references + depends_on + "provider before its resources"
  │     check for cycles → "Error: Cycle: ..."
  │
  ├─ 6. Walk the plan graph (parallel, default 10 at once)
  │     for each RESOURCE node, in dependency order:
  │       a. REFRESH: ReadResource(prior state) → current real attributes
  │          (object gone → removed; values differ → "changed outside of Terraform")
  │       b. evaluate the block's expressions (unknowns stay unknown)
  │       c. PlanResourceChange(prior, config) → planned new state
  │          + list of attributes that RequiresReplace
  │       d. decide action: create / update / delete+create / no-op / read
  │     for each DATA SOURCE: read now if its config is fully known, else "<= read during apply"
  │     resources in state but not in config ("orphans") → delete
  │
  ├─ 7. Render the plan → (plan -out: save file) → prompt "yes"
  │     (plan command stops here: release lock, exit)
  │
  ├─ 8. Build the APPLY graph from the plan's changes
  │     destroy actions get REVERSED edges (dependents destroyed first)
  │
  ├─ 9. Walk the apply graph (parallel, default 10)
  │     for each change:
  │       ApplyResourceChange(planned) → provider calls the real API
  │       ← new state of the object
  │       core checks: result consistent with plan? (else "Provider produced
  │       inconsistent result after apply")
  │       record the object in state; state is persisted as the walk goes
  │       error? → stop starting NEW nodes that depend on it; let running ones finish
  │
  ├─ 10. Evaluate outputs → final state write (serial + 1)
  │
  └─ 11. RELEASE LOCK → stop provider processes → print summary
```

### Step details worth knowing

**Step 3, locking.** Both `plan` and `apply` lock the state by default, so two people can't plan/apply the same state at once. The local backend uses an OS file lock and shows `.terraform.tfstate.lock.info`; S3 uses a lock object (topic 36). `-lock=false` disables it: ⚠️ don't, except in emergencies.

**Step 4, configure.** Provider blocks are evaluated **before** most of the graph. That's why a provider argument can't depend on a resource that doesn't exist yet (e.g. a Kubernetes provider configured from a cluster created in the same run is fragile; topic 28).

**Step 6a, refresh happens inside the plan walk.** In modern Terraform, refresh isn't a separate pass; each resource is re-read just before it's planned. `-refresh=false` skips it (faster, but blind to drift). `-refresh-only` does *only* this step and proposes updating state to match reality (topic 37).

**Step 6c, the provider decides.** Core doesn't know whether changing `instance_type` can be done in place. It sends prior state + new config to the provider's `PlanResourceChange`; the provider answers with the planned values and which attributes **require replacement**. That's the source of every `# forces replacement` line.

**Step 8, destroy order.** Creation follows dependencies forward (network → VM). Destruction goes backwards (VM → network). For a **replacement**, the default is "destroy old, then create new" (`-/+`); `create_before_destroy` flips it (`+/-`) (topic 24).

**Step 9, parallelism.** At most **10** operations run at once (`-parallelism=N` changes it). Independent resources run side by side; dependent ones wait. Lower it if you hit API rate limits (topic 63).

**Step 9, partial failure.**
- Resources that finished **stay in state**. The next `apply` doesn't redo them.
- A resource whose create call succeeded but which then failed (e.g. a timeout waiting for it to become ready) is saved as **tainted**: next plan will **replace** it.
- Resources that never started are simply still planned next time.

So the fix for a failed apply is almost always: **read the error, fix the cause, run `plan`/`apply` again.** Terraform is designed to be re-run.

**Step 10, the final write.** `serial` goes up. With a remote backend, state is uploaded; versioning keeps the previous one (topic 35).

### Who talks to whom

```
           gRPC                          HTTPS / socket
terraform ──────▶ provider process ───────────────────▶ platform API
  core    ◀────── (one per provider) ◀───────────────── (AWS, Docker...)
   │   GetProviderSchema, ConfigureProvider,
   │   ValidateResourceConfig, ReadResource,
   │   PlanResourceChange, ApplyResourceChange,
   │   ReadDataSource, ImportResourceState
   ▼
 backend (state + lock)
```

Core **never** calls the platform API itself. Providers **never** touch state; they return data and core writes it.

---

## Code / commands

### Watch the steps in the debug log

In your topic 11 project (`random_pet` → `local_file`):

```bash
TF_LOG=DEBUG terraform plan 2>&1 | grep -E 'Terraform version|backend/local|starting plugin|plugin started|references: \[|Starting graph walk|plan operation completed'
```

Trimmed and simplified sample output:

```
[INFO]  Terraform version: 1.17.2
[INFO]  backend/local: starting Plan operation
[DEBUG] provider: starting plugin: path=.terraform/providers/registry.terraform.io/hashicorp/random/3.7.2/linux_amd64/terraform-provider-random_v3.7.2_x5
[DEBUG] provider: plugin started: path=.../terraform-provider-random_v3.7.2_x5 pid=41872
[DEBUG] provider: starting plugin: path=.../hashicorp/local/2.5.3/linux_amd64/terraform-provider-local_v2.5.3_x5
[DEBUG] ReferenceTransformer: "local_file.motd" references: [random_pet.server]
[DEBUG] Starting graph walk: walkValidate
[DEBUG] Starting graph walk: walkPlan
[INFO]  backend/local: plan operation completed
```

| Line | Step |
|---|---|
| `Terraform version` | 1, CLI start |
| `starting Plan operation` | 3, backend opened, state loaded and locked |
| `starting plugin` / `plugin started ... pid=` | 4, each provider is its own process (look for it with `ps` during a long apply) |
| `"local_file.motd" references: [random_pet.server]` | 5, an **edge** in the graph |
| `walkValidate`, `walkPlan` | 5–6, graph walks (validate first, then plan) |
| `plan operation completed` | 7, done, lock released |

`TF_LOG` levels: `TRACE` (everything, including gRPC calls), `DEBUG`, `INFO`, `WARN`, `ERROR`. `TF_LOG_PATH=./tf.log` writes to a file. `TF_LOG_CORE` / `TF_LOG_PROVIDER` split core and provider logs (topic 64).

### See the graph

```bash
terraform graph
```

```
digraph G {
  rankdir = "RL";
  node [shape = rect, fontname = "sans-serif"];
  "local_file.motd" [label="local_file.motd"];
  "random_password.db" [label="random_password.db"];
  "random_pet.server" [label="random_pet.server"];
  "local_file.motd" -> "random_pet.server";
}
```

- Output is **DOT**, a text format for graphs. `"local_file.motd" -> "random_pet.server"` means "motd depends on server".
- Paste it into an online Graphviz viewer, or `sudo apt install -y graphviz && terraform graph | dot -Tpng > graph.png`. Terraform 1.16+ can also print **Mermaid** (a text diagram format that GitHub renders in Markdown): `terraform graph -format=mermaid`.
- `random_password.db` has no edges: it runs in parallel with everything. Topic 20 goes deeper.

### See the lock and the parallelism with a slow resource

Add the `hashicorp/time` provider (it has `time_sleep`, a resource that just waits):

```hcl
# add to required_providers:  time = { source = "hashicorp/time", version = "~> 0.13" }

resource "time_sleep" "wait" {
  create_duration = "30s"            # Create takes 30 seconds
  depends_on      = [random_pet.server]
}
```

`depends_on` is an **explicit** dependency: "create this after `random_pet.server`" even though nothing is referenced (topic 20). Run `terraform init` (new provider), then:

```bash
terraform apply -auto-approve
```

```
random_password.db: Creating...
random_pet.server: Creating...
random_pet.server: Creation complete after 0s [id=calm-heron]
random_password.db: Creation complete after 0s [id=none]
local_file.motd: Creating...
time_sleep.wait: Creating...
local_file.motd: Creation complete after 0s [id=4e7a...]
time_sleep.wait: Still creating... [00m10s elapsed]
time_sleep.wait: Still creating... [00m20s elapsed]
time_sleep.wait: Creation complete after 30s [id=2026-11-19T10:15:42Z]
```

- `local_file.motd` and `time_sleep.wait` **start together** once `random_pet.server` is done: they don't depend on each other.

**While it waits**, in a second terminal in the same folder:

```bash
cat .terraform.tfstate.lock.info | jq '{ID, Operation, Who, Created}'
terraform plan
```

```json
{
  "ID": "b7c4f1d2-...",
  "Operation": "OperationTypeApply",
  "Who": "you@laptop",
  "Created": "2026-11-19T10:15:12.4Z"
}
```

```
╷
│ Error: Error acquiring the state lock
│
│ Error message: resource temporarily unavailable
│ Lock Info:
│   ID:        b7c4f1d2-...
│   Path:      terraform.tfstate
│   Operation: OperationTypeApply
│   Who:       you@laptop
│   Version:   1.17.2
│   Created:   2026-11-19 10:15:12.4 +0000 UTC
│   Info:
│
│ Terraform acquires a state lock to protect the state from being written
│ by multiple users at the same time. Please resolve the issue above and try
│ again. For most commands, you can disable locking with the "-lock=false"
│ flag, but this is not recommended.
╵
```

That's step 3 protecting you. Also try `ps -ef | grep terraform-provider` during the wait: you'll see the provider processes from step 4.

### A partial apply

Make a resource fail **after** others succeed. Remove the `time_sleep` block (to save time), run `terraform destroy -auto-approve`, then add:

```hcl
resource "local_file" "bad" {
  filename   = "/root/forbidden.txt"     # your user can't write here
  content    = "x"
  depends_on = [local_file.motd]
}
```

```
$ terraform apply -auto-approve
...
random_pet.server: Creation complete after 0s [id=keen-lark]
random_password.db: Creation complete after 0s [id=none]
local_file.motd: Creating...
local_file.motd: Creation complete after 0s [id=8d2b...]
local_file.bad: Creating...
╷
│ Error: Create local file error
│
│   with local_file.bad,
│   on main.tf line 30, in resource "local_file" "bad":
│   30: resource "local_file" "bad" {
│
│ An unexpected error occurred while writing the file
│
│ +Original Error: open /root/forbidden.txt: permission denied
╵
```

- `with local_file.bad, on main.tf line 30`: which resource failed, and where it's defined.
- `Original Error: ... permission denied`: the real cause, passed up from the OS (or the cloud API) through the provider.

`terraform state list` shows everything **except** `local_file.bad`. Fix the path to `${path.module}/ok.txt`, apply again: only `local_file.bad` is created.

---

## Hands-on lab

1. In your topic 11 project, run the `TF_LOG=DEBUG` command and label each line with its step number.
2. `terraform graph` → draw the graph by hand, then check it with Graphviz or Mermaid.
3. Add `time_sleep` (with `terraform init` for the new provider) and watch: parallel starts, the lock file, the lock error in a second terminal, the provider processes in `ps`.
4. Run `terraform apply -parallelism=1 -auto-approve` after a `terraform destroy`. Everything now runs one at a time. Compare the timing with the default.
5. Do the **partial apply** experiment and fix it.
6. Run `TF_LOG=TRACE terraform plan 2>trace.log` and `grep -c 'PlanResourceChange\|ReadResource' trace.log` to see the gRPC calls of step 6.
7. `terraform destroy` and watch the reverse order. Delete `trace.log`.

---

## Where you'll use it

- **Interviews**: "Walk me through `terraform apply`" → steps 1–11 in your own words.
- **Debugging**: identify the failing phase first (table in "What it is"), then the cause.
- **Failed pipelines**: knowing that state is saved during apply, the answer to "apply failed halfway, what now?" is "fix the cause and re-run; don't delete state, don't run destroy in a panic".
- **Performance**: understanding refresh and parallelism explains slow plans and API throttling (topic 63).

---

## When it breaks

| Symptom | Phase | Meaning | Fix |
|---|---|---|---|
| `Error acquiring the state lock` | 3 | Another run holds the lock, or a dead run left it. | Wait; confirm nothing is running; `force-unlock` only then (topic 36). |
| `Error: configuring Terraform AWS Provider: no valid credential sources` | 4 | Provider can't authenticate. | Credentials/profile (topic 30). |
| `Error: Cycle: aws_security_group.a, aws_security_group.b` | 5 | Two resources reference each other. | Break the loop, e.g. use separate rule resources (topic 20). |
| `Note: Objects have changed outside of Terraform` | 6a | Drift detected during refresh. | Decide: code it, revert it, or ignore it (topic 37). |
| `Error: Invalid count argument ... depends on resource attributes that cannot be determined until apply` | 6b | `count`/`for_each` uses an unknown value. | Use values known at plan time (topics 21–22). |
| `Error: Provider produced inconsistent final plan` / `inconsistent result after apply` | 9 | The provider returned something different from what it planned: a **provider bug**. | Re-run; upgrade the provider; report it with a `TRACE` log (topic 64). |
| Apply failed after creating half the resources | 9 | Partial apply; finished resources are in state. | Fix the cause and apply again. |
| Resource shows `(tainted)` in `state list`/plan says `is tainted, so must be replaced` | 9 | Created but didn't finish successfully. | Usually let Terraform replace it; `terraform untaint` if you're sure it's fine (topic 41). |

---

## Quick recap

- Load config → lock + read state → start/configure providers → **refresh** → **build graph** → **plan walk** (provider decides changes and replacements) → approve → **apply walk** (parallel, 10 at a time, provider calls API) → **state saved as it goes** → outputs → unlock.
- Core never calls cloud APIs; providers never touch state.
- Destroys run in **reverse** dependency order.
- Failures leave a **partial** state that's safe to re-apply after fixing the cause.

### Interview questions

1. **Q: Walk me through what happens when you run `terraform apply`.**
   A: Terraform loads and validates the configuration, checks providers against the lock file, locks and reads state, starts the provider plugins and configures them, refreshes every resource in state to get real attributes, builds a dependency graph, and walks it asking each provider to plan the change. It shows the plan and waits for approval, then walks the graph again (up to 10 in parallel, destroys in reverse order), calling the providers, which call the platform APIs. It saves each result to state as it goes, writes outputs, and releases the lock.
2. **Q: An apply failed halfway. What state are you in and what do you do?**
   A: Resources that completed are recorded in state; the failed one may be marked tainted; the rest are still pending. Read the error, fix the cause (permissions, quota, a bad value), then run plan and apply again. Terraform continues from where it stopped.
3. **Q: Who decides whether a change is in place or a replacement?**
   A: The provider. During planning, core sends the prior state and new config to the provider's `PlanResourceChange`, which returns the planned values and which attributes require replacement.
