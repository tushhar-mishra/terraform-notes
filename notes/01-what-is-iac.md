# 01 · What Is Infrastructure as Code?

> **Level:** ★ foundation
>
> **You will learn:** what "infrastructure" means, how teams built it before IaC (Infrastructure as Code) and why that hurt, what IaC is, the difference between **declarative** and **imperative**, what **idempotent** means, what **drift** is, and **mutable vs immutable** infrastructure. Every later topic builds on these words.
>
> **You need:** nothing. This is the start.
>
> **Part 0 · Foundations** · Next: [02 · The IaC tool landscape](02-iac-tool-landscape.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** **Infrastructure** is everything your app runs *on*: servers, networks, firewalls, load balancers, databases, DNS records, storage buckets. Before IaC, people created these by **clicking in a web console** or typing commands by hand. Nobody could see exactly what was built, nobody could rebuild it the same way twice, and environments slowly drifted apart.

**Infrastructure as Code** means you **describe the infrastructure you want in text files**, keep those files in **Git**, and let a **tool** create or change the real infrastructure to match the files.

**Picture it.** A recipe instead of "cook from memory". Anyone can follow the recipe, it gives the same dish every time, and if the dish changes you can see *which line of the recipe* changed.

**The one thing to know.** Good IaC is **declarative** ("I want 3 servers") and **idempotent** (running it twice gives the same result as running it once). The tool works out *how* to get from what exists now to what you described.

**Try it now** (in your WSL2 Ubuntu terminal, no Terraform needed yet):

```bash
mkdir -p ~/tf-lab/01 && cd ~/tf-lab/01
mkdir demo            # imperative: run it twice → second time fails
mkdir demo            # mkdir: cannot create directory 'demo': File exists
mkdir -p demo         # "make sure it exists" → safe to run 100 times
```

`mkdir -p` behaves like an IaC tool: it describes the end state ("this folder exists") and is happy when it's already true.

**You've got it if you can:** explain to a friend why "a Git repo of `.tf` files" is safer than "a wiki page of console screenshots".

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### What "infrastructure" means

Your app is the code you wrote. **Infrastructure** is everything underneath it that has to exist before the app can run:

| Layer | Examples |
|---|---|
| Compute | Virtual machines (VMs), containers, Kubernetes clusters, serverless functions |
| Network | Networks and subnets, routes, firewalls, load balancers, DNS (Domain Name System) records |
| Storage | Disks, object storage buckets (e.g. AWS S3, Simple Storage Service), file shares |
| Data | Managed databases, caches, message queues |
| Identity | Users, roles, permissions, API (Application Programming Interface) keys |
| Platform "glue" | Monitoring alerts, log groups, certificates, secrets |

In the cloud, **all of this is created by calling an API**. Clicking "Create instance" in the AWS console is just a web page calling an AWS API for you. Keep that in mind: it's the reason IaC tools can exist at all.

### What Infrastructure as Code is

**IaC = writing down your infrastructure in text files, and using a tool to make reality match those files.**

Three parts:

1. **Code**: text files that describe what you want, e.g. "one network, two subnets, one VM of size small".
2. **Version control**: the files live in Git, so every change has an author, a date, a reason and a review.
3. **A tool**: reads the files, compares them with what really exists, and calls the cloud APIs to fix the difference. Terraform is one such tool.

### Four words you'll use forever

**Declarative vs imperative.**

- **Imperative** = you write the *steps*. "Create a server. Then create a disk. Then attach the disk." A Bash script is imperative.
- **Declarative** = you write the *end result*. "There is a server with a disk attached." The tool decides the steps.

You already know a declarative system: **Kubernetes**. A Deployment YAML says "I want 3 replicas", and controllers keep making reality match ([Kubernetes topic 15](../../CKA-Certification-Course-2025/notes/15-imperative-vs-declarative.md)). Terraform uses the same idea for cloud infrastructure, with one big difference you'll meet in topic 03: Terraform only checks reality **when you run it**, it doesn't watch all the time.

**Idempotent** = doing it again changes nothing if it's already done. `mkdir -p dir` is idempotent. `mkdir dir` isn't (the second run errors). `echo line >> file` isn't (each run adds another line). Declarative tools are idempotent by design: if reality already matches the file, they do nothing.

**Drift** = reality no longer matches the code. Someone changed a firewall rule in the console at 2 a.m. during an incident and forgot to put it in the code. The code says one thing; the cloud says another. Topic 37 covers drift fully.

**Mutable vs immutable infrastructure.**

- **Mutable** = you change servers in place: SSH (Secure Shell) in, `dnf update`, edit config files. Over months each server becomes slightly different.
- **Immutable** = you never change a running server. To change it, you build a new one from a new image and throw away the old one. Containers work this way ([Docker topic 01](../../Docker/notes/01-why-containers.md)): you don't patch a running container, you build a new image.

IaC tools make immutable infrastructure practical, because "throw it away and create a new one" is just one more run of the tool.

---

## Why it exists

### Life before IaC: "ClickOps"

**ClickOps** is the joke name for building infrastructure by clicking in web consoles. It works for one server. It breaks down quickly:

| Problem | What it looks like in real life |
|---|---|
| **No record** | "Who opened port 22 to the whole internet on the prod database?" Nobody knows. |
| **Not repeatable** | Building staging "the same as prod" takes 3 days, and it still isn't the same. |
| **Snowflake servers** | Every server is unique, like a snowflake: hand-patched, hand-tuned. Nobody dares to touch it or rebuild it. |
| **Slow** | A new environment needs a ticket, a queue, and a person clicking for hours. |
| **No review** | A change goes straight to production. Nobody checks it before it happens. |
| **Disaster recovery is a guess** | The region goes down. Can you rebuild everything in another region from memory? |
| **Drift** | Dev, staging and prod slowly become different. "Works in staging" stops meaning anything. |

Teams first tried **runbooks** (wiki pages of steps) and then **scripts** (Bash calling the cloud CLI (Command-Line Interface)). Scripts were better, but imperative scripts are hard:

```bash
aws ec2 run-instances --image-id ami-0abc --instance-type t3.micro
```

Run it twice: **two** servers. To be safe, the script must first check "does it already exist?", then "does it have the right size?", then "what if it exists with the wrong size, do I resize or recreate?"... for every kind of resource. That checking logic is exactly what an IaC tool writes for you, once, for everyone.

### What IaC gives you

| Benefit | How |
|---|---|
| **History and blame** | `git log` shows who changed what, when and why. |
| **Review before change** | Changes go through a Pull Request (PR). The tool shows a **preview** of what will change ("plan") before anything happens. |
| **Repeatable environments** | Same code + different inputs = dev, staging, prod that really match. |
| **Speed** | A full environment in minutes, by anyone allowed to run the pipeline. |
| **Disaster recovery** | Point the same code at another region. |
| **Self-documentation** | The code *is* the current documentation. It can't go out of date like a wiki page. |
| **Automation** | It plugs into CI/CD (Continuous Integration / Continuous Delivery) pipelines like the ones in your Jenkins and GitHub Actions notes. |

---

## Analogy

**Building flat-pack furniture.**

- **ClickOps** = a carpenter builds a cupboard by eye. It's lovely, but if you want a second identical one, he has to remember everything. If he's on holiday, nobody can fix it.
- **Imperative script** = a list of steps: "drill hole A, screw B into A...". If you run the steps on a half-built cupboard, you drill holes twice.
- **Declarative IaC** = the **picture of the finished cupboard** plus a clever assistant. You show the picture; the assistant looks at what's already built and does only the missing steps. Show the picture again tomorrow and the assistant says "already done" (idempotent). If someone removes a shelf, the assistant notices the difference the next time you ask (drift).

---

## How it works inside

Every declarative IaC tool runs the same loop. Terraform's version of it is traced in detail in topic 13; here is the general shape:

```
   ┌──────────────┐     ┌────────────────────┐     ┌──────────────────┐
   │ Desired state│     │ What exists now    │     │ Difference       │
   │ (your files) │ ──▶ │ (ask the cloud API)│ ──▶ │ create / change /│
   └──────────────┘     └────────────────────┘     │ delete           │
                                                   └────────┬─────────┘
                                                            │ show it to a human (preview)
                                                            ▼
                                                   ┌──────────────────┐
                                                   │ Call the APIs to │
                                                   │ apply the change │
                                                   └──────────────────┘
```

1. **Read the desired state** from your files.
2. **Read the actual state** by asking the real system.
3. **Compute the difference**: what to create, change or delete.
4. **Show a preview** so a human can check it.
5. **Make the changes** by calling APIs, in the right order.
6. Running it again: step 3 finds no difference, so nothing happens (idempotent).

---

## Code / commands

You haven't installed Terraform yet (topic 06), so here is the same goal written three ways. Read it; don't run the AWS parts.

**Goal:** one small Linux VM in AWS called `web`.

**1. ClickOps:** EC2 (Elastic Compute Cloud, AWS's virtual machine service) console → Launch instance → name `web` → pick image → pick `t3.micro` → Launch. No record except the AWS audit log.

**2. Imperative script** (Bash + AWS CLI):

```bash
#!/usr/bin/env bash
set -euo pipefail
# Must check first, or every run creates another VM
existing=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=web" "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].InstanceId' --output text)

if [ -z "$existing" ]; then
  aws ec2 run-instances --image-id ami-0abc123 --instance-type t3.micro \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web}]'
else
  echo "web already exists: $existing"
  # ...and what if its size is wrong? More code. What if it's stopped? More code.
fi
```

**3. Declarative IaC** (Terraform, explained properly in topics 07–11):

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abc123"
  instance_type = "t3.micro"
  tags = {
    Name = "web"
  }
}
```

- `resource "aws_instance" "web"`: "there is an AWS VM, and in my code I call it `web`".
- `ami`: which disk image to boot from (AMI = Amazon Machine Image).
- `instance_type`: the VM size.
- `tags`: labels stored on the VM.

There are no "if it exists" checks. Change `t3.micro` to `t3.small`, run the tool again, and it works out by itself that it must **change the size of the existing VM**, not create a second one. A preview would look like this (sample shape; you'll learn to read it in topic 09):

```
  # aws_instance.web will be updated in-place
  ~ resource "aws_instance" "web" {
        id            = "i-0a1b2c3d4e5f67890"
      ~ instance_type = "t3.micro" -> "t3.small"
        tags          = { "Name" = "web" }
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

- `~`: this thing will be **changed in place**.
- `"t3.micro" -> "t3.small"`: old value → new value.
- `Plan: 0 to add, 1 to change, 0 to destroy.`: the summary. Always read it before saying yes.

---

## Hands-on lab

No cloud needed. You'll *feel* why idempotency matters, using Bash from your [Linux notes](../../linux-mastery-rhel/notes/040-script-basics.md).

1. Create a "non-idempotent" setup script and run it three times:

   ```bash
   mkdir -p ~/tf-lab/01 && cd ~/tf-lab/01
   cat > setup-bad.sh <<'EOF'
   mkdir app
   echo "port=8080" >> app/config.ini
   EOF
   bash setup-bad.sh; bash setup-bad.sh; bash setup-bad.sh
   cat app/config.ini
   ```

   You'll see `mkdir: cannot create directory 'app': File exists` twice, and **three** `port=8080` lines. That's what imperative re-runs do.

2. Write an idempotent version:

   ```bash
   rm -rf app
   cat > setup-good.sh <<'EOF'
   mkdir -p app
   printf 'port=8080\n' > app/config.ini     # overwrite = "the file contains exactly this"
   EOF
   bash setup-good.sh; bash setup-good.sh; bash setup-good.sh
   cat app/config.ini                          # exactly one line
   ```

3. Simulate **drift**: `echo "debug=true" >> app/config.ini`. Run `bash setup-good.sh` again, then `cat app/config.ini`. The drift is gone, because the script re-asserted the desired state. Terraform does the same for cloud resources, but it first **shows you** the difference instead of silently overwriting.

4. Think: what would `setup-good.sh` need to do if the desired state said "**no** `app` folder"? (Answer: delete it. A declarative tool must also know what to *remove*. Terraform remembers what it created in its **state** file, topic 12, so it knows what to delete.)

---

## Where you'll use it

- **Every cloud job.** Most companies on AWS, Azure or GCP (Google Cloud Platform) manage infrastructure with IaC. "Do you know Terraform?" is a standard DevOps interview question.
- **New environments**: spin up a full copy of prod for testing, then delete it.
- **Disaster recovery**: rebuild in another region from the same code.
- **Compliance and audits**: "show me every firewall change in the last year" = `git log`.
- **Platform teams**: they publish reusable IaC building blocks (modules, Part 6) so product teams can create a database in a standard, safe way.

---

## When it breaks

IaC doesn't remove problems; it changes which problems you get. The ones you'll meet in this course:

| Problem | What it means | Where it's fixed |
|---|---|---|
| Someone clicks in the console anyway | **Drift**: code and reality disagree | Topic 37 |
| Two people run the tool at once | They overwrite each other's changes | Locking, topic 36 |
| The tool's memory of what it built is lost | It may try to create everything again | State recovery, topic 42 |
| A small code change deletes and recreates a database | **Unexpected replacement** | Reading plans (09), `lifecycle` (24), `-replace` (41) |
| Secrets written in the code | Passwords leaked into Git history | Topic 51 |

---

## Quick recap

- **Infrastructure** = what your app runs on. In the cloud, all of it is created by API calls.
- **IaC** = infrastructure described in text files, stored in Git, applied by a tool.
- **Declarative** (describe the end state) + **idempotent** (re-running changes nothing) is what makes IaC safe.
- **Drift** = reality differs from the code. **Immutable** = replace instead of patching in place.
- IaC gives history, review, repeatability, speed and disaster recovery.

### Interview questions

1. **Q: What is Infrastructure as Code and why use it?**
   A: Managing infrastructure through version-controlled definition files that a tool applies, instead of manual clicks. It gives repeatable environments, review through PRs, history, faster provisioning and easier disaster recovery.
2. **Q: Declarative vs imperative: which is Terraform, and why does it matter?**
   A: Terraform is declarative: you describe the end state and it computes the steps. That makes it idempotent: re-running with no code changes makes no changes, and it can detect drift.
3. **Q: What is configuration drift?**
   A: When real infrastructure no longer matches its code, usually because of manual changes. IaC tools detect it by comparing the code (and their stored state) with reality.
