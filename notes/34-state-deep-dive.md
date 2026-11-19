# 34 · State Deep Dive

> **Level:** ★★ intermediate
>
> **You will learn:** state beyond the basics: the **three worlds** Terraform compares (configuration, state, reality) and every combination of disagreement between them; what **refresh** really does and the three refresh modes (`plan`, `-refresh-only`, `-refresh=false`); ⚠️ why `terraform refresh` is deprecated; how state records **resource instances, data sources, outputs, check results and deposed objects**; **schema versions** and state upgrades; **`serial` and `lineage`** as concurrency guards; what's **sensitive** in state; and the read-only state commands you'll use in every investigation.
>
> **You need:** topics 12 (state basics), 13 (plan/apply trace), 21–22 (instance keys), 24 (deposed objects).
>
> **Part 5 · State in depth** · Previous: [33 · Project 2: a web server on AWS](33-project-2-web-server-on-aws.md) · Next: [35 · Remote backends](35-remote-backends.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Terraform always juggles **three versions of the truth**:

```
   CONFIGURATION            STATE                    REALITY
   (.tf files: what         (terraform.tfstate:      (the real objects in
    you WANT)                what Terraform LAST      AWS/Docker/... right
                             SAW and did)             now)
```

`plan` = refresh **state** from **reality**, then compare **configuration** with that refreshed state.

| Mode | Refresh from reality? | Compare with config? | Writes state? |
|---|---|---|---|
| `terraform plan` | Yes | Yes | No |
| `terraform plan -refresh-only` | Yes | **No** (just shows drift) | No |
| `terraform apply -refresh-only` | Yes | No | **Yes**: accept reality into state |
| `terraform plan -refresh=false` | **No** (trusts state) | Yes | No |

**Picture it.** A shop's stock: the **order sheet** (config), the **stock book** (state) and the **shelves** (reality). Each morning the manager counts the shelves (refresh), updates a copy of the stock book, and works out what to order.

**The one thing to know.** `terraform refresh` is deprecated: it silently overwrote state with reality. Use `terraform apply -refresh-only`, which **shows you** first.

**Try it now:** in a lab, change something by hand, run `terraform plan -refresh-only`, read it, then `terraform apply -refresh-only`.

**You've got it if you can:** say what Terraform does when a resource is (a) in config but not state, (b) in state but not config, (c) in state but deleted in reality.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### The three worlds and their disagreements

| Config | State | Reality | What `plan` proposes | Why |
|---|---|---|---|---|
| ✅ | ✅ | ✅ same | Nothing | All agree |
| ✅ | ✅ | ✅ different | **Update/replace** to match config (and shows "changed outside of Terraform") | Drift |
| ✅ | ✅ | ❌ gone | **Create** (refresh removes it from state first) | Someone deleted it |
| ✅ | ❌ | ❌ | **Create** | New resource |
| ✅ | ❌ | ✅ exists | **Create** → probably fails with "already exists" | Terraform can't see unmanaged objects → **import** (topic 39) |
| ❌ | ✅ | ✅ | **Destroy** | Removed from code → use `removed` (topic 40) if you only want to stop managing it |
| ❌ | ✅ | ❌ gone | Nothing (refresh drops it) | Already gone |
| ❌ | ❌ | ✅ | Nothing | Unmanaged; Terraform doesn't scan for it |

Two rows cause most production surprises:

- **Config + reality but no state** → Terraform tries to create a duplicate.
- **State + reality but no config** → Terraform deletes a real thing.

### Refresh

**Refresh** = for each resource **in state**, call the provider's **Read** with its ID and replace the stored attributes with what the API returns. In memory only, during `plan`.

- It reads **only what's in state**. It never discovers new objects.
- If Read says "not found", the resource is dropped from the refreshed state (the plan then proposes to create it, if it's still in config).
- `-refresh=false` skips it: faster on huge configurations, but **blind to drift**. Use only when you know nothing changed outside Terraform (topic 63).
- `-refresh-only` shows **only** the drift (no config comparison) and, with `apply`, writes it into state. Use it to **accept** an intentional out-of-band change, or to update outputs.
- ⚠️ `terraform refresh` (the old command) = `apply -refresh-only -auto-approve`: no review. If credentials pointed at the wrong account/region, it would "refresh" every resource as deleted and wipe them from state. Deprecated for that reason.

### What's in a state file (complete picture)

| Section | Contents |
|---|---|
| Header | `version` (format, 4), `terraform_version`, `serial`, `lineage` |
| `outputs` | Root module outputs with values, types and sensitive flags |
| `resources[]` | One entry per resource **block** (and per data source), with `module` path if inside a module, `mode` (`managed`/`data`), `type`, `name`, `provider` |
| `resources[].instances[]` | One per instance: `index_key` (for `count`/`for_each`), `schema_version`, `attributes`, `sensitive_attributes`, `private` (provider-private data, base64), `dependencies`, `create_before_destroy`, `status: "tainted"` if tainted, `deposed` key for deposed objects |
| `check_results` | Last results of checks/conditions (for `check` blocks and HCP Terraform health) |

A resource inside a module:

```json
{
  "module": "module.network",
  "mode": "managed",
  "type": "aws_subnet",
  "name": "public",
  "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
  "instances": [
    { "index_key": "ap-south-1a", "schema_version": 1, "attributes": { "id": "subnet-0a3b..." } }
  ]
}
```

Its address is `module.network.aws_subnet.public["ap-south-1a"]`.

### Schema versions and state upgrades

Providers change their resource schemas over time. Each instance records the `schema_version` it was written with. When a newer provider reads an older entry, it runs an **upgrade function** and Terraform writes the upgraded form on the next apply. This is automatic and one-way: after upgrading a provider and applying, **older providers may not read the state** any more. Same for Terraform itself: a newer CLI may write state an older CLI rejects. That's why versions are pinned and upgraded together (topic 62).

### `serial` and `lineage` as guards

- **`lineage`**: a UUID (Universally Unique Identifier, a random ID) created when a state is born. Terraform refuses to overwrite a state with one of a **different lineage** (protects you from pushing state from project A over project B).
- **`serial`**: increments on every write. Backends and `state push` refuse to replace a state with a **lower** serial (protects you from an old copy overwriting a newer one). Saved plans record it, which is how "Saved plan is stale" works.

### What's sensitive in state

**Everything Terraform knows**, in plain text: generated passwords (`random_password`), database master passwords, private keys (`tls_private_key`), secrets read by data sources, and `user_data` (hashed for EC2, but not everywhere). `sensitive` only hides values in CLI output. Protections (topics 35, 51): an encrypted backend with tight access; ephemeral values and write-only arguments so secrets never land in state; OpenTofu's client-side state encryption.

---

## Why it exists

Terraform could, in theory, rebuild its understanding from the APIs every time. It can't in practice: many objects have no reliable way to find "the one this code means", some attributes can't be read back, and large estates would take forever to scan. State is Terraform's **memory**; refresh is its **reality check**. Understanding both, and the eight combinations above, is what lets you predict a plan before you run it.

---

## Analogy

**A shop's stock control.**

- **Order sheet** (config): "we should stock 20 red mugs, 10 blue plates".
- **Stock book** (state): "last count: 20 red mugs (shelf A3), 10 blue plates (shelf B1)".
- **Shelves** (reality).
- Each morning the manager walks to **only the shelves listed in the book** and recounts them (refresh). Anything on a shelf not in the book is invisible (unmanaged objects).
- `plan` = recount, then compare the order sheet with the recount: "need 5 more mugs".
- `-refresh-only` = "just update the book to what's on the shelves; don't order anything", shown to you first.
- ⚠️ The old `refresh` = updating the book blindly. If the manager walked into the **wrong shop**, they'd write "0 of everything" in your book.

---

## How it works inside

```
terraform plan
  ├─ load prior state  (from the backend; serial N)
  ├─ for each resource instance in prior state:
  │     provider.ReadResource(prior attributes, private) → current attributes / "gone"
  │     → "refreshed state" (in memory)                 → diff vs prior = DRIFT report
  ├─ for each resource in config (+ orphans in state):
  │     provider.PlanResourceChange(refreshed state, config) → planned state + action
  └─ print: drift notes ("changed outside of Terraform"), then the actions
     (state on disk/backend unchanged: still serial N)

terraform apply -refresh-only
  ├─ same refresh
  ├─ plan = "update state to refreshed values" (no resource actions)
  └─ approve → write state (serial N+1)
```

---

## Code / commands

Use your topic 27 Docker stack, or this small lab (`~/tf-lab/34-state/`, providers `docker`, `random`):

```hcl
resource "docker_image" "nginx" {
  name         = "nginx:1.29"
  keep_locally = true
}

resource "docker_container" "web" {
  for_each = toset(["a", "b"])
  name     = "state-lab-${each.key}"
  image    = docker_image.nginx.image_id
  labels {
    label = "owner"
    value = "terraform"
  }
}

output "containers" {
  value = [for c in docker_container.web : c.name]
}
```

```bash
terraform init && terraform apply -auto-approve
```

### Read-only state commands

```bash
terraform state list
terraform state list 'docker_container.web'
terraform state show 'docker_container.web["a"]'
terraform show -json | jq '.values.root_module.resources[] | {address, type, id: .values.id}'
terraform state pull | jq '{serial, lineage, terraform_version}'
```

```
docker_container.web["a"]
docker_container.web["b"]
docker_image.nginx
...
{
  "serial": 4,
  "lineage": "8e2f...",
  "terraform_version": "1.17.2"
}
```

- `state list ADDRESS-PREFIX` filters.
- `terraform state pull` prints the **raw** state from whatever backend is configured (local or remote). It's the safe way to back up or inspect remote state: `terraform state pull > backup-$(date +%F-%H%M).tfstate`.
- `terraform show -json` gives the stable, documented JSON for scripts.

### The eight combinations, live

**Reality deleted (config ✅ state ✅ reality ❌):**

```bash
docker rm -f state-lab-a
terraform plan
```

```
Note: Objects have changed outside of Terraform
  # docker_container.web["a"] has been deleted
...
  # docker_container.web["a"] will be created
Plan: 1 to add, 0 to change, 0 to destroy.
```

**Unmanaged twin (config ✅ state ❌ reality ✅):** add `"c"` to the set, but first create the container yourself:

```bash
docker run -d --name state-lab-c nginx:1.29
terraform apply -auto-approve
```

```
│ Error: Unable to create container: Error response from daemon: Conflict. The container
│ name "/state-lab-c" is already in use by container "4f1e...".
```

Terraform didn't know it existed. Fix: delete it, or **import** it (topic 39).

**Out-of-band change you want to keep:** run `terraform apply -refresh-only` after changing something that doesn't conflict with config (for example, drift in an attribute you then add to `ignore_changes`, or a resource where you'll update config to match).

```
$ terraform apply -refresh-only
Note: Objects have changed outside of Terraform
  # docker_container.web["b"] has changed
  ~ resource "docker_container" "web" {
      ~ ...
    }

This is a refresh-only plan, so Terraform will not take any actions to undo
these. If you were expecting these changes then you can apply this plan to
record the updated values in the Terraform state without changing any remote
objects.

Would you like to update the Terraform state to reflect these detected changes?
  Enter a value: yes

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
```

- `0 added, 0 changed, 0 destroyed` but `serial` went up: only state changed.

**Skip refresh:**

```bash
docker rm -f state-lab-b
terraform plan -refresh=false
```

```
No changes. Your infrastructure matches the configuration.
```

**Wrong!** The container is gone, but Terraform trusted state. That's the danger of `-refresh=false`. A normal `plan` shows the truth.

### Watching serial and the backup

```bash
terraform state pull | jq .serial                # e.g. 6
terraform apply -auto-approve                    # fix the drift
terraform state pull | jq .serial                # 7
jq .serial terraform.tfstate.backup              # 6
```

---

## Hands-on lab

1. Build the lab. Run every read-only command; save a backup with `state pull`.
2. Reproduce at least five of the eight combinations from the table (delete in reality; create an unmanaged twin; remove a key from config; add a key; change a label by hand).
3. Run `terraform plan -refresh-only` after each manual change. Compare its output with a normal plan.
4. Show the `-refresh=false` blind spot.
5. Find the `dependencies` of `docker_container.web["a"]` in the raw state (`state pull | jq`).
6. Clean up: `terraform destroy`; `docker rm -f state-lab-c` if it still exists.

---

## Where you'll use it

- **Every "why does the plan want to do that?" investigation** starts with the three-worlds table.
- **Accepting an emergency console change**: update config to match, or `apply -refresh-only` so state reflects it, so the next plan is clean.
- **Backups before risky operations**: `terraform state pull > backup.tfstate` (topic 42).
- **Upgrades**: understanding schema versions explains why provider downgrades fail (topic 62).

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| Plan wants to destroy things you didn't touch | They're in state but not in config (deleted/renamed blocks, wrong folder, a teammate's unmerged branch applied). | Don't apply. Check `git log`, the folder, the workspace. Use `moved`/`removed` (topic 40). |
| Plan wants to create things that exist | Not in state: lost state, wrong backend/key, or never imported. | Find the right state (topic 35) or import (topic 39). |
| `-refresh-only` shows everything "deleted" | Wrong credentials/account/region: the provider can't see your resources. | **Don't apply.** Fix credentials (`allowed_account_ids`, topic 30). |
| `Resource instance managed by newer provider version` | State was written by a newer provider than you have. | Upgrade the provider to at least that version. |
| `state snapshot was created by Terraform v1.18.x, which is newer than current v1.17.x` | Newer CLI wrote it. | Upgrade your CLI. Pin versions in CI. |
| `Failed to persist state` / `Error saving state` | Backend write failed after apply (network, permissions). | Terraform writes `errored.tfstate` locally; push it (topic 42). Don't lose that file. |

---

## Quick recap

- Three worlds: **config** (want), **state** (last seen), **reality** (now). `plan` refreshes state from reality, then diffs config against it.
- Refresh reads only resources **in state**; it never discovers new ones.
- `plan -refresh-only` shows drift; `apply -refresh-only` records it; `-refresh=false` is fast but blind; ⚠️ `terraform refresh` is deprecated.
- State holds instances, data sources, outputs, checks, deposed objects, schema versions, **and secrets in plain text**.
- `lineage` and `serial` stop wrong or older states overwriting newer ones.

### Interview questions

1. **Q: What does refresh do, and why is `terraform refresh` deprecated?**
   A: Refresh re-reads each resource in state from the provider to get current attributes and detect drift. The standalone command wrote the result to state without review; with wrong credentials it could drop everything from state. `terraform apply -refresh-only` shows the changes and asks first.
2. **Q: A resource exists in AWS and in your code, but not in state. What does `plan` do?**
   A: It plans to create it, which usually fails with "already exists" (or creates a duplicate). The fix is to import the existing object into state with an `import` block.
3. **Q: What are `serial` and `lineage` used for?**
   A: `lineage` identifies a state's history so a different state can't overwrite it; `serial` is a version counter so an older copy can't overwrite a newer one, and saved plans can detect staleness.
