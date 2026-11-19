# 36 · State Locking

> **Level:** ★★ intermediate
>
> **You will learn:** why two Terraform runs on the same state at the same time is dangerous, how **locking** prevents it, which commands take the lock, how the **S3 native lock file** (`use_lockfile`) works using S3 **conditional writes**, why **DynamoDB locking is deprecated** and how to migrate off it, how to read a **lock error**, `-lock-timeout`, ⚠️ `-lock=false`, and **`terraform force-unlock`**: exactly when it's safe and when it will corrupt your state. Plus how CI pipelines avoid lock fights in the first place.
>
> **You need:** topics 13 (the lock in the trace), 35 (S3 backend).
>
> **Part 5 · State in depth** · Previous: [35 · Remote backends](35-remote-backends.md) · Next: [37 · Drift](37-drift.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Before any command that might **write** state (and `plan`, to get a consistent read), Terraform takes a **lock**. A second run on the same state waits or fails with `Error acquiring the state lock` until the first run finishes and releases it.

```hcl
backend "s3" {
  bucket       = "tf-course-state-123456789012"
  key          = "dev/app/terraform.tfstate"
  region       = "ap-south-1"
  use_lockfile = true          # lock = the object dev/app/terraform.tfstate.tflock
}
```

```bash
terraform apply -lock-timeout=5m      # wait up to 5 minutes for the lock instead of failing at once
terraform force-unlock <LOCK_ID>      # ⚠️ only when you're SURE the holder is dead
```

**Picture it.** A shared meeting room with a single "In use" sign. You flip it before entering and flip it back when you leave. Anyone who finds it flipped waits. If someone faints inside (a crashed run), the sign stays flipped; you check the room is really empty before flipping it back yourself.

**The one thing to know.** Never `force-unlock` a lock that might belong to a **running** apply. Check who holds it (the `Who` and `Created` fields) and confirm that process or pipeline is dead.

**Try it now:** with the S3 backend from topic 35, start a slow apply in one terminal and run `terraform plan` in another.

**You've got it if you can:** explain what happens if two applies run on the same state without locking, and list the checks before `force-unlock`.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Why concurrent runs are dangerous

Without a lock, two runs on the same state:

```
Alice: read state (serial 10) ─ plan ─ apply: create db ───────── write state (serial 11: has db)
Bob:   read state (serial 10) ─ plan ─ apply: create cache ──────────── write state (serial 11: has cache, NO db)
```

Bob's write wins. The database now exists in AWS but **not in state**: an orphan nobody manages, and the next plan may try to create it again. Worse cases: both create the same resource; one destroys what the other just built.

### Which commands lock

Every command that reads state to make decisions or writes it: `plan`, `apply`, `destroy`, `import`, `refresh`/`apply -refresh-only`, `state mv/rm/push/replace-provider`, `taint`/`untaint`, `console` (while open), `force-unlock` (the opposite). Read-only `state list`/`state show`/`show`/`output` don't need to.

### Lock implementations

| Backend | Lock mechanism |
|---|---|
| `local` | An OS file lock on `terraform.tfstate` + `.terraform.tfstate.lock.info` (same machine only) |
| **`s3` + `use_lockfile = true`** | A `<key>.tflock` object created with an S3 **conditional write** ("create only if it doesn't exist") |
| `s3` + `dynamodb_table` | ⚠️ **Deprecated.** An item in a DynamoDB table (`LockID` key) |
| `azurerm` | A **lease** on the state blob |
| `gcs` | A `.tflock` object (generation preconditions) |
| HCP Terraform | Built in; runs are queued per workspace |

### S3 native locking

S3 added **conditional writes** in 2024: a `PutObject` with `If-None-Match: *` succeeds **only if the object doesn't exist**. Terraform 1.10 used this to lock without any other service; it became generally available in **1.11**, and the `dynamodb_table` argument (and its related `dynamodb_*` settings) were **deprecated** at the same time and are planned for removal.

- Lock acquire: `PUT <key>.tflock` with the lock info JSON, conditional → `200` = locked by you; `412 Precondition Failed` = someone else holds it.
- Lock release: `DELETE <key>.tflock`.
- Needs `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` on the `.tflock` object (topic 35's IAM policy).

### Migrating from DynamoDB to the lock file

Both can be enabled **at the same time** during a transition (Terraform then takes **both** locks):

```hcl
backend "s3" {
  # ...
  use_lockfile   = true                        # step 1: add this, keep DynamoDB
  dynamodb_table = "terraform-locks"           # step 2: once every user/pipeline runs >= 1.11
}                                              #         with use_lockfile, remove this line
```

Run `terraform init -reconfigure` after changing backend settings (the state itself doesn't move). When nothing references the table any more, delete it.

### The lock info

```json
{
  "ID": "c9b3a1f2-7d4e-4b5a-9e6f-1a2b3c4d5e6f",
  "Operation": "OperationTypeApply",
  "Info": "",
  "Who": "runner@gha-runner-abc123",
  "Version": "1.17.2",
  "Created": "2026-11-19T15:04:05.123Z",
  "Path": "tf-course-state-123456789012/dev/app/terraform.tfstate"
}
```

- `ID`: what `force-unlock` needs.
- `Operation`: what's running: `OperationTypePlan`, `OperationTypeApply`, `OperationTypeInvalid` (state commands)...
- `Who`: `user@hostname` of the process holding it. In CI, a runner's name.
- `Created`: when. A lock from 3 minutes ago is probably a live run; one from 3 days ago is probably dead.

---

## Why it exists

Terraform is often run by several people and several pipelines against the same infrastructure. State is a single document; two writers will lose one's changes. Locking turns "silent corruption" into a clear error: "someone else is working on this; wait".

---

## Analogy

**A meeting room with a sign.**

- Before entering, you flip the sign to **"In use: Alice, 15:04, editing"** (lock info).
- Bob arrives, sees the sign, and either waits (`-lock-timeout`) or leaves (error).
- Alice finishes and flips it back (release).
- If Alice is rushed to hospital mid-meeting (a CI runner is killed), the sign stays on **"In use"** forever. Before Bob flips it back (`force-unlock`), he must **look inside** and make sure the room is truly empty. If Alice is actually still in there and Bob walks in and starts rearranging the furniture, chaos.
- `-lock=false` = ignoring the sign entirely. ⚠️

---

## How it works inside

```
terraform apply  (backend s3, use_lockfile)
 1. PUT  s3://bucket/dev/app/terraform.tfstate.tflock   If-None-Match: *
        body = lock info JSON
    ├─ 200 OK                 → we hold the lock
    └─ 412 PreconditionFailed → GET the .tflock to show who holds it
                                → with -lock-timeout: retry with backoff until timeout
                                → otherwise: "Error acquiring the state lock"
 2. GET  terraform.tfstate → plan → (approve) → apply → PUT terraform.tfstate
 3. DELETE terraform.tfstate.tflock
 (Ctrl-C once: Terraform stops gracefully, saves state, releases the lock.
  Ctrl-C twice / kill -9 / runner killed: step 3 never happens → STALE LOCK)
```

A **stale lock** is just a lock whose owner died. Terraform can't tell a stale lock from a live one: a slow apply of an RDS database can legitimately hold the lock for 20 minutes. That's why only a human (or a careful script) should decide to `force-unlock`.

---

## Code / commands

Use the topic 35 lab project (state in S3, `use_lockfile = true`). Add a slow resource so you have time to experiment:

```hcl
resource "time_sleep" "slow" {
  create_duration = "90s"
}
```

(Add `time = { source = "hashicorp/time", version = "~> 0.13" }` to `required_providers` and run `terraform init`.)

### Two runs at once

Terminal 1:

```bash
terraform apply -auto-approve
```

Terminal 2, while terminal 1 waits on `time_sleep`:

```bash
aws s3 ls s3://tf-course-state-123456789012/lab/34-state/
terraform plan
```

```
2026-11-19 15:04:05        233 terraform.tfstate.tflock
2026-11-19 14:59:31       8921 terraform.tfstate

╷
│ Error: Error acquiring the state lock
│
│ Error message: operation error S3: PutObject, https response error
│ StatusCode: 412, RequestID: ..., api error PreconditionFailed: At least one
│ of the pre-conditions you specified did not hold
│ Lock Info:
│   ID:        c9b3a1f2-7d4e-4b5a-9e6f-1a2b3c4d5e6f
│   Path:      tf-course-state-123456789012/lab/34-state/terraform.tfstate
│   Operation: OperationTypeApply
│   Who:       you@laptop
│   Version:   1.17.2
│   Created:   2026-11-19 15:04:05.123 +0000 UTC
│   Info:
│
│ Terraform acquires a state lock to protect the state from being written
│ by multiple users at the same time. Please resolve the issue above and try
│ again. For most commands, you can disable locking with the "-lock=false"
│ flag, but this is not recommended.
╵
```

| Part | Meaning |
|---|---|
| `StatusCode: 412 ... PreconditionFailed` | The conditional `PutObject` failed: the `.tflock` object already exists. |
| `Operation: OperationTypeApply` | The holder is applying, the most dangerous thing to interrupt. |
| `Who: you@laptop` | It's you, in terminal 1. |
| `Created` | 1 minute ago: certainly alive. **Wait.** |

Now with a timeout, terminal 2 waits instead of failing:

```bash
terraform plan -lock-timeout=3m
```

```
Acquiring state lock. This may take a few moments...
(waits until terminal 1 finishes, then plans normally)
```

### Creating a stale lock (on purpose)

Destroy the `time_sleep` (`terraform apply -replace=time_sleep.slow -auto-approve` starts a new 90 s create), then in that same terminal press **Ctrl-C twice** (or `kill -9` the `terraform` process from another terminal):

```
^C
Interrupt received.
Please wait for Terraform to exit or data loss may occur.
Gracefully shutting down...
^C
Two interrupts received. Exiting immediately. Note that data loss may have occurred.
```

The `.tflock` object is still in S3. Every run now fails with the lock error.

### Safe `force-unlock` checklist

Before running it, confirm **all** of these:

1. **Read the lock info**: `Who`, `Operation`, `Created`.
2. **Find the holder**: is that person's terminal / that CI job **really** finished or killed? Check the pipeline UI, ask the person, `ps -ef | grep terraform` on the machine.
3. **Check the age** against how long the operation could legitimately take.
4. **Back up the state**: `terraform state pull > before-unlock.tfstate` (reading doesn't need the lock).
5. Then unlock:

```bash
terraform force-unlock c9b3a1f2-7d4e-4b5a-9e6f-1a2b3c4d5e6f
```

```
Do you really want to force-unlock?
  Terraform will remove the lock on the remote state.
  This will allow local Terraform commands to modify this state, even though it
  may still be in use. Only 'yes' will be accepted to confirm.

  Enter a value: yes

Terraform state has been successfully unlocked!

The state has been unlocked, and Terraform commands should now be able to
obtain a new lock on the remote state.
```

6. **Run `terraform plan` immediately.** If the killed run was mid-apply, some resources may exist without being in state (orphans), or be marked tainted. The plan shows it; see topic 42 for repairs.

`force-unlock` only works with the lock's **ID** (it refuses if the ID doesn't match the current lock), so you can't accidentally remove a newer lock someone just took.

### ⚠️ `-lock=false`

```bash
terraform plan -lock=false
```

Skips locking entirely. Acceptable only for a **read-only** plan when you fully understand that the result might be inconsistent (e.g. a quick look during an incident while a known-long apply runs). **Never** with `apply`.

### Avoiding lock fights in CI

Locks are the last line of defence. Pipelines should avoid collisions **before** they happen:

- **GitHub Actions**: a `concurrency` group per state, e.g. `concurrency: { group: terraform-${{ inputs.env }}-app, cancel-in-progress: false }` ([GitHub Actions topic 29](../../GitHub-Actions-Basics-To-Production/notes/29-concurrency.md)). Never cancel an in-progress **apply** (that's how stale locks are born).
- **Jenkins**: `lock(resource: "tf-dev-app")` from the Lockable Resources plugin, or `disableConcurrentBuilds()` ([Jenkins topic 38](../../Jenkins/notes/38-when-input-milestone-lock.md)).
- **Everyone**: `-lock-timeout=5m` in pipelines so short overlaps wait instead of failing.

---

## Hands-on lab

1. Add `time_sleep`, run two commands at once, read the lock error, and find the `.tflock` object in S3 (`aws s3 cp s3://.../terraform.tfstate.tflock - | jq`).
2. Repeat with `-lock-timeout=3m` in terminal 2.
3. Create a stale lock with a double Ctrl-C. Go through the **whole** `force-unlock` checklist, unlock, and run `terraform plan`. Is `time_sleep.slow` in state? Tainted? (It depends on the moment you interrupted.)
4. Try `terraform force-unlock` with a wrong ID and read the refusal.
5. Look at the **Versions** of `terraform.tfstate` in S3: see one version per successful write.
6. Remove `time_sleep` and apply to clean up.

---

## Where you'll use it

- **Daily**: lock errors when a teammate or pipeline is running; the answer is usually "wait".
- **Incidents**: stale locks after a cancelled pipeline or a runner that died, which is the most common real-world reason to use `force-unlock`.
- **Platform work**: migrating backends from DynamoDB locking to `use_lockfile` across many repositories.
- **Pipeline design**: concurrency groups so applies to the same state never overlap.

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `Error acquiring the state lock` with a recent `Created` | Someone/something is running. | Wait, or use `-lock-timeout`. Find out who. |
| Same error with an old `Created` and a dead holder | **Stale lock**. | Checklist → `force-unlock <ID>` → `plan`. |
| `AccessDenied` on `...tflock` | The identity can read/write the state but not the lock object. | Add `s3:PutObject`/`DeleteObject` on `<key>.tflock`. |
| `Warning: Deprecated Parameter ... dynamodb_table` | Old locking setup. | Migrate to `use_lockfile` (both on, then remove DynamoDB). |
| `Error releasing the state lock` | The lock couldn't be deleted (network, permissions); it may still exist. | Check the `.tflock` object; remove with `force-unlock` once you're sure the run finished. |
| Two pipelines keep blocking each other | No serialization in CI. | Concurrency groups / Jenkins `lock`. |
| `Local state cannot be unlocked by another process` | Trying to force-unlock a **local** lock held by another live process. | Find and stop that process; local locks disappear when it exits. |

---

## Quick recap

- Locking stops two runs writing the same state; most commands that read-to-decide or write take the lock.
- S3: `use_lockfile = true` → a `<key>.tflock` object created with a conditional write. ⚠️ `dynamodb_table` is deprecated; enable both during migration, then remove DynamoDB.
- Lock info (`ID`, `Operation`, `Who`, `Created`) tells you who holds it.
- `-lock-timeout` to wait; ⚠️ never `-lock=false` on apply.
- `force-unlock <ID>` only after confirming the holder is dead and backing up state; then `plan` to find damage.

### Interview questions

1. **Q: What is state locking and why is it needed?**
   A: A mechanism that lets only one Terraform operation work with a given state at a time. Without it, concurrent runs read the same state and the last write wins, losing resources from state or creating duplicates.
2. **Q: How is locking done with the S3 backend today?**
   A: With `use_lockfile = true` (Terraform 1.10+, GA in 1.11): Terraform creates a `.tflock` object next to the state using an S3 conditional write that only succeeds if it doesn't exist. DynamoDB-based locking is deprecated.
3. **Q: When is it safe to use `terraform force-unlock`?**
   A: Only when you've confirmed the process holding the lock is no longer running (killed CI job, crashed machine), ideally after backing up the state. Then run a plan to check for partial changes. Unlocking a live run can corrupt state.
