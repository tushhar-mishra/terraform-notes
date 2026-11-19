# 52 · Provisioners, and What to Use Instead

> **Level:** ★★ intermediate
>
> **You will learn:** what ⚠️ **provisioners** are (`local-exec`, `remote-exec`, `file`, with `connection` blocks, `when = destroy`, `on_failure`), why HashiCorp calls them a **last resort**, the built-in **`terraform_data`** resource that replaced ⚠️ `null_resource`, and the better tools for each job: **`user_data`/cloud-init**, **Packer** images, **Ansible** after Terraform, **SSM Run Command**, provider-native resources, and Terraform 1.14's **`action`** blocks.
>
> **You need:** topics 01 (immutable infrastructure), 02 (Ansible, Packer), 24 (`terraform_data`, `replace_triggered_by`), 33 (`user_data`, SSM), 41 (tainted).
>
> **Part 8 · Security and quality** · Previous: [51 · Secrets and sensitive data](51-secrets.md) · Next: [53 · Linting and security scanning](53-linting-and-scanning.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **provisioner** runs a **command** as part of creating (or destroying) a resource: on your machine (`local-exec`) or on the new server over SSH (`remote-exec`, `file`).

```hcl
resource "aws_instance" "web" {
  # ...
  provisioner "local-exec" {                  # ⚠️ last resort
    command = "echo ${self.public_ip} >> inventory.txt"
  }
}
```

Terraform can't plan, diff or undo what a command does. So provisioners are **imperative holes** in a declarative tool.

| Job | Instead of a provisioner, use |
|---|---|
| Configure a VM at boot | **`user_data` / cloud-init** (topic 33) |
| Install software repeatably | **Packer**-built image (immutable) |
| Configure many servers over time | **Ansible** (reads Terraform outputs) / **SSM** |
| Write a file locally | `local_file` resource |
| Call an API Terraform has no resource for | Find/write a **provider**; or `terraform_data` + `local-exec` as a last resort |
| A one-off operation (invalidate a CDN cache, run a Lambda) | Terraform 1.14+ **`action`** blocks, if the provider offers one |

**Picture it.** Hiring a builder (Terraform) who works from blueprints, then slipping him a sticky note "also, on the way out, repaint the neighbour's fence". He'll do it, but there's no blueprint for it, no record, and no way to undo it.

**The one thing to know.** If you write a provisioner, you must also answer: what happens if it fails? If it runs twice? How do I change it later?

**Try it now:** run the `terraform_data` + `local-exec` example below, then make the command fail and see the resource become **tainted**.

**You've got it if you can:** list three problems with provisioners and the better tool for each common use.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### The three provisioners

| Provisioner | Runs | Typical (legacy) use |
|---|---|---|
| `local-exec` | A command **on the machine running Terraform** | Call a script/CLI, write an inventory |
| `remote-exec` | Commands **on the remote resource**, over SSH or WinRM | Install packages on a new VM |
| `file` | Copies files **to** the remote resource | Upload config before `remote-exec` |

```hcl
resource "aws_instance" "web" {
  # ...
  connection {                                # how remote-exec/file reach the server
    type        = "ssh"
    host        = self.public_ip              # "self" = this resource (provisioners can't reference it by name)
    user        = "ec2-user"
    private_key = file("~/.ssh/id_ed25519")
  }

  provisioner "file" {
    source      = "app.conf"
    destination = "/tmp/app.conf"
  }

  provisioner "remote-exec" {
    inline = [
      "sudo dnf -y install nginx",
      "sudo mv /tmp/app.conf /etc/nginx/conf.d/",
      "sudo systemctl enable --now nginx",
    ]
  }
}
```

Settings:

| Setting | Meaning |
|---|---|
| (default) | Runs **once**, after the resource is **created**. Not on updates. |
| `when = destroy` | Runs **before** the resource is destroyed. Can only use `self`, `count.index`, `each.key`. |
| `on_failure = fail` (default) | A failure **taints** the resource (topic 41) and stops the apply. |
| `on_failure = continue` | Ignore failures. ⚠️ Hides problems. |
| `local-exec`: `interpreter`, `environment`, `working_dir`, `quiet` | Control how the command runs. |

### Why they're a last resort

| Problem | Explanation |
|---|---|
| **Not in the plan** | The plan says "create instance"; it can't show what the script will do. |
| **No drift detection, no updates** | Change the script → nothing happens (it ran once at create). The state has no record of its effects. |
| **Not idempotent** | A retry after a partial failure may run commands twice. |
| **Failures taint resources** | A flaky `dnf install` = the whole VM is replaced on the next apply. |
| **Network and credentials** | `remote-exec` needs SSH from the Terraform runner to every VM (open port 22, keys in CI). |
| **Destroy-time provisioners are fragile** | Don't run if the resource is removed with its block in some cases, or if `create_before_destroy` interacts badly. |
| **Environment-dependent** | `local-exec` depends on tools installed on whatever machine runs Terraform (laptop vs CI image). |

HashiCorp's docs: "Provisioners should only be used as a last resort."

### `terraform_data` (replaces ⚠️ `null_resource`)

When you truly need a command that isn't tied to a real resource, attach it to `terraform_data` (built-in, topic 24):

```hcl
resource "terraform_data" "warm_cache" {
  triggers_replace = [aws_instance.web.id]          # re-run when the instance is replaced

  provisioner "local-exec" {
    command = "curl -fsS --retry 10 --retry-delay 6 http://${aws_instance.web.public_ip}/warmup"
  }
}
```

- `null_resource` (from the `hashicorp/null` provider, with `triggers`) did the same; it's ⚠️ superseded by `terraform_data`. Migrate with a `moved` block (topic 40).

### `action` blocks (Terraform 1.14+)

Some operations are **actions**, not things: "invalidate the CloudFront cache", "invoke this Lambda", "run this SSM command". Terraform 1.14 added **`action`** blocks, which are **provider-defined** operations you can trigger from a resource's lifecycle events (`after_create`, `after_update`; 1.16 added `before_destroy`/`after_destroy`) or run explicitly (`terraform apply -invoke=action.TYPE.NAME`). Unlike `local-exec`, they're typed, validated, shown in plans and don't depend on tools installed on the runner. Provider coverage is growing; check your provider's docs for an "Actions" section (topic 67).

```hcl
action "aws_lambda_invoke" "notify" {
  config {
    function_name = aws_lambda_function.notifier.function_name
    payload       = jsonencode({ event = "deployed" })
  }
}

resource "terraform_data" "release" {
  input = var.app_version
  lifecycle {
    action_trigger {
      events  = [after_update]
      actions = [action.aws_lambda_invoke.notify]
    }
  }
}
```

(Shape shown for orientation; exact action types and arguments come from the provider.)

---

## Why it exists

Before Terraform had rich providers, Packer, cloud-init and SSM, provisioners were the only way to finish configuring a VM in one tool. They remain for edge cases where nothing else exists. Knowing **why** they're discouraged, and the replacements, is a standard interview topic.

---

## Analogy

**A builder with blueprints and sticky notes.**

- **Blueprints** (resources) are drawn, reviewed, compared to the building, and changed safely.
- **Sticky notes** (provisioners) say "on the way out, repaint the fence". There's no blueprint, no record, no way to see if it's still painted, and if the builder slips while painting, the **whole house** is marked "faulty" (tainted).
- **Better options**: deliver the house with the furniture already inside (**Packer image**); a move-in checklist the house runs itself (**cloud-init**); a caretaker who visits regularly (**Ansible/SSM**); a proper service contract for special jobs (**provider resources/actions**).

---

## How it works inside

```
apply: aws_instance.web Create
   provider creates the instance → returns attributes
   core runs provisioners in order (creation-time):
     connection: SSH to self.public_ip (retries until timeout, default 5m)
     file → remote-exec → ...
   any failure (and on_failure = fail) → instance saved as TAINTED → apply stops
   success → instance saved normally
later plans: provisioners are NOT re-run (unless the resource is replaced)
destroy: when = destroy provisioners run BEFORE the provider's Delete
```

---

## Code / commands

### `terraform_data` + `local-exec` (free, local)

`~/tf-lab/52-provisioners/main.tf`:

```hcl
variable "app_version" {
  type    = string
  default = "1.0.0"
}

resource "terraform_data" "deploy_marker" {
  input            = var.app_version
  triggers_replace = [var.app_version]           # new version → replace → command runs again

  provisioner "local-exec" {
    command = "echo \"$(date -Is) deployed ${self.input}\" >> deploy.log"
  }

  provisioner "local-exec" {
    when    = destroy
    command = "echo \"$(date -Is) removed ${self.input}\" >> deploy.log"
  }
}
```

```bash
terraform init && terraform apply -auto-approve
```

```
terraform_data.deploy_marker: Creating...
terraform_data.deploy_marker: Provisioning with 'local-exec'...
terraform_data.deploy_marker (local-exec): Executing: ["/bin/sh" "-c" "echo \"$(date -Is) deployed 1.0.0\" >> deploy.log"]
terraform_data.deploy_marker: Creation complete after 0s [id=6e1b...]
```

```bash
terraform apply -auto-approve -var="app_version=1.1.0"
cat deploy.log
```

```
2026-11-19T17:01:02+05:30 deployed 1.0.0
2026-11-19T17:03:44+05:30 removed 1.0.0
2026-11-19T17:03:44+05:30 deployed 1.1.0
```

The destroy-time provisioner ran for the old object, the create-time one for the new.

### A failing provisioner → tainted

Change the command to `exit 1` and apply with a new version:

```
terraform_data.deploy_marker (local-exec): Executing: ["/bin/sh" "-c" "exit 1"]
╷
│ Error: local-exec provisioner error
│
│   with terraform_data.deploy_marker,
│   on main.tf line 9, in resource "terraform_data" "deploy_marker":
│    9:   provisioner "local-exec" {
│
│ Error running command 'exit 1': exit status 1. Output:
╵
```

```bash
terraform plan
```

```
  # terraform_data.deploy_marker is tainted, so must be replaced
```

Now imagine that was a production VM with a flaky package mirror.

### The better way for VM setup (recap)

| Approach | Where | Topic |
|---|---|---|
| `user_data` + cloud-init at first boot | `aws_instance.user_data` / launch templates | 33 |
| Pre-built image | Packer template → AMI ID → `aws_instance.ami` (via a data source with `most_recent` and your image name) | below |
| Post-boot configuration | Ansible inventory from `terraform output -json`, run in the pipeline after apply | below |
| Fleet commands | SSM Run Command / State Manager associations (`aws_ssm_association`) | 33 |

A minimal **Packer** template (HCL2), to see the idea:

```hcl
# web.pkr.hcl
source "amazon-ebs" "web" {
  region        = "ap-south-1"
  instance_type = "t3.micro"
  source_ami_filter {
    filters     = { name = "al2023-ami-2023.*-x86_64" }
    owners      = ["amazon"]
    most_recent = true
  }
  ssh_username = "ec2-user"
  ami_name     = "acme-web-{{timestamp}}"
}

build {
  sources = ["source.amazon-ebs.web"]
  provisioner "shell" {
    inline = ["sudo dnf -y install nginx", "sudo systemctl enable nginx"]
  }
}
```

`packer build web.pkr.hcl` → a new AMI. Terraform then just launches it (immutable infrastructure, topic 01). Packer's provisioners run **at image build time**, where failures are cheap and nothing in production depends on them.

**Ansible after Terraform**, in a pipeline step:

```bash
terraform output -json web_ips | jq -r '.[]' > inventory.ini
ansible-playbook -i inventory.ini site.yml
```

---

## Hands-on lab

1. Run the `terraform_data` example; bump the version twice; read `deploy.log`.
2. Make the provisioner fail; see the taint; fix it.
3. Rewrite topic 33's web server **as if** with `remote-exec` (on paper): list what you'd need (SSH key in CI, port 22 open, retries...). Compare with the `user_data` version you built.
4. Migrate a `null_resource` to `terraform_data`: create one with the `hashicorp/null` provider, apply, then switch to `terraform_data` with a `moved` block (topic 40) and plan.
5. Read your AWS provider docs for an **Actions** section and note one available action.
6. `terraform destroy`.

---

## Where you'll use it

- **Reading legacy code**: lots of older Terraform uses `remote-exec`, `null_resource` and `local-exec`; migrating them is common work.
- **Rare glue**: calling a script that has no provider equivalent, as a documented exception.
- **Image pipelines**: Packer + Terraform is the standard immutable approach.
- **Interviews**: "Why are provisioners discouraged? What would you use instead?"

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `timeout - last error: dial tcp x.x.x.x:22: i/o timeout` | `remote-exec` can't SSH: security group, no public IP, wrong user/key. | Don't use SSH provisioning; use `user_data`/SSM. |
| Resource tainted after a provisioner failure | Default `on_failure = fail`. | Fix the script; let Terraform replace; better: move the logic out of provisioners. |
| Script changed but nothing ran | Creation-time provisioners run once. | `triggers_replace` on `terraform_data`, or don't use provisioners. |
| Works on a laptop, fails in CI | `local-exec` relies on tools/paths on the runner. | Install them in the CI image, or avoid `local-exec`. |
| Destroy-time provisioner didn't run | It was removed together with the resource block, or the resource was tainted. | Keep logic out of destroy provisioners; use `removed` blocks carefully. |
| `Deprecated provider "null"` thoughts | `null_resource` superseded. | `terraform_data` + `moved`. |

---

## Quick recap

- Provisioners (`local-exec`, `remote-exec`, `file`) run commands at create/destroy; ⚠️ last resort: not planned, not idempotent, taint on failure, need network access.
- Use `terraform_data` (not `null_resource`) when you must, with `triggers_replace`.
- Better tools: **cloud-init/`user_data`**, **Packer** images, **Ansible/SSM** after apply, provider resources, and **`action` blocks** (1.14+) for provider-defined operations.

### Interview questions

1. **Q: Why are provisioners discouraged?**
   A: Their effects aren't in the plan or state, they run only at create/destroy, aren't idempotent, taint resources on failure, and often need SSH access and tools on the runner. HashiCorp recommends them only as a last resort.
2. **Q: How would you install software on EC2 instances created by Terraform?**
   A: Bake it into an AMI with Packer (immutable), or use `user_data`/cloud-init at first boot; for ongoing configuration, Ansible or SSM after Terraform. Not `remote-exec`.
3. **Q: What replaced `null_resource`?**
   A: The built-in `terraform_data` resource (1.4+): it stores an `input`, exposes an `output`, and is replaced when `triggers_replace` changes, so it can host provisioners or act as a `replace_triggered_by` trigger.
