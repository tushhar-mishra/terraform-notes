# 35 · Remote Backends

> **Level:** ★★ intermediate
>
> **You will learn:** what a **backend** is, why **local state fails** for teams and CI, how to configure the **S3 backend** (bucket, key, region, encryption, locking, roles), **partial backend configuration** (`-backend-config`) for multiple environments, how to **migrate** state between backends (`init -migrate-state`, `-reconfigure`), the **bootstrap problem** (where does the state bucket's own state live?), a recommended **key layout**, the IAM permissions Terraform needs, the hidden `.terraform/terraform.tfstate` file, and the other backend types (azurerm, gcs, HCP Terraform's `cloud` block).
>
> **You need:** topics 12, 31 (S3 bucket settings), 34.
>
> **Part 5 · State in depth** · Previous: [34 · State deep dive](34-state-deep-dive.md) · Next: [36 · State locking](36-state-locking.md) · [Back to INDEX](../INDEX.md)
>
> 💰 **Cost:** one small versioned S3 bucket: a few cents a month at most.

---

## ⚡ Core: the 10-minute version

**The idea.** A **backend** decides **where state lives**. The default is the **local** file. For any shared work, use a **remote backend**: the state is in a central, encrypted, versioned, **locked** place that everyone (and CI) uses.

```hcl
terraform {
  backend "s3" {
    bucket       = "acme-tfstate-123456789012"      # created once, by a bootstrap config
    key          = "dev/network/terraform.tfstate"  # one key per environment + component
    region       = "ap-south-1"
    encrypt      = true                             # server-side encryption
    use_lockfile = true                             # native S3 locking (topic 36)
  }
}
```

```bash
terraform init -migrate-state    # move existing local state into the bucket
```

**Picture it.** Local state is a notebook in your bag: nobody else can read it, and if you lose the bag it's gone. A remote backend is a **shared ledger in the office safe**, with a sign-out sheet (lock) and photocopies of every old page (versioning).

**The one thing to know.** The backend block **can't use variables**. For different environments, use a different `key` via **partial configuration**: `terraform init -backend-config=dev.s3.tfbackend`.

**Try it now:** follow the lab: bootstrap a state bucket, then migrate a local-provider project's state into it.

**You've got it if you can:** explain why the state bucket is created by a separate configuration, and what `-migrate-state` vs `-reconfigure` do.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Why local state fails

| Problem | With `terraform.tfstate` on a laptop |
|---|---|
| **Sharing** | Teammates and CI don't have it → they'd plan to create everything again. |
| **Concurrency** | Two people apply from two copies → overwrite each other, corrupt infrastructure. |
| **Loss** | Laptop dies, folder deleted → state gone (topic 42). |
| **Security** | Secrets in plain text on laptops, in backups, maybe in Git. |
| **History** | Only one `.backup` copy. |

A good remote backend gives **one shared copy + locking + encryption + access control + version history**.

### Backend types

| Backend | Storage | Locking |
|---|---|---|
| `local` (default) | A file | OS file lock (same machine only) |
| **`s3`** | Amazon S3 object | **S3 lock file** (`use_lockfile`), ⚠️ DynamoDB (deprecated) |
| `azurerm` | Azure Blob Storage | Blob lease |
| `gcs` | Google Cloud Storage | Lock object |
| `kubernetes` | A Secret in a cluster | Lease |
| `pg` | PostgreSQL | Advisory locks |
| `http` | Any REST endpoint (e.g. GitLab managed state) | Optional |
| HCP Terraform | `cloud { }` block (not `backend`) | Built in; remote runs (topic 50) |

### The S3 backend arguments

| Argument | Meaning |
|---|---|
| `bucket` | The state bucket (must already exist). |
| `key` | Path of the state object in the bucket. **One key = one state.** |
| `region` | Bucket's region. |
| `encrypt` | Server-side encryption of the state object. Set `true`. |
| `kms_key_id` | Use your own KMS key instead of S3-managed keys (stricter access control). |
| `use_lockfile` | Lock with a `.tflock` object next to the state (Terraform 1.10+; GA in 1.11). |
| `dynamodb_table` | ⚠️ Deprecated older locking via a DynamoDB table (topic 36). |
| `profile`, `assume_role { role_arn = ... }` | Credentials **for the backend**, separate from the provider's. |
| `workspace_key_prefix` | Prefix for non-default workspaces' state (topic 48). Default `env:`. |

The backend has its **own** credentials. It doesn't reuse the `provider "aws"` block. It reads the same credential chain (topic 30): env vars, `AWS_PROFILE`, etc.

### Partial configuration

The `backend` block is processed during `init`, **before** variables exist, so it can't contain `var.` or `local.` references. Instead, leave out the values that change and pass them at `init`:

```hcl
# backend.tf
terraform {
  backend "s3" {}          # everything else comes from -backend-config
}
```

```hcl
# backends/dev.s3.tfbackend
bucket       = "acme-tfstate-123456789012"
key          = "dev/app/terraform.tfstate"
region       = "ap-south-1"
encrypt      = true
use_lockfile = true
```

```bash
terraform init -backend-config=backends/dev.s3.tfbackend
# or individual values:  -backend-config="key=prod/app/terraform.tfstate"
```

`.tfbackend` is the conventional extension (any name works).

### Key layout

One state per **environment × component** keeps each state small and limits the **blast radius** (what one mistake can damage):

```
s3://acme-tfstate-123456789012/
├── bootstrap/terraform.tfstate            (optional: the bucket's own config, see below)
├── dev/network/terraform.tfstate
├── dev/app/terraform.tfstate
├── prod/network/terraform.tfstate
└── prod/app/terraform.tfstate
```

Even better for production: **separate buckets per AWS account** (dev state in the dev account, prod state in the prod account), so dev credentials can't even read prod state. Topic 49 covers layouts.

### The bootstrap ("chicken and egg") problem

The backend bucket must exist **before** `terraform init` can use it, so the configuration that **uses** it can't be the one that **creates** it. Options:

1. **A small bootstrap configuration** with **local state** that creates the bucket (versioning, encryption, public access block, TLS-only policy, `prevent_destroy`). Keep its tiny state somewhere safe, or migrate it into the bucket it created (a common trick).
2. Create the bucket with the AWS CLI or a CloudFormation template once.
3. Let a platform (HCP Terraform, Terragrunt's `remote_state` auto-creation, topic 49) handle it.

### `.terraform/terraform.tfstate`

After `init` with a remote backend, you'll find `.terraform/terraform.tfstate`. It is **not** your infrastructure state. It records **which backend this folder is configured for** (type, bucket, key...). Terraform compares it with the `backend` block on every run and asks you to re-run `init` when they differ.

---

## Why it exists

Teams and pipelines need **one source of truth** for "what Terraform manages". Backends let state live where access can be controlled and audited, with history to recover from mistakes and locks to stop concurrent writes. Without it, Terraform is a single-user tool.

---

## Analogy

**The office ledger.**

- **Local state** = your personal notebook. Fine alone; useless to colleagues; lost with your bag.
- **Remote backend** = the official ledger in the office **safe** (encrypted bucket).
- **Key** = which ledger in the safe: "Dev network", "Prod app". Separate ledgers mean spilling coffee on one doesn't ruin the others (blast radius).
- **Versioning** = a photocopy of every page before it's changed.
- **Lock** = the sign-out sheet: while you have the ledger, nobody else can write in it (topic 36).
- **Bootstrap** = someone has to buy the safe **before** any ledger can be put in it. You can't record the safe's purchase in a ledger that lives inside the safe you haven't bought yet.

---

## How it works inside

```
terraform init
  1. read the backend block (+ -backend-config values)
  2. compare with .terraform/terraform.tfstate (the folder's last backend settings)
       same       → nothing to do
       different  → ask: migrate (-migrate-state) or forget the old one (-reconfigure)?
       local state exists + new remote backend + remote empty → offer to COPY it
  3. check access: s3:ListBucket on the bucket; read the key if it exists
  4. write the new backend settings to .terraform/terraform.tfstate

terraform plan / apply
  1. acquire the lock  (PUT <key>.tflock, only if absent → else "Error acquiring the state lock")
  2. GET <key>                          (download state into memory; never written to disk)
  3. ... plan / apply ...
  4. PUT <key>  (new serial)            (apply only; S3 keeps the old one as a previous version)
  5. DELETE <key>.tflock                (release lock)
```

Note: with a remote backend, **no state file is written** in your working directory. If you see a `terraform.tfstate` next to your `.tf` files in a remote-backend project, it's a **leftover** from before migration. Delete it after confirming the remote copy is correct, to avoid confusion.

IAM permissions the backend needs (for the identity running Terraform):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::acme-tfstate-123456789012" },
    { "Effect": "Allow", "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::acme-tfstate-123456789012/dev/app/terraform.tfstate" },
    { "Effect": "Allow", "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::acme-tfstate-123456789012/dev/app/terraform.tfstate.tflock" }
  ]
}
```

(Plus `kms:Encrypt`/`kms:Decrypt`/`kms:GenerateDataKey` on the key if you use `kms_key_id`.) Scoping by key prefix (`dev/*`) is how you give the dev pipeline access to dev state only.

---

## Code / commands

### Step 1: the bootstrap configuration

`~/tf-lab/35-bootstrap/main.tf`:

```hcl
terraform {
  required_version = ">= 1.14"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.0" }
  }
  # no backend block: local state for the bootstrap itself
}

provider "aws" {
  region = "ap-south-1"
  default_tags {
    tags = { project = "tf-course", purpose = "terraform-state", managed_by = "terraform" }
  }
}

data "aws_caller_identity" "current" {}

resource "aws_s3_bucket" "state" {
  bucket = "tf-course-state-${data.aws_caller_identity.current.account_id}"

  lifecycle {
    prevent_destroy = true            # losing this bucket = losing every state
  }
}

resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.state.id
  versioning_configuration {
    status = "Enabled"                # history of every state version: your undo button
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "state" {
  bucket                  = aws_s3_bucket.state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Refuse any non-TLS access
data "aws_iam_policy_document" "state" {
  statement {
    sid     = "DenyInsecureTransport"
    effect  = "Deny"
    actions = ["s3:*"]
    resources = [
      aws_s3_bucket.state.arn,
      "${aws_s3_bucket.state.arn}/*",
    ]
    principals {
      type        = "*"
      identifiers = ["*"]
    }
    condition {
      test     = "Bool"
      variable = "aws:SecureTransport"
      values   = ["false"]
    }
  }
}

resource "aws_s3_bucket_policy" "state" {
  bucket = aws_s3_bucket.state.id
  policy = data.aws_iam_policy_document.state.json
}

# Keep old state versions for 90 days, not forever
resource "aws_s3_bucket_lifecycle_configuration" "state" {
  bucket = aws_s3_bucket.state.id
  rule {
    id     = "expire-old-state-versions"
    status = "Enabled"
    filter {}
    noncurrent_version_expiration {
      noncurrent_days = 90
    }
  }
}

output "state_bucket" {
  value = aws_s3_bucket.state.bucket
}
```

- The **account ID** in the name makes it globally unique and self-describing.
- `prevent_destroy` + versioning + TLS-only + no public access: the standard state-bucket checklist.

```bash
cd ~/tf-lab/35-bootstrap
export AWS_PROFILE=tf-lab
terraform init && terraform apply
```

```
Outputs:

state_bucket = "tf-course-state-123456789012"
```

### Step 2: migrate an existing project from local to S3

Take your topic 34 Docker lab (local providers, local state). Make sure it's applied, then add `backend.tf`:

```hcl
terraform {
  backend "s3" {}
}
```

and `backends/lab.s3.tfbackend`:

```hcl
bucket       = "tf-course-state-123456789012"
key          = "lab/34-state/terraform.tfstate"
region       = "ap-south-1"
encrypt      = true
use_lockfile = true
```

```bash
cd ~/tf-lab/34-state
terraform state pull > before-migration.tfstate        # backup first, always
terraform init -backend-config=backends/lab.s3.tfbackend -migrate-state
```

```
Initializing the backend...
Do you want to copy existing state to the new backend?
  Pre-existing state was found while migrating the previous "local" backend to the
  newly configured "s3" backend. No existing state was found in the newly
  configured "s3" backend. Do you want to copy this state to the new "s3"
  backend? Enter "yes" to copy and "no" to start with an empty state.

  Enter a value: yes

Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.
...
Terraform has been successfully initialized!
```

Verify:

```bash
terraform plan                       # must say: No changes.
aws s3 ls s3://tf-course-state-123456789012/lab/34-state/
terraform state pull | jq .serial
```

```
No changes. Your infrastructure matches the configuration.

2026-11-19 14:02:11       8921 terraform.tfstate
```

`No changes` proves the remote state matches. Now remove the old local files: `rm terraform.tfstate terraform.tfstate.backup` (keep `before-migration.tfstate` somewhere safe until you're confident, then delete it: it has secrets).

Docker containers run on your laptop, the state lives in AWS: backends and providers are independent.

### Changing the backend later

Edit the key (e.g. `lab/34-state-v2/terraform.tfstate`) and run plain `terraform init`:

```
╷
│ Error: Backend configuration changed
│
│ A change in the backend configuration has been detected, which may require
│ migrating existing state.
│
│ If you wish to attempt automatic migration of the state, use "terraform init
│ -migrate-state".
│ If you wish to store the current configuration with no changes to the state,
│ use "terraform init -reconfigure".
╵
```

| Flag | Does | Use when |
|---|---|---|
| `-migrate-state` | **Copies** state from the old location to the new one. | Moving state (renaming a key, changing bucket, local → S3). |
| `-reconfigure` | **Forgets** the old backend settings; uses the new location as-is (doesn't copy). | The state is already at the new location, or you switched to a different environment's key on purpose. ⚠️ Pointing at an **empty** key this way makes Terraform think nothing exists. |

### Migrating back to local (rarely needed)

Remove the `backend` block, then `terraform init -migrate-state`: it offers to copy the state back into a local file.

---

## Hands-on lab

1. Create the bootstrap bucket. Check versioning, encryption and the bucket policy in the console.
2. Migrate the topic 34 lab (or the topic 27 stack) into the bucket with partial configuration. Verify with `plan`.
3. Apply a change and look at **S3 → bucket → object → Versions**: one version per apply.
4. While a long apply runs (add a `time_sleep`, topic 13), list the bucket: you'll see `terraform.tfstate.tflock` appear and disappear.
5. Try to `terraform destroy` the bootstrap config: `prevent_destroy` stops it.
6. Change the key and practise both `-migrate-state` and `-reconfigure` (on a throwaway key). Notice what `-reconfigure` to an empty key does to `terraform plan`.
7. Simulate a new teammate: `rm -rf .terraform` (keep the code), run `terraform init -backend-config=...`, then `terraform plan`: they see the same state with no local file.

Keep the bucket: topics 36–42 and the capstone use it. To delete it at the very end of the course: remove `prevent_destroy`, empty all versions, destroy.

---

## Where you'll use it

- **Every team project**: an S3 (or azurerm/gcs) backend with partial config per environment.
- **CI pipelines** (topics 57–58): the runner's role has access only to its environment's key prefix.
- **Account vending / landing zones**: each new AWS account gets its own state bucket from a bootstrap module.
- **Migrations**: consolidating or splitting states, moving between backends or to HCP Terraform.

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `Error: Failed to get existing workspaces: S3 bucket "x" does not exist` | Bucket missing or typo, or wrong region. | Create it (bootstrap); check `bucket`/`region`. |
| `AccessDenied` on `ListObjectsV2` / `GetObject` / `PutObject` | The identity lacks bucket/key permissions (or KMS permissions). | Grant the IAM actions above; check `AWS_PROFILE`. |
| `Backend configuration changed` | The backend block or `-backend-config` differs from `.terraform/terraform.tfstate`. | `init -migrate-state` (copy) or `init -reconfigure` (switch). |
| `Variables not allowed` in the backend block | Backends are configured before variables exist. | Partial config with `-backend-config`. |
| Plan wants to create everything after switching backends | You pointed at an empty key (`-reconfigure` to a new key, typo in `key`). | Fix the key; or migrate the state there. |
| `state data in S3 does not have the expected content ... checksum` | Leftover checksum from older DynamoDB-based setups or an interrupted write. | See topic 42 (compare versions; restore the right one). |
| A stray `terraform.tfstate` next to a remote-backend project | Leftover from before migration. | Confirm remote is correct, then delete it. |

---

## Quick recap

- A backend decides where state lives; use a remote one for anything shared: one copy, locking, encryption, access control, history.
- S3 backend: `bucket`, `key`, `region`, `encrypt = true`, `use_lockfile = true`. One key per environment × component.
- Backend blocks can't use variables → **partial configuration** with `-backend-config` files.
- Create the state bucket in a separate **bootstrap** configuration (versioned, encrypted, private, TLS-only, `prevent_destroy`).
- `init -migrate-state` copies state; `init -reconfigure` just switches. Back up with `state pull` first.

### Interview questions

1. **Q: Why use a remote backend?**
   A: To share one authoritative state across the team and CI, with locking to prevent concurrent writes, encryption and IAM for the secrets it contains, and versioning to recover from mistakes. Local state can't do any of that.
2. **Q: How do you use the same code with different state files for dev and prod?**
   A: Partial backend configuration: an empty or partial `backend "s3" {}` block, and per-environment `-backend-config` files with different keys (ideally different buckets/accounts), passed to `terraform init`.
3. **Q: How do you create the S3 bucket that stores state?**
   A: With a separate bootstrap configuration using local state (or a one-time CLI/CloudFormation step), with versioning, encryption, public access block, a TLS-only policy and `prevent_destroy`. Optionally migrate the bootstrap's own state into the bucket afterwards.
