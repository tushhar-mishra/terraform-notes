# 42 · State Surgery and Recovery

> **Level:** ★★★ advanced
>
> **You will learn:** the low-level state commands (`state pull`, `state push`, `state rm`, `state mv`, `state replace-provider`) and the **golden rules** for using them; then a **recovery runbook** for every state emergency you're likely to meet: stale lock, failed state upload (`errored.tfstate`), a bad or corrupted state, a **deleted** state (restore from S3 versioning, or rebuild by importing), orphaned resources after a crash, state pointing at the wrong object, lineage/serial errors, version errors, provider source changes, and splitting or merging states. You'll practise three drills on free resources.
>
> **You need:** all of Part 5 (topics 34–41).
>
> **Part 5 · State in depth** · Previous: [41 · Replacing resources on purpose](41-replace.md) · Next: [43 · Modules: the basics](43-modules-basics.md) · [Back to INDEX](../INDEX.md)
>
> 💰 **Cost:** free (Docker lab with state in your topic 35 S3 bucket).

---

## ⚡ Core: the 10-minute version

**The idea.** Most state problems are fixed with **normal** Terraform (plan, apply, import, moved, removed). When they're not, you do **state surgery**, carefully, following five rules:

1. **Stop everyone else**: pause pipelines, tell the team. One surgeon at a time.
2. **Back up first**: `terraform state pull > backup-$(date +%F-%H%M%S).tfstate`.
3. **Prefer code over commands**: `import`/`moved`/`removed` blocks > `state mv`/`rm` > `state push` > hand-editing JSON.
4. **Change one thing at a time**, then run `terraform plan`.
5. **Done means** `terraform plan` shows exactly what you expect (usually `No changes`).

The two most valuable recovery tools:

```bash
aws s3api list-object-versions --bucket <state-bucket> --prefix <key>   # S3 versioning = undo history
terraform plan -generate-config-out=...  (with import blocks)            # rebuild a lost state
```

**Picture it.** A hospital ward's patient register. Usually you update it through normal admissions (plan/apply). If the register is damaged, you **photocopy it first**, close the ward to new admissions, fix one entry at a time, and then walk the ward to check every bed matches the register (plan).

**The one thing to know.** A versioned state bucket turns almost every disaster into "restore the previous version". If your state bucket isn't versioned, fix that today (topic 35).

**Try it now:** do Drill 1 below: break your state on purpose, restore it from an S3 version.

**You've got it if you can:** say what to do if (a) the state file was deleted, (b) an apply crashed halfway, (c) Terraform printed "Failed to persist state".

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### The state commands

| Command | What it does | Locks? | Notes |
|---|---|---|---|
| `terraform state list [ADDR]` | List addresses | No | Read-only |
| `terraform state show ADDR` | One resource's attributes | No | Read-only |
| `terraform state pull` | Print the whole current state (from any backend) to stdout | No | Your **backup** tool |
| `terraform state push FILE` | **Overwrite** the backend's state with a local file | Yes | Refuses a **lower serial** or **different lineage** unless `-force` |
| `terraform state rm ADDR...` | Forget objects (keep them in reality) | Yes | Code equivalent: `removed` (topic 40) |
| `terraform state mv SRC DST` | Change an address | Yes | Code equivalent: `moved` (topic 40) |
| `terraform state replace-provider FROM TO` | Change the provider source recorded for resources | Yes | For forks/registry moves |
| `terraform force-unlock ID` | Remove a stale lock | — | Topic 36 |
| `terraform untaint ADDR` | Clear a taint | Yes | Topic 41 |

All writing commands create a backup automatically when using the **local** backend (`terraform.tfstate.<timestamp>.backup`). With remote backends, **your** `state pull` backup and the bucket's **versioning** are the safety net.

### The golden rules (expanded)

| Rule | Why |
|---|---|
| **Freeze** | A pipeline applying during surgery will overwrite or conflict with your fix. Disable scheduled/merge-triggered runs; hold the lock if you can. |
| **Back up** | `state pull` to a file, and note the S3 version ID. Treat the backup like a secret (it is one). |
| **Least powerful tool** | Blocks are reviewed and repeatable; commands are immediate; `push -force` and hand-edits can destroy history. |
| **Small steps + plan** | After each step, `terraform plan` tells you if reality, state and config now agree. |
| **Write it down** | A short incident note: what broke, what you ran, the final plan. The next person will thank you. |

---

## Why it exists

State is a single document that a crash, a typo, a bad merge, a deleted file or a concurrent run can damage. Terraform is designed so most problems can be fixed by re-running it, but the remaining ones need precise tools. Knowing the runbook calmly is what makes a senior engineer valuable during an incident, and it's a classic interview topic ("you lost your state file, what now?").

---

## Analogy

**A hospital ward's patient register.**

- Normally the register (state) is updated only through admissions and discharges (plan/apply).
- If a page is torn out (lost state), you restore it from **yesterday's photocopy** (S3 versioning). If there's no photocopy, a nurse walks every bed and re-registers each patient (**import**).
- If a patient is in a bed but not in the register (orphan), you register them (import) or discharge them (delete).
- If the register lists a patient in bed 12 but bed 12 holds someone else (state pointing to the wrong object), you correct that entry (`state rm` + `import`).
- During the fix, the ward is **closed to new admissions** (freeze), and you photocopy the register **before** touching it (backup).

---

## How it works inside

### How `state push` protects you

```
terraform state push new.tfstate
  1. read the backend's current state: lineage L1, serial S1
  2. read new.tfstate:                  lineage L2, serial S2
  3. L2 != L1 → ERROR "cannot import state with lineage ... over unrelated state"
     S2 <  S1 → ERROR "cannot import state with serial ... over newer state"
     (-force skips both checks ⚠️)
  4. take the lock → write new.tfstate → release
```

`-force` is for when you **know** the file you're pushing is the right one (e.g. restoring an older version on purpose). Prefer restoring through S3 versioning, which keeps a full trail.

### Where state can be lost or damaged

```
apply ──▶ provider calls ──▶ state written in memory ──▶ upload to backend
   │            │                                           │
   │      crash/kill here: some objects                upload fails here:
   │      created but not recorded                     Terraform writes errored.tfstate
   │      → ORPHANS (scenario E)                       locally → push it (scenario B)
   │
 lock never released → STALE LOCK (scenario A)
```

---

## Code / commands: the recovery runbook

For each scenario: **symptoms → cause → steps**.

### A. Stale lock

- **Symptom:** `Error acquiring the state lock`, old `Created`, holder dead.
- **Steps:** topic 36's checklist → `force-unlock ID` → `plan`, then continue with E if the dead run was mid-apply.

### B. "Failed to persist state" / `errored.tfstate`

- **Symptom:** after an apply:

  ```
  │ Error: Failed to save state
  │
  │ Error saving state: failed to upload state: ... RequestError: send request failed
  │
  │ Error: Failed to persist state to backend
  │
  │ The error shown above has prevented Terraform from writing the updated state to
  │ the configured backend. To allow for recovery, the state has been written to the
  │ file "errored.tfstate" in the current working directory.
  │
  │ Running "terraform apply" again at this point will create a forked state,
  │ making it harder to recover.
  │
  │ To retry writing this state, use the following command:
  │     terraform state push errored.tfstate
  ```

- **Cause:** the apply **succeeded**, but the upload of the new state failed (network, expired credentials, permissions).
- **Steps:**
  1. **Don't** run apply again (the backend has the **old** state; a new apply would try to create things that already exist).
  2. Fix the cause (re-login: `aws sso login`).
  3. `terraform state push errored.tfstate`.
  4. `terraform plan` → expect `No changes`.
  5. Delete `errored.tfstate` (it contains secrets) once the push is verified.

### C. Bad or corrupted state (bad push, broken hand-edit, a wrong `state rm`)

- **Symptoms:** `Failed to load state: ... invalid character`, or a plan suddenly wanting to create/destroy many things after someone's surgery.
- **Steps (S3 backend with versioning):**

  ```bash
  B=tf-course-state-123456789012
  K=lab/34-state/terraform.tfstate

  # 1. list versions, newest first
  aws s3api list-object-versions --bucket "$B" --prefix "$K" \
    --query 'Versions[].{v:VersionId,at:LastModified,size:Size,latest:IsLatest}' --output table

  # 2. download a candidate and check it
  aws s3api get-object --bucket "$B" --key "$K" --version-id <VERSION_ID> good.tfstate
  jq '{serial, lineage, resources: (.resources | length)}' good.tfstate

  # 3. make it the current version again (S3 keeps the bad one as history)
  aws s3api copy-object --bucket "$B" --key "$K" \
    --copy-source "$B/$K?versionId=<VERSION_ID>"

  # 4. verify
  terraform plan
  ```

  ```
  ----------------------------------------------------------------------
  |                         ListObjectVersions                         |
  +--------------------------+--------+-------+------------------------+
  |            at            | latest | size  |           v            |
  +--------------------------+--------+-------+------------------------+
  |  2026-11-19T16:20:01Z    |  True  |  212  |  bad...Zx              |
  |  2026-11-19T15:58:44Z    |  False |  8921 |  good..Qp              |
  |  2026-11-19T14:59:31Z    |  False |  8710 |  older.Lm              |
  +--------------------------+--------+-------+------------------------+
  ```

  - The tiny `212`-byte latest version is suspicious (an almost-empty state). The previous one is the candidate.
  - **The restored version has a lower serial than the bad one.** That's fine with `copy-object` (you're changing S3 directly, not using `state push`). With `state push`, you'd need `-force`.
  - If anything was **applied after** the good version was written, its objects aren't in the restored state: handle them as orphans (E).

### D. State deleted entirely

- **Symptom:** plan wants to **create everything**; the state object is gone (or a new empty key is being used).
- **First, make sure it's really lost**: wrong folder? wrong `-backend-config`? wrong workspace? `-reconfigure` to an empty key (topic 35)? Check `aws s3 ls s3://bucket/ --recursive | grep tfstate`.
- **If the bucket is versioned**, a deleted object is just hidden behind a **delete marker**: remove the marker and the last version is back.

  ```bash
  aws s3api list-object-versions --bucket "$B" --prefix "$K" \
    --query 'DeleteMarkers[?IsLatest].VersionId' --output text
  aws s3api delete-object --bucket "$B" --key "$K" --version-id <DELETE_MARKER_VERSION_ID>
  terraform plan
  ```

- **If there's no backup at all**: rebuild by importing.
  1. **Freeze**: nobody applies (an apply now would try to create duplicates).
  2. Inventory what exists: by tags (`managed_by=terraform`, `stack=...`), the console, `terraform query` if your provider supports list resources (1.14+).
  3. Write `import` blocks for each resource (`for_each` for groups) (topic 39). The configuration already exists; you're only re-linking it.
  4. `terraform plan` → iterate until `N to import, 0 to add, 0 to change, 0 to destroy`.
  5. Apply. Then enable versioning on the state bucket so it never happens again.

### E. Orphans after a crash or a killed apply

- **Symptom:** after a killed apply/stale lock, plan wants to **create** something that already exists, or apply fails with `already exists`.
- **Cause:** the provider created it, but Terraform died before recording it.
- **Steps:** find the real object (console/CLI, by name/tags), then **import** it to the planned address (topic 39). Or, if it's half-broken, delete it by hand and let Terraform create a fresh one.

### F. State points at the wrong object (or an object that can't be read)

- **Symptoms:** plan shows changes to the wrong thing; or every plan fails while refreshing one resource (`Error reading ...` that isn't "not found", e.g. permissions on an object now owned by another account).
- **Steps:**
  ```bash
  terraform state pull > backup.tfstate
  terraform state rm 'aws_instance.web'                 # forget the wrong binding
  # then an import block: to = aws_instance.web, id = "<correct id>"
  terraform plan
  ```
  (A `removed { destroy = false }` block + `import` block in the same PR does the same as code.)

### G. Lineage / serial errors

| Error | Meaning | Action |
|---|---|---|
| `cannot import state with lineage "A" over unrelated state with lineage "B"` | You're pushing a state from a **different** project/state into this one. | Stop. You almost certainly have the wrong file or wrong backend key. |
| `cannot import state with serial 7 over newer state with serial 9` | Your file is **older** than what's there. | Usually stop. If you're deliberately restoring an older version, use S3 version restore (C) or `push -force` knowingly. |

### H. Version errors

| Error | Action |
|---|---|
| `state snapshot was created by Terraform v1.18.0, which is newer than current v1.17.2` | Upgrade your CLI to at least that version. Don't try to "downgrade" state. |
| `Resource instance managed by newer provider version` / schema version errors | Upgrade the provider to at least the version that wrote it (`init -upgrade` with a matching constraint). |

### I. Provider source changed

A provider moved namespaces, or you switch to a fork (or to the OpenTofu registry equivalent):

```bash
terraform state replace-provider registry.terraform.io/oldorg/foo registry.terraform.io/neworg/foo
```

```
Terraform will perform the following actions:

  ~ Updating provider:
    - registry.terraform.io/oldorg/foo
    + registry.terraform.io/neworg/foo

Changing 3 resources:

  foo_thing.a
  foo_thing.b
  foo_thing.c

Do you want to make these changes?
Only 'yes' will be accepted to continue.

Enter a value: yes

Successfully replaced provider for 3 resources.
```

Then update `required_providers` to the new source and `terraform init`.

### J. Splitting or merging states

Moving resources from configuration **X** (state X) to configuration **Y** (state Y):

1. In **Y**: add the resource blocks + `import` blocks with the real IDs. Plan: `N to import, 0 to change`. Don't apply yet.
2. In **X**: delete the resource blocks, add `removed { destroy = false }`. Plan: `N to forget`.
3. Apply **X** first (forget), then **Y** (import). Between the two, nobody manages them, so keep that window short.
4. Plan both: `No changes`.

Merging is the same in the other direction. Avoid the old `terraform state mv -state=... -state-out=...` trick: it only works with local state files and skips review.

### K. Hand-editing state (last resort)

Only when no command can do it (e.g. removing a corrupt attribute that a provider can't read):

```bash
terraform state pull > s.json
cp s.json s.backup.json
jq '(.resources[] | select(.type=="foo_thing" and .name=="a") | .instances[0].attributes.bad_field) = null
    | .serial += 1' s.json > s.fixed.json
terraform state push s.fixed.json
terraform plan
```

- **Increment `serial`**, or `push` refuses (the serial must be higher than the current one).
- Keep `lineage` unchanged.
- Validate the JSON (`jq . s.fixed.json`) before pushing.

---

## Hands-on lab

Use the topic 34 Docker lab whose state lives in S3 (topic 35). Containers are free; the bucket is versioned.

**Drill 1: restore a bad state from S3 versioning.**
1. `terraform state pull > drill-backup.tfstate`.
2. Damage it: `terraform state rm 'docker_container.web["a"]'`. Plan → wants to create `["a"]` (which exists → would fail with a name conflict).
3. Restore the previous version with `list-object-versions` + `copy-object` (scenario C). Plan → `No changes`.

**Drill 2: lost state → rebuild with imports.**
1. Back up, then delete the state object: `aws s3 rm s3://$B/$K`. Plan → wants to create everything.
2. **Do not apply.** Rebuild: for each container, `docker inspect -f '{{.Id}}' state-lab-a` etc., and add `import` blocks (use `for_each`).
3. Plan until the containers show only `will be imported`. The `docker_image` will show `will be created`: that's fine, because "creating" an image that's already pulled is harmless and changes nothing. Not every resource needs importing; judge each one by what its create would actually do. Apply.
4. Bonus: delete again, and this time recover by removing the **delete marker** instead.

**Drill 3: `errored.tfstate`.**
1. Start an apply that changes something, and while it runs, break the backend credentials for the final upload (e.g. in another terminal `aws sso logout`, timed right; or temporarily deny `s3:PutObject` on the key with a bucket policy and remove it afterwards).
2. See `errored.tfstate`, fix credentials, `terraform state push errored.tfstate`, plan.

Write a one-paragraph incident note for each drill.

---

## Where you'll use it

- **On-call**: stale locks, failed uploads, and "the pipeline wants to recreate prod" are real pages.
- **Platform migrations**: splitting monolithic states, moving to new backends or to HCP Terraform, replacing providers.
- **Audits**: proving state buckets are versioned, encrypted and restorable (practise restores, like backups).
- **Interviews**: "You deleted the state file. What happens and how do you recover?"

---

## When it breaks

| Symptom | Likely scenario | First move |
|---|---|---|
| Lock error, old timestamp | A | Topic 36 checklist |
| `Failed to persist state`, `errored.tfstate` present | B | Don't apply; fix creds; `state push errored.tfstate` |
| `Failed to load state` / garbage JSON | C | Restore the previous S3 version |
| Plan creates everything | D (or wrong folder/key/workspace) | Check location first; then delete marker/version restore; then import |
| `already exists` during apply after a crash | E | Import the orphan |
| Plan touches the wrong object / refresh fails for one resource | F | `state rm` + import (or removed + import blocks) |
| `unrelated state` / `newer state` on push | G | Stop and check the file and key |
| `newer than current` | H | Upgrade CLI/provider |
| `Failed to query provider ... namespace changed` | I | `state replace-provider` |

---

## Quick recap

- Rules: **freeze, back up (`state pull`), least powerful tool, small steps, plan to verify**.
- S3 **versioning** is your undo button: restore versions with `copy-object`; undelete by removing delete markers.
- `errored.tfstate` = apply succeeded but upload failed → **push it**, don't re-apply.
- Lost state without backups → **import** everything back (topic 39), freezing applies meanwhile.
- `state push` checks lineage and serial; `-force` and hand-edits are last resorts (bump `serial`, keep `lineage`).
- Move resources between states with `removed` (forget) + `import` blocks.

### Interview questions

1. **Q: Your Terraform state file was deleted. What happens and what do you do?**
   A: Terraform thinks nothing exists, so a plan would create duplicates or fail with "already exists". Freeze applies. If the backend is versioned (S3), restore the previous version or remove the delete marker. Otherwise rebuild state by importing every existing resource with import blocks until the plan shows only imports and no changes.
2. **Q: An apply finished but Terraform said it failed to save state. What now?**
   A: Don't re-run apply. Terraform wrote `errored.tfstate` locally. Fix the backend access problem, run `terraform state push errored.tfstate`, then confirm with a plan that shows no changes.
3. **Q: How do you move resources from one Terraform configuration to another?**
   A: Add resource and import blocks in the destination, and a `removed` block with `destroy = false` in the source. Apply the source (forget) then the destination (import), and verify both plans show no changes. The legacy alternative is `state rm` plus `import`.
