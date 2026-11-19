# 41 · Replacing Resources on Purpose

> **Level:** ★★ intermediate
>
> **You will learn:** how to **force** Terraform to destroy and recreate a resource whose configuration hasn't changed, with **`terraform apply -replace=ADDRESS`**: when you'd want it, how it shows in plans, how to combine it with saved plans and `create_before_destroy`; what **tainted** resources are, how they happen (failed creates) and how to clear them with `untaint`; and why ⚠️ **`terraform taint`** is deprecated. Plus a recap table of every reason a resource can be replaced.
>
> **You need:** topics 13 (partial apply, tainted), 24 (replacement causes, `create_before_destroy`, `replace_triggered_by`), 26 (Docker provider), 33 (`user_data`).
>
> **Part 5 · State in depth** · Previous: [40 · Refactoring: `moved` and `removed` blocks](40-moved-and-removed.md) · Next: [42 · State surgery and recovery](42-state-recovery.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Sometimes a resource is **broken in a way Terraform can't see**: a VM with a corrupted disk, a container in a bad state, a server whose boot script you want to re-run. Its config matches, so Terraform says "No changes". `-replace` forces a fresh one:

```bash
terraform plan  -replace='aws_instance.web'       # preview
terraform apply -replace='aws_instance.web'       # destroy + recreate it (only it)
```

```
  # aws_instance.web will be replaced, as requested
-/+ resource "aws_instance" "web" { ... }
```

**Picture it.** Your car runs badly but nothing on the dashboard says so. You tell the garage: "Replace it with an identical new one, even though the paperwork says it's fine."

**The one thing to know.** `-replace` is a **plan option**: it goes through plan and review. ⚠️ The old `terraform taint` marked the resource in **state immediately** for the next apply by anyone: no preview, easy to forget. Use `-replace`.

**Try it now:** `terraform apply -replace='docker_container.web'` on your topic 26 lab, and watch the container ID change.

**You've got it if you can:** explain the difference between `-replace` and `taint`, and what a tainted resource is.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### `-replace=ADDRESS`

- A flag for `plan` and `apply` (Terraform 0.15.2+).
- Plans a **replacement** of that resource instance, even with no config changes.
- Can be repeated: `-replace=A -replace=B`.
- Works with `count`/`for_each` addresses: `-replace='aws_instance.web[1]'`, `-replace='docker_container.app["b"]'` (quote for the shell).
- Respects `lifecycle`: `create_before_destroy` makes it `+/-`; `prevent_destroy` **blocks** it.
- Works with saved plans: `terraform plan -replace=X -out=tfplan` → review → `terraform apply tfplan`.
- Dependent resources are updated (or replaced, if their force-new arguments use the replaced resource's new ID, topic 24's cascade).

### Tainted resources

A resource is **tainted** when Terraform **created** it, but the create didn't finish successfully. For example, an instance was launched but the provider then timed out waiting for it to become ready, or a provisioner (topic 52) failed. Terraform can't trust it, so it:

- saves it in state with `"status": "tainted"`,
- shows `(tainted)` in the header of `terraform state show ADDRESS`, and
- plans to **replace** it on the next apply: `is tainted, so must be replaced`.

If you know the object is fine (e.g. the timeout was a slow API, and the thing is healthy now), clear the mark: `terraform untaint ADDRESS`.

### ⚠️ `terraform taint` (deprecated since 0.15.2)

`terraform taint ADDRESS` marked a resource as tainted **in state, right now**. Problems:

- It changed **shared state** immediately, with no plan and no review.
- The **next apply by anyone** (maybe a teammate's unrelated change, maybe CI) would replace it, possibly at a bad moment.
- Forgotten taints caused surprise replacements.

`-replace` gives the same result **inside a reviewed plan**. `terraform untaint` is still useful for clearing taints that Terraform itself set.

### Every reason Terraform replaces something (recap)

| Plan says | Cause | Topic |
|---|---|---|
| `# forces replacement` on an argument | A changed argument can't be updated in place | 24 |
| `-> (known after apply) # forces replacement` | An upstream value is unknown (cascade) | 11, 24 |
| `is tainted, so must be replaced` | A failed create | 13, this topic |
| `will be replaced, as requested` | `-replace` | this topic |
| `will be replaced due to changes in replace_triggered_by` | A trigger changed | 24 |
| Destroy at one address + create at another | Renamed/moved in code → you need `moved` | 40 |

---

## Why it exists

Terraform compares **configuration** with **recorded attributes**. Many failures live outside those attributes: disk corruption, a hung process, a half-applied boot script, a node that needs a fresh start, a credential baked into an image that must be regenerated. You need a way to say "rebuild this one" that's as safe and reviewable as any other change.

---

## Analogy

**A car that runs badly.**

- The paperwork (state) says: "blue car, 1.6 L engine". The car is blue with a 1.6 L engine (config matches), so the garage says "nothing to do".
- But it stalls every morning. You say: **"replace it with an identical new one"** (`-replace`). The garage writes up the job, you sign it (plan → apply).
- **Taint** (old way) = putting a sticky note on the car's file saying "replace on next service". Whoever does the next service, even for a tyre change, also swaps the whole car. Maybe on the busiest day.
- A **tainted** car (failed create) = the factory delivered it but the final inspection never happened. The garage won't trust it until it's inspected or replaced.

---

## How it works inside

```
terraform plan -replace=docker_container.web
  normal plan walk; when it reaches docker_container.web:
    provider plans "no-op" (config unchanged)
    core sees the address in the -replace list → converts the action to "replace"
      lifecycle.create_before_destroy? → CreateThenDelete (+/-) : DeleteThenCreate (-/+)
      lifecycle.prevent_destroy?       → error
  resources that depend on its new ID are re-planned (unknown → maybe cascade)

failed create:
  provider returns an error together with a partial new object
  → core saves the object with status "tainted"
  → next plan: action = replace ("is tainted, so must be replaced")
terraform untaint ADDR: removes the "tainted" status in state (takes the lock; no plan)
```

---

## Code / commands

Use the topic 26 Docker lab (`~/tf-lab/26-docker`, re-apply it if needed).

### `-replace`

```bash
docker ps --filter name=tf-web --format '{{.ID}} {{.Status}}'
terraform plan -replace='docker_container.web'
```

```
a1b2c3d4e5f6 Up 2 hours (healthy)

docker_container.web: Refreshing state... [id=a1b2c3d4e5f6...]

Terraform will perform the following actions:

  # docker_container.web will be replaced, as requested
-/+ resource "docker_container" "web" {
      ~ id                = "a1b2c3d4e5f6..." -> (known after apply)
      ~ network_data      = [ ... ] -> (known after apply)
        name              = "tf-web"
        # (many unchanged attributes hidden)
    }

Plan: 1 to add, 0 to change, 1 to destroy.
```

- `will be replaced, as requested`: the reason is your flag, not a config change.
- `-/+`: destroy first (default).

```bash
terraform apply -replace='docker_container.web' -auto-approve
docker ps --filter name=tf-web --format '{{.ID}} {{.Status}}'
```

```
9f8e7d6c5b4a Up 5 seconds (health: starting)
```

New container ID; the data in the **volume** survived (topic 26).

With a saved plan, the normal production flow:

```bash
terraform plan -replace='docker_container.web' -out=tfplan
terraform show tfplan | grep -A1 'replaced'
terraform apply tfplan
```

### `-replace` with `prevent_destroy`

If the resource has `lifecycle { prevent_destroy = true }`:

```
│ Error: Instance cannot be destroyed
│ ...
│ Resource docker_container.web has lifecycle.prevent_destroy set, but the plan calls
│ for this resource to be destroyed.
```

The safety catch applies to forced replacements too.

### A tainted resource

Make a container that exits right away, so the provider can't confirm it's running:

```hcl
resource "docker_container" "broken" {
  name     = "tf-broken"
  image    = docker_image.nginx.image_id
  command  = ["sh", "-c", "exit 1"]     # dies immediately
  must_run = true                       # the provider checks it's running after create
}
```

```bash
terraform apply -auto-approve
```

```
docker_container.broken: Creating...
╷
│ Error: container exited immediately
│
│   with docker_container.broken,
│   on main.tf line 60, in resource "docker_container" "broken":
│   60: resource "docker_container" "broken" {
╵
```

```bash
terraform state list
terraform plan
```

```
docker_container.broken
docker_image.nginx
...

$ terraform state show docker_container.broken | head -1
# docker_container.broken: (tainted)

  # docker_container.broken is tainted, so must be replaced
-/+ resource "docker_container" "broken" {
...
```

- The container **was created** (it exists, stopped), so it's in state, marked tainted.
- The next plan replaces it. Fix the command (e.g. remove the `command` line), apply: the tainted one is destroyed and a good one created.

If a resource was tainted by a **transient** error and is actually fine:

```bash
terraform untaint 'docker_container.broken'
```

```
Resource instance docker_container.broken has been successfully untainted.
```

### The deprecated way (so you recognise it)

```bash
terraform taint 'docker_container.web'
```

```
Resource instance docker_container.web has been marked as tainted.
```

Nothing was planned or reviewed, and the next `apply` by anyone replaces it. Undo it now: `terraform untaint 'docker_container.web'`.

### On AWS: re-running `user_data`

In topic 33, if you'd left out `user_data_replace_on_change`, the fix for "my new boot script didn't run" is:

```bash
terraform apply -replace='aws_instance.web'
```

A fresh instance runs the new `user_data` at first boot.

---

## Hands-on lab

1. In the topic 26 lab, `plan` and `apply` with `-replace` on the container. Compare container IDs before and after.
2. Do it as `plan -out` → `show` → `apply tfplan`.
3. Add `prevent_destroy = true` and try `-replace`. Remove it.
4. Create the `broken` container, see it tainted, then fix it by correcting the config. Then recreate the taint and clear it with `untaint` instead.
5. Use `terraform taint` once, see the change in `terraform state show`, then `untaint`. Note how easy it would be to forget.
6. On a `for_each` resource (topic 27's `docker_container.app`), replace one instance: `-replace='docker_container.app["b"]'`. Which other resources change (think `replace_triggered_by` on the proxy)?
7. `terraform destroy`.

---

## Where you'll use it

- **Broken or compromised instances**: replace instead of repairing by hand.
- **Re-running boot logic**: `user_data`, cloud-init, or anything that only runs at creation.
- **Rotating generated values**: `-replace=random_password.db` (then the database password must also change: plan carefully).
- **Rolling a single node** of a group, one at a time.
- **Recovering from failed creates**: tainted resources after timeouts.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `Error: Invalid replace address` / no such resource | Typo or missing shell quotes around `["key"]`. | `terraform state list` for the exact address; quote it. |
| `Instance cannot be destroyed` | `prevent_destroy` blocks the replacement. | Intended protection; remove it in a reviewed PR if replacement is truly wanted. |
| `-replace` replaced more than expected | Dependents use its ID in force-new arguments (cascade). | Read the whole plan; use `create_before_destroy` / references that don't force replacement. |
| A resource replaced "for no reason" during someone's apply | A leftover `terraform taint`, or a real tainted state from a failed create. | `terraform state show` shows `(tainted)`; decide replace or `untaint`. Stop using `taint`. |
| Replacement causes downtime | Default `-/+` destroys first. | `create_before_destroy` with unique names, behind a load balancer. |
| Tainted again after every apply | The create keeps failing the same way (bad command, failing health check). | Read the error; fix the config, not the taint. |

---

## Quick recap

- `terraform plan/apply -replace=ADDR` forces a replacement inside a **reviewed plan**; respects `create_before_destroy` and `prevent_destroy`.
- **Tainted** = created but not completed successfully; replaced on next apply; clear with `untaint` if it's fine.
- ⚠️ `terraform taint` is deprecated: it changed shared state immediately without review.
- Replacements come from force-new changes, unknown upstream values, taint, `-replace`, or `replace_triggered_by`.

### Interview questions

1. **Q: How do you force Terraform to recreate a resource that has no config changes?**
   A: `terraform apply -replace=<address>` (or plan with `-replace` and apply the saved plan). It replaces that instance as part of a normal reviewed plan.
2. **Q: Why was `terraform taint` deprecated?**
   A: It modified state immediately without a plan, so the replacement happened on whoever's next apply, unreviewed and possibly at a bad time. `-replace` shows the replacement in the plan you approve.
3. **Q: What does "tainted" mean?**
   A: Terraform created the object but the creation didn't complete successfully (e.g. a timeout or failed provisioner), so it's recorded as tainted and will be replaced on the next apply. `terraform untaint` clears it if you've verified the object is fine.
