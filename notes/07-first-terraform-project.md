# 07 · Your First Terraform Project

> **Level:** ★ foundation
>
> **You will learn:** the full Terraform life cycle on a tiny, free project: write → `init` → `plan` → `apply` → change → `apply` again → see drift → `destroy`. You'll read every line of output, see which files Terraform creates on disk, and meet your first **replacement** (`-/+`). Later topics explain each piece in depth; this one gives you the feel.
>
> **You need:** topics 03 (core, providers, state), 05 (`.gitignore`), 06 (Terraform installed).
>
> **Part 1 · First steps and the core loop** · Previous: [06 · Lab setup](06-lab-setup.md) · Next: [08 · HCL syntax](08-hcl-syntax.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A Terraform project is just **a folder of `.tf` files**. You run commands **inside that folder**. The `local` provider manages files on your own disk, so you can learn the whole workflow without any cloud account.

**Picture it.** A dress rehearsal on an empty stage: same script and moves as the real show, but nothing expensive can break.

**The one thing to know.**

```bash
mkdir -p ~/tf-lab/07-first && cd ~/tf-lab/07-first
cat > main.tf <<'EOF'
resource "local_file" "hello" {
  filename = "${path.module}/hello.txt"
  content  = "Hello, Terraform!\n"
}
EOF
terraform init       # downloads the "local" provider
terraform plan       # "+ create" 1 resource
terraform apply      # type: yes
cat hello.txt
terraform destroy    # type: yes
```

**Try it now:** run it all. Before `destroy`, change the content, run `terraform apply` again and notice the `-/+` symbol.

**You've got it if you can:** name the three things that appear in the folder after `init` and `apply` (`.terraform/`, `.terraform.lock.hcl`, `terraform.tfstate`) and say what each is.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

A **Terraform working directory** (also called the **root module**; "module" is explained in topic 43) is the folder you run `terraform` in. Terraform reads **every `*.tf` file in that folder** (not sub-folders) as one configuration.

Our project manages one resource of type `local_file`, from the **`hashicorp/local`** provider. A **resource** is one thing Terraform manages: here a file, later a container, a VM or a database (topic 11).

We didn't write a `terraform { required_providers { ... } }` block. For providers in the `hashicorp/` namespace, Terraform guesses the source from the resource's prefix (`local_` → `hashicorp/local`). That's fine for a first lab, but real projects always declare providers explicitly (topic 10), and so will these notes from the next topic on.

---

## Why it exists

Before touching a cloud bill, you want to *feel* the loop: what each command prints, what goes on disk, what "replace" looks like, what happens when reality changes behind Terraform's back. The local provider makes mistakes free.

---

## Analogy

Learning to drive in an empty car park. Same pedals, same mirrors, same moves as on the motorway, but if you get it wrong, nobody gets hurt.

---

## How it works inside

Each command, in one line (topic 13 has the full trace):

| Command | What happens |
|---|---|
| `init` | Reads the `.tf` files to find required providers → downloads `hashicorp/local` into `.terraform/` → records the exact version and checksums in `.terraform.lock.hcl`. |
| `plan` | Loads state (none yet) → asks the provider "what exists?" → compares with config → prints the difference. Changes **nothing**. |
| `apply` | Makes a fresh plan → asks you to confirm → tells the provider to create the file → writes `terraform.tfstate`. |
| `destroy` | Plans to delete **everything in the state** → asks you → deletes → state becomes empty. |

Files on disk after each step:

```
after init:                    after apply:
07-first/                      07-first/
├── main.tf                    ├── main.tf
├── .terraform.lock.hcl  ◀ commit   ├── .terraform.lock.hcl
└── .terraform/          ◀ ignore   ├── .terraform/
    └── providers/                  │   └── providers/registry.terraform.io/hashicorp/local/2.5.3/linux_amd64/
        └── ...                     │         terraform-provider-local_v2.5.3_x5   ◀ the plugin binary
                                    ├── hello.txt                ◀ the thing we manage
                                    └── terraform.tfstate         ◀ never commit
```

During `apply` you may also see `.terraform.tfstate.lock.info` for a moment: the **lock** that stops two commands changing the same state at once (topic 36). It disappears when the command finishes. After a second apply, `terraform.tfstate.backup` appears: the previous state version.

---

## Code / commands

### Step 1: write the configuration

```bash
mkdir -p ~/tf-lab/07-first && cd ~/tf-lab/07-first
```

`main.tf`:

```hcl
resource "local_file" "hello" {
  filename = "${path.module}/hello.txt"
  content  = "Hello, Terraform!\n"
}
```

- `resource`: the block type: "manage a thing".
- `"local_file"`: the **resource type**: which kind of thing (provider `local`, type `file`).
- `"hello"`: the **local name**: your name for it inside this code. Together, `local_file.hello` is its **address**.
- `filename`, `content`: **arguments**: settings for the file.
- `${path.module}`: an expression meaning "the folder this `.tf` file is in" (topic 17).

### Step 2: `terraform init`

```
$ terraform init
Initializing the backend...
Initializing provider plugins...
- Finding latest version of hashicorp/local...
- Installing hashicorp/local v2.5.3...
- Installed hashicorp/local v2.5.3 (signed by HashiCorp)
Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!
...
```

| Line | Meaning |
|---|---|
| `Initializing the backend...` | Where state will be stored. No `backend` block = **local**: `terraform.tfstate` in this folder (topic 35). |
| `Finding latest version of hashicorp/local...` | No version constraint, so it picks the newest. (In real code you set one, topic 10.) |
| `Installed ... (signed by HashiCorp)` | The download's signature was checked. |
| `created a lock file` | The exact version + checksums are recorded so everyone gets the same provider. |

### Step 3: `terraform plan`

```
$ terraform plan

Terraform used the selected providers to generate the following execution
plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # local_file.hello will be created
  + resource "local_file" "hello" {
      + content              = <<-EOT
            Hello, Terraform!
        EOT
      + content_base64sha256 = (known after apply)
      + content_base64sha512 = (known after apply)
      + content_md5          = (known after apply)
      + content_sha1         = (known after apply)
      + content_sha256       = (known after apply)
      + content_sha512       = (known after apply)
      + directory_permission = "0777"
      + file_permission      = "0777"
      + filename             = "./hello.txt"
      + id                   = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

─────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't
guarantee to take exactly these actions if you run "terraform apply" now.
```

- `+ create` / `+ resource`: will be created.
- `(known after apply)`: the provider computes these when it creates the file.
- `Plan: 1 to add...`: the summary.
- The `Note:` is about **saved plans** (`terraform plan -out=tfplan`), which matter in CI (topics 09, 56). Without one, `apply` makes a fresh plan, which might differ if something changed in between.

`ls` now: **no `hello.txt` yet**. Plan never changes anything.

### Step 4: `terraform apply`

```
$ terraform apply
...(the same plan is printed again)...
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

local_file.hello: Creating...
local_file.hello: Creation complete after 0s [id=1b8e3c6a7f0b5d2c9e4a8f1d3b6c0e7a2f5d9b4c]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

- `id=1b8e...`: for `local_file`, the ID is the SHA-1 hash of the content. Every resource type chooses its own ID (an AWS VM's is like `i-0abc...`).

```bash
cat hello.txt                # Hello, Terraform!
terraform state list         # local_file.hello
```

`terraform state list` prints every address Terraform is tracking (topic 12).

### Step 5: apply again with no change (idempotency)

```
$ terraform apply
local_file.hello: Refreshing state... [id=1b8e3c6a7f0b5d2c9e4a8f1d3b6c0e7a2f5d9b4c]

No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration
and found no differences, so no changes are needed.

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
```

- `Refreshing state...`: Terraform asked the provider to re-read the real file.
- `No changes.`: idempotent, as in topic 01.

### Step 6: change the content → a replacement

Edit `main.tf`: `content = "Hello again, Terraform!\n"`. Then:

```
$ terraform plan
local_file.hello: Refreshing state... [id=1b8e3c6a7f0b5d2c9e4a8f1d3b6c0e7a2f5d9b4c]

Terraform used the selected providers to generate the following execution
plan. Resource actions are indicated with the following symbols:
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # local_file.hello must be replaced
-/+ resource "local_file" "hello" {
      ~ content              = <<-EOT # forces replacement
          - Hello, Terraform!
          + Hello again, Terraform!
        EOT
      ~ content_base64sha256 = "h3Zl1o..." -> (known after apply)
      ~ content_md5          = "6f0a1..." -> (known after apply)
        (...the other four checksums change the same way...)
      ~ id                   = "1b8e3c6a7f0b5d2c9e4a8f1d3b6c0e7a2f5d9b4c" -> (known after apply)
        # (3 unchanged attributes hidden)
    }

Plan: 1 to add, 0 to change, 1 to destroy.
```

- `-/+ destroy and then create replacement`: the old object is **deleted**, then a new one is created.
- `# forces replacement`: **this** argument can't be changed in place. The provider says so. For `local_file`, *every* argument forces replacement. For an AWS VM, changing the size is in place (`~`), but changing the disk image forces replacement.
- `1 to add, 1 to destroy`: always look for `destroy` in the summary. On a database, this line is the difference between a normal day and an outage. Topic 24 shows how to control it.

Apply it, and `cat hello.txt`.

### Step 7: drift: delete the file behind Terraform's back

```bash
rm hello.txt
terraform plan
```

```
local_file.hello: Refreshing state... [id=7a1d...]

Note: Objects have changed outside of Terraform

Terraform detected the following changes made outside of Terraform since the
last "terraform apply" which may have affected this plan:

  # local_file.hello has been deleted
  - resource "local_file" "hello" {
      - filename = "./hello.txt" -> null
        # (...)
    }

Unless you have made equivalent changes to your configuration, or ignored the
relevant attributes using ignore_changes, the following plan may include
actions to undo or respond to these changes.

─────────────────────────────────────────────────────────────────────────────

  # local_file.hello will be created
  + resource "local_file" "hello" {
  ...
Plan: 1 to add, 0 to change, 0 to destroy.
```

- `Objects have changed outside of Terraform`: Terraform found **drift** during refresh and is telling you **before** proposing anything.
- The plan **re-creates** the file to match the code. Run `terraform apply` to fix it.

### Step 8: `terraform destroy`

```
$ terraform destroy
local_file.hello: Refreshing state... [id=7a1d...]

Terraform will perform the following actions:

  # local_file.hello will be destroyed
  - resource "local_file" "hello" {
      - content  = <<-EOT
            Hello again, Terraform!
        EOT -> null
      ...
    }

Plan: 0 to add, 0 to change, 1 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

local_file.hello: Destroying... [id=7a1d...]
local_file.hello: Destruction complete after 0s

Destroy complete! Resources: 1 destroyed.
```

- `-> null`: the value goes away.
- `There is no undo.`: true for real infrastructure. Treat `destroy` with respect.

`terraform state list` now prints nothing. `terraform.tfstate` still exists but is empty of resources.

---

## Hands-on lab

1. Do steps 1–8 above in `~/tf-lab/07-first`. Read every line of output.
2. Look inside: `cat .terraform.lock.hcl`, `ls -R .terraform | head`, and (after an apply) `cat terraform.tfstate | jq '.resources[0].instances[0].attributes.content'`. Don't edit; just look. Topic 12 explains the file.
3. Add a **second** resource in the same file and apply:

   ```hcl
   resource "local_file" "notes" {
     filename = "${path.module}/notes.txt"
     content  = "Managed by Terraform. Do not edit.\n"
   }
   ```

   The plan should say `1 to add` (the existing `hello` is unchanged).
4. Remove the `notes` block from `main.tf` and run `terraform plan`. It says `1 to destroy`: removing code means removing the resource, because Terraform remembers it in state. (That's the "Ansible can't do this" point from topic 02.) Apply.
5. Edit `hello.txt` by hand (`echo extra >> hello.txt`) and run `terraform plan`. What does it propose? (It reports `local_file.hello has been deleted` and plans `+ create`. The `local` provider treats a file whose content no longer matches its ID (the content hash) as gone, so Terraform writes it again. Other providers handle drift differently: an AWS provider usually shows `~ update in-place` back to your value. Topic 37 covers this.)
6. Clean up: `terraform destroy`, then `cd .. && rm -rf 07-first`.

---

## Where you'll use it

- The exact loop (init → plan → apply) is what every Terraform job in every company runs, by hand at first and in CI later.
- Reading `-/+` and `forces replacement` before typing `yes` is **the** most important production habit.
- The `local` provider itself is used in real code to write generated files: an Ansible inventory, a kubeconfig, a rendered config file.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `Error: Inconsistent dependency lock file` or `Missing required provider` | You ran `plan` before `init`. | `terraform init`. |
| `Error: No configuration files` | Wrong folder. | `cd` into the project folder. |
| `Error: Failed to query available provider packages` / `could not connect to registry.terraform.io` | No internet, or a proxy/DNS (Domain Name System) problem in WSL2. | Check `curl -I https://registry.terraform.io`. Fix WSL DNS (`/etc/resolv.conf`). |
| Plan shows `-/+` when you only "changed text" | That argument forces replacement. | Normal for `local_file`. For real infrastructure, read why before applying (topic 24). |
| `Error: Error acquiring the state lock` | Another `terraform` command is running in this folder, or one crashed. | Wait for it, or see topic 36. |
| Typed `y` instead of `yes` and nothing happened | Only `yes` approves. | `Apply cancelled.` is printed. Run it again. |

---

## Quick recap

- A project = one folder of `.tf` files. Run commands **inside** it.
- `init` → providers + lock file; `plan` → preview, changes nothing; `apply` → confirm, change, write state; `destroy` → delete everything in state.
- Symbols: `+` create, `~` update in place, `-` destroy, `-/+` replace. Read the summary line every time.
- Removing a block from code = Terraform destroys that resource. Changing things by hand = drift, shown on the next plan.

### Interview questions

1. **Q: What does `terraform init` do?**
   A: Prepares the working directory: configures the backend, downloads required providers (and modules) into `.terraform/`, and creates or checks `.terraform.lock.hcl`.
2. **Q: What does `-/+` mean in a plan?**
   A: The resource must be replaced: destroyed and recreated (or created then destroyed with `create_before_destroy`), because a changed argument can't be updated in place. The plan marks the cause with `# forces replacement`.
3. **Q: You delete a resource block from your code. What happens on the next apply?**
   A: Terraform still has it in state, so it plans to destroy the real object. To stop managing it without deleting it, use a `removed` block (topic 40).
