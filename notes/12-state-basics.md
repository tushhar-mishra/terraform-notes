# 12 · State: The Basics

> **Level:** ★ foundation
>
> **You will learn:** what the **state file** is, the four reasons Terraform needs it, how to read `terraform.tfstate` field by field (`version`, `serial`, `lineage`, `resources`, `instances`, `attributes`, `dependencies`), the safe ways to look at state (`state list`, `state show`, `show`), why state can hold **secrets**, what the `.backup` file is, and what happens if you **lose** state. Topics 34–42 go much deeper; this is the foundation.
>
> **You need:** topics 07–11.
>
> **Part 1 · First steps and the core loop** · Previous: [11 · Resources and references](11-resources-and-references.md) · Next: [13 · What happens inside `plan` and `apply`](13-inside-plan-and-apply.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** The **state** is Terraform's memory: a JSON file that says "the resource I call `random_pet.server` is the real object with ID `brave-otter`, and these were its attributes last time I looked". Every plan starts from it.

**Picture it.** A cloakroom ticket stub. Your code says "my coat"; the stub says "your coat is hook #217". Lose the stub, and the attendant can't tell which coat is yours. They might give you a new one.

**The one thing to know.** Look at state only through Terraform commands, never edit it by hand, never commit it to Git:

```bash
terraform state list                       # every address Terraform tracks
terraform state show random_pet.server     # everything known about one
terraform show                             # the whole state, human-readable
```

**Try it now:** in your topic 11 lab, `apply`, then `terraform state list`, then `cat terraform.tfstate | jq '.serial, .lineage'`. Apply a change and check `serial` again: it went up by one.

**You've got it if you can:** give two reasons Terraform can't work without state, and explain what happens on the next plan if `terraform.tfstate` is deleted.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

The **state** is a JSON document Terraform writes after every successful `apply` (and `destroy`). By default it's the file `terraform.tfstate` in your working directory (the **local backend**). In teams it lives in a **remote backend** such as an S3 bucket (topic 35).

For each managed resource it stores:

- its **address** (`local_file.motd`),
- which **provider** manages it,
- the real object's **ID** and **all attributes** as last seen,
- its **dependencies** (so Terraform can delete in the right order even after you delete the code),
- the **schema version** the provider used (for upgrades),

plus the **outputs** of the configuration and some bookkeeping (`serial`, `lineage`).

---

## Why it exists

"Why not just look at the cloud each time?" Four reasons:

1. **Mapping code to real objects.** Your code says `aws_instance.web`. The cloud has 400 VMs. Which one is "web"? Tags aren't reliable (anyone can change them; many resource types have no tags). State records the exact ID.
2. **Knowing what to delete.** When you remove a block from the code, Terraform has no code left to look at. Only state remembers that the object exists and belongs to this configuration. (That's the difference from Ansible in topic 02.)
3. **Dependency order for deletion.** If you delete both a network and the VM in it from the code, state still records that the VM depended on the network, so the VM is deleted first.
4. **Performance and caching.** In large setups, state lets Terraform avoid re-discovering everything; it reads only what it already knows about (and with `-refresh=false`, nothing at all, topic 63).

Also, some attributes **can't be read back** from the API later (for example an initial password or a generated private key). State is the only record of them, which is exactly why state can contain secrets.

---

## Analogy

**A cloakroom.**

- Your code = "I own a black coat and a red scarf".
- The cloakroom has hundreds of coats (the cloud).
- The **ticket stubs** (state) say "black coat → hook 217, red scarf → hook 48".
- To swap your scarf for a blue one, the attendant uses the stub to find hook 48.
- If you say "I don't own a scarf any more", the stub tells them which scarf to throw away.
- **Lose the stubs** → the attendant doesn't know which coats are yours. If you ask for "a black coat", they hang up a **new** one; now there are two.
- **Two attendants writing on the same stub at once** → a mess. That's why state needs **locking** (topic 36).

---

## How it works inside

### Where state fits in every run

```
            ┌─────────── terraform plan / apply ───────────┐
 .tf files ─┤                                               │
            │  1. read state (last known world)             │
 state ─────┤  2. refresh: ask providers to re-read each    │
            │     object in state by its ID                 │
            │  3. diff: config vs refreshed state           │
            │  4. (apply) make changes                      │
            │  5. write new state: serial + 1               │
            └───────────────────────────────────────────────┘
```

Note what step 2 does **not** do: Terraform doesn't scan the cloud for things it doesn't already know about. A VM someone created by hand is invisible to Terraform until you **import** it (topic 39).

### The file, field by field

After the topic 11 lab, `cat terraform.tfstate` shows something like this (shortened):

```json
{
  "version": 4,
  "terraform_version": "1.17.2",
  "serial": 3,
  "lineage": "5c1b6e2a-8f43-4d7e-9a61-0f2c3b4d5e6f",
  "outputs": {},
  "resources": [
    {
      "mode": "managed",
      "type": "local_file",
      "name": "motd",
      "provider": "provider[\"registry.terraform.io/hashicorp/local\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "content": "Welcome to brave-otter\nImage:     nginx:1.29\n",
            "directory_permission": "0777",
            "file_permission": "0777",
            "filename": "./motd.txt",
            "id": "9c1f0e7b..."
          },
          "sensitive_attributes": [],
          "dependencies": [
            "random_pet.server"
          ]
        }
      ]
    },
    {
      "mode": "managed",
      "type": "random_password",
      "name": "db",
      "provider": "provider[\"registry.terraform.io/hashicorp/random\"]",
      "instances": [
        {
          "schema_version": 3,
          "attributes": {
            "id": "none",
            "length": 20,
            "result": "q9!Lx2#vT...",
            "special": true
          },
          "sensitive_attributes": [
            [ { "type": "get_attr", "value": "result" } ]
          ]
        }
      ]
    }
  ],
  "check_results": null
}
```

| Field | Meaning |
|---|---|
| `version: 4` | The **state file format** version (not Terraform's). Has been 4 since Terraform 0.12. |
| `terraform_version` | The Terraform version that last wrote it. Older Terraform versions may refuse to read state written by newer ones. |
| `serial` | A counter that goes **up by 1** every time state changes. Used to detect stale plans (topic 09) and to stop an old copy overwriting a newer one. |
| `lineage` | A random ID created when this state was first made. Two files with different lineages are **different states**, even for the same code. Terraform refuses to overwrite one with the other. |
| `outputs` | Values of `output` blocks (topic 15). |
| `mode` | `managed` = a `resource`; `data` = a data source (topic 19). |
| `type`, `name` | Together, the address. |
| `provider` | Which provider configuration manages it (including aliases like `provider["..."].us`). |
| `instances` | One entry normally; several with `count`/`for_each` (each with an `index_key`). |
| `schema_version` | The provider's schema version for this type; the provider upgrades old entries automatically. |
| `attributes` | **Every** argument and attribute, as last seen. |
| `sensitive_attributes` | Which attributes are sensitive. **They're still stored here in plain text.** Marking only hides them from CLI output. |
| `dependencies` | Addresses this object depended on when it was created. |

### The backup file

Every time the local backend writes state, it first copies the previous version to `terraform.tfstate.backup`. It's **one** step of history only. Remote backends use **versioning** (e.g. S3 object versions) for real history (topic 35).

---

## Code / commands

### `terraform state list`

```
$ terraform state list
local_file.motd
random_password.db
random_pet.server
```

Filter by address prefix: `terraform state list random_pet` → only `random_pet.server`.

### `terraform state show`

```
$ terraform state show random_password.db
# random_password.db:
resource "random_password" "db" {
    bcrypt_hash      = (sensitive value)
    id               = "none"
    length           = 20
    lower            = true
    ...
    result           = (sensitive value)
    special          = true
}
```

Hidden in the CLI. But:

```bash
jq -r '.resources[] | select(.type=="random_password") | .instances[0].attributes.result' terraform.tfstate
```

```
q9!Lx2#vT...
```

**Anyone who can read the state file can read every secret in it.** That's why state goes in an encrypted, access-controlled backend, never in Git (topics 35, 51).

### `terraform show`

```bash
terraform show            # whole state, human-readable
terraform show -json | jq '.values.root_module.resources[].address'
```

`show -json` is the **supported** machine-readable format. Scripts should use it rather than parsing `terraform.tfstate` directly, because the raw file's details may change between versions.

### Watching serial change

```bash
jq '.serial' terraform.tfstate         # 3
# change something in main.tf, then:
terraform apply -auto-approve
jq '.serial' terraform.tfstate         # 4
jq '.serial' terraform.tfstate.backup  # 3
```

### What losing state looks like

```bash
mv terraform.tfstate /tmp/state-hidden
terraform plan
```

```
Terraform will perform the following actions:

  # local_file.motd will be created
  ...
  # random_password.db will be created
  ...
  # random_pet.server will be created
  ...
Plan: 3 to add, 0 to change, 0 to destroy.
```

Terraform has no memory, so it thinks nothing exists. Here, applying would give a **new** pet name and password and overwrite `motd.txt`. In the cloud it would be worse: **duplicate** VMs, or `already exists` errors for things with unique names (buckets, DNS records), and the old objects become **orphans** nobody manages.

Put it back: `mv /tmp/state-hidden terraform.tfstate`, then `terraform plan` → `No changes.`

---

## Hands-on lab

Use your `~/tf-lab/11-resources` project (re-apply it if you destroyed it).

1. `terraform state list`, then `terraform state show` for each address.
2. Read the raw file: `jq '{version, terraform_version, serial, lineage}' terraform.tfstate`.
3. Find the dependency: `jq '.resources[] | {addr: "\(.type).\(.name)", deps: .instances[0].dependencies}' terraform.tfstate`.
4. Prove secrets are in plain text: extract `random_password.db`'s `result` with `jq`.
5. Watch `serial` go up after a change, and compare it with the `.backup` file.
6. Simulate **lost state** (move the file away), run `terraform plan`, read it, **don't apply**, and put the file back.
7. Simulate **drift**: change `motd.txt` by hand, run `terraform plan`, and see how state (what Terraform last saw) and reality differ.
8. Leave the project applied if you want to keep using it in topic 13; otherwise `terraform destroy`.

---

## Where you'll use it

- **Every incident**: "what does Terraform think exists?" → `state list` / `state show`.
- **Scripts and CI**: `terraform show -json` and `terraform output -json` instead of hand-parsing.
- **Security reviews**: "where is our state, who can read it, is it encrypted?" State is one of the most sensitive files in a company.
- **Interviews**: "What is Terraform state and why is it needed?" is almost guaranteed.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| Plan wants to create everything that exists | State is missing, or you're pointing at a different/empty state (wrong folder, wrong backend, or wrong **workspace**, which is a separate state for the same code, topic 48). | **Don't apply.** Find the right state. If it's truly lost, import (topics 39, 42). |
| `Error: state snapshot was created by Terraform v1.18.0, which is newer than current v1.17.2` | Someone applied with a newer Terraform. | Upgrade your Terraform. Agree on one version (topic 06). |
| `Error: Error acquiring the state lock` | Another run holds the lock, or a crashed run left one. | Topic 36. |
| State file committed to Git | Secrets leaked to everyone with repo access, forever in history. | Remove it, move state to a backend, **rotate every secret** in it. |
| Hand-edited state, now `Error: Failed to load state` / `unsupported attribute` | The JSON is invalid or doesn't match the schema. | Restore `terraform.tfstate.backup` or a backend version (topic 42). Use `state` commands instead (topic 42). |

---

## Quick recap

- State = Terraform's memory: address → real ID + attributes + dependencies, plus outputs.
- It's needed for mapping, deleting removed resources, deletion order and performance.
- `serial` counts changes; `lineage` identifies one state's history.
- **Sensitive values are stored in plain text** in state; protect it, never commit it.
- Use `state list`, `state show`, `show -json`; never edit by hand. Lost state = Terraform thinks nothing exists.

### Interview questions

1. **Q: What is Terraform state and why is it needed?**
   A: A JSON record mapping each resource in the configuration to the real object's ID and attributes, with dependency and output data. It lets Terraform know what it manages, compute diffs, delete resources removed from code in the right order, and plan efficiently.
2. **Q: Is it safe to store state in Git?**
   A: No. It can contain secrets in plain text, and Git has no locking, so concurrent changes can corrupt it. Use a remote backend with encryption, access control, versioning and locking.
3. **Q: What are `serial` and `lineage`?**
   A: `serial` is a version number incremented on every state write; `lineage` is a unique ID for a state's history. Terraform uses them to refuse stale plans and stop a different or older state overwriting the current one.
