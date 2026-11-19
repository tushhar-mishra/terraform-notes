# 51 · Secrets and Sensitive Data

> **Level:** ★★★ advanced
>
> **You will learn:** every place a secret can leak in a Terraform workflow (code, tfvars, state, plan files, logs, outputs, CI), why **`sensitive`** isn't protection, and the modern tools that keep secrets **out of state entirely**: **ephemeral variables, ephemeral resources and ephemeral outputs** (1.10+), **write-only arguments** (1.11+), and letting the **cloud generate and hold** secrets (RDS `manage_master_user_password`). Plus secret stores (AWS Secrets Manager, SSM, Vault), state protection (encryption, access, OpenTofu's state encryption), and CI handling.
>
> **You need:** topics 12 (secrets in state), 14–15 (`sensitive`), 19 (data sources), 30, 35 (state bucket), 47.
>
> **Part 8 · Security and quality** · Previous: [50 · HCP Terraform and other platforms](50-hcp-terraform-and-platforms.md) · Next: [52 · Provisioners, and what to use instead](52-provisioners-and-alternatives.md) · [Back to INDEX](../INDEX.md)
>
> 💰 **Cost:** one Secrets Manager secret for an hour or so: a few cents at most (USD 0.40/secret/month, prorated). Delete it at the end.

---

## ⚡ Core: the 10-minute version

**The idea.** Terraform stores **everything it knows** in state and plan files. So the goal isn't to "hide" secrets; it's to make sure Terraform **never stores** them, or never **knows** them at all.

| Best → worst | Example |
|---|---|
| 1. **The cloud generates and keeps the secret**; Terraform never sees it | `aws_db_instance { manage_master_user_password = true }` |
| 2. **Ephemeral value + write-only argument**: Terraform passes it through but never stores it | `ephemeral "random_password"` → `password_wo` |
| 3. **Read from a secret store at apply time**, ephemerally | `ephemeral "aws_secretsmanager_secret_version"` |
| 4. `sensitive = true` | Hidden in output, **still in state in plain text** |
| 5. ❌ In code or committed tfvars | Leaked forever in Git history |

```hcl
ephemeral "random_password" "db" {          # generated during the run, never saved
  length = 24
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id                = aws_secretsmanager_secret.db.id
  secret_string_wo         = ephemeral.random_password.db.result   # write-only: not stored
  secret_string_wo_version = 1                                     # bump to rotate
}
```

**Picture it.** A bank courier who carries the cash from the vault to the till but **never writes the amount in his notebook** (ephemeral + write-only). Even better: the till generates its own cash and the courier never touches it (cloud-managed secrets).

**The one thing to know.** `sensitive = true` only stops **printing**. Anyone who can read state or a plan file can read every "sensitive" value.

**Try it now:** do the lab: create a secret with a write-only argument, then `terraform state pull | grep` for the password: it isn't there.

**You've got it if you can:** explain ephemeral vs sensitive, and name three places secrets leak besides code.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Where secrets leak

| Place | How | Protection |
|---|---|---|
| `.tf` code | Hard-coded `password = "..."` | Never. Use inputs or generation. |
| `.tfvars` files | Committed to Git | Keep secrets out of tfvars; `TF_VAR_` from a secret store; `.gitignore` (topic 05) |
| **State** | Every attribute of every resource and data source | Ephemeral/write-only; cloud-managed secrets; encrypted, access-controlled backend |
| **Saved plan files** | Contain config, variable values and planned attributes | Treat as secrets; short-lived CI artifacts with restricted access |
| **CLI output / CI logs** | Printing outputs, `TF_LOG=TRACE`, `terraform output -raw/-json` | `sensitive`; CI masking; never `TRACE` in shared logs |
| **Outputs** | Exposed to `terraform_remote_state` consumers | Don't output secrets; share via secret stores |
| **`user_data`** | Visible to anyone with `ec2:DescribeInstanceAttribute` | Fetch secrets at boot from Secrets Manager with the instance role |
| **Provider debug logs** | Request/response bodies | Avoid `TRACE` with real secrets; scrub before sharing |

### `sensitive` (recap: topics 14–15)

Marks a value so the CLI shows `(sensitive value)`. Values derived from it are sensitive too. **Stored in state and plans in plain text.**

### Ephemeral values (Terraform 1.10+)

**Ephemeral** = exists only during **one** plan or apply, never written to state or plan files.

| Feature | Syntax | Notes |
|---|---|---|
| **Ephemeral input variable** | `variable "token" { ephemeral = true }` | Can only be used where ephemeral values are allowed |
| **Ephemeral resource** | `ephemeral "TYPE" "NAME" { ... }`, referenced as `ephemeral.TYPE.NAME.attr` | Opened/read during the run, closed at the end; never in state. Examples: `aws_secretsmanager_secret_version`, `aws_ssm_parameter`, `random_password` (random ≥ 3.7), Vault secrets, Kubernetes tokens |
| **Ephemeral output** | `output "x" { ephemeral = true }` | **Child modules only**, to pass ephemeral values to the caller |
| **Ephemeral locals** | A local that uses an ephemeral value is ephemeral | |

**Where ephemeral values may go**: provider configuration blocks (e.g. a database provider's password), other ephemeral resources, **write-only arguments**, ephemeral outputs/locals, provisioner/connection blocks. They may **not** go into normal resource arguments, because those are stored in state. Terraform refuses with a clear error.

### Write-only arguments (Terraform 1.11+)

Resource arguments that the provider **accepts but never stores**:

- Named with a `_wo` suffix by convention: `password_wo` (`aws_db_instance`, `aws_rds_cluster`), `secret_string_wo` (`aws_secretsmanager_secret_version`), `value_wo` (`aws_ssm_parameter`)...
- Usually paired with a **version** argument (`password_wo_version`). Since Terraform can't compare a value it never stored, it can't detect changes; you **bump the version** to send a new value (rotation).
- Accept ephemeral **and** normal values; shown in plans as `(write-only attribute)`.

### Let the cloud own the secret

For many services, the best option is to never generate the secret in Terraform at all:

```hcl
resource "aws_db_instance" "main" {
  # ...
  username                    = "app_admin"
  manage_master_user_password = true      # RDS generates the password, stores and rotates it
}                                          # in Secrets Manager; Terraform never sees it

output "db_secret_arn" {
  value = aws_db_instance.main.master_user_secret[0].secret_arn   # the ARN is not a secret
}
```

Apps read the secret at runtime with their IAM role.

### Secret stores

| Store | Terraform usage |
|---|---|
| **AWS Secrets Manager** | Create the secret container with Terraform; set the value with write-only/ephemeral; apps read it at runtime. Rotation built in. |
| **AWS SSM Parameter Store** (`SecureString`) | Similar, cheaper, simpler; `value_wo` write-only argument. |
| **HashiCorp Vault** | `hashicorp/vault` provider: ephemeral resources for dynamic credentials (e.g. short-lived DB users), KV secrets. |
| **External Secrets Operator / CSI driver** (Kubernetes) | Terraform creates the store and IAM; the operator syncs secrets into pods (topic 28 hand-off). |

### Protecting state itself

Even with all of the above, state still holds sensitive metadata (ARNs, IPs, some keys). Protect it:

- Remote backend with **encryption** (SSE-S3 or SSE-KMS with a key whose policy limits who can decrypt), TLS-only, **no public access**, versioning (topic 35).
- **Least-privilege access**: only the pipeline role (and break-glass admins) can read prod state.
- **OpenTofu** offers **client-side state encryption** (topic 04): state is encrypted before it leaves the machine.
- In HCP Terraform, state is encrypted at rest and access is controlled per workspace.

### In CI

- Secrets reach Terraform as **environment variables** (`TF_VAR_x`, provider env vars) from the CI secret store ([GitHub Actions topic 17](../../GitHub-Actions-Basics-To-Production/notes/17-secrets-and-variables.md), [Jenkins topic 24](../../Jenkins/notes/24-using-credentials-safely.md)), or better, from **OIDC** roles with no stored secrets at all (topic 57).
- Mask values; never print `terraform output -json` in logs; keep plan artifacts private and short-lived.

---

## Why it exists

Infrastructure is full of secrets: database passwords, API tokens, TLS keys. Terraform's model (state remembers everything) is the opposite of what secrets need. For years the only answer was "protect your state very well". Ephemeral values and write-only arguments (2024–2025) finally let Terraform **use** a secret without **keeping** it.

---

## Analogy

**Moving cash in a bank.**

- **Hard-coded secret** = writing the vault code on the vault door.
- **`sensitive`** = writing the cash amount in the courier's notebook in **pencil** and putting a sticky note over it. Anyone who lifts the note reads it.
- **Ephemeral + write-only** = the courier carries the cash from the vault to the till and **never writes the amount down**. Next time, if the amount must change, the bank issues "delivery #2" (the version number).
- **Cloud-managed secret** = the till prints its own cash. The courier never handles it.
- **State encryption and access control** = the notebook itself is locked in a safe, just in case.

---

## How it works inside

```
plan/apply
  ephemeral "random_password" "db"
     → provider OpenEphemeralResource → value lives in memory for this run only
  aws_secretsmanager_secret_version.db
     secret_string_wo = <ephemeral value>
     → plan shows "(write-only attribute)"; planned state stores NULL for it
     → apply: provider sends the value to AWS (PutSecretValue)
     → state records secret_string_wo = null, secret_string_wo_version = 1
  end of run → provider CloseEphemeralResource; value discarded

next plan: secret_string_wo_version unchanged → no change (Terraform can't see the value)
           bump to 2 → provider sends a new value → rotation
```

Terraform enforces the rules at plan time: referencing an ephemeral value in a normal argument fails with `Invalid use of ephemeral value`.

---

## Code / commands

`~/tf-lab/51-secrets/` with `aws ~> 6.0` and `random ~> 3.7` in `required_providers`, plus the usual provider block.

```hcl
resource "random_id" "suffix" {
  byte_length = 3
}

# ❌ the old way, for comparison: generated and STORED in state
resource "random_password" "old" {
  length  = 20
  special = false
}

# ✅ generated during the run, NEVER stored
ephemeral "random_password" "db" {
  length  = 24
  special = false
}

resource "aws_secretsmanager_secret" "db" {
  name                    = "tf-course/db-password-${random_id.suffix.hex}"
  recovery_window_in_days = 0              # lab: delete immediately on destroy (default is 30 days)
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id                = aws_secretsmanager_secret.db.id
  secret_string_wo         = ephemeral.random_password.db.result
  secret_string_wo_version = 1
}

output "secret_arn" {
  value = aws_secretsmanager_secret.db.arn
}
```

### Plan

```
  # aws_secretsmanager_secret_version.db will be created
  + resource "aws_secretsmanager_secret_version" "db" {
      + arn                      = (known after apply)
      + id                       = (known after apply)
      + secret_id                = (known after apply)
      + secret_string_wo         = (write-only attribute)
      + secret_string_wo_version = 1
      + version_id               = (known after apply)
      + version_stages           = (known after apply)
    }
...
  # random_password.old will be created
  + resource "random_password" "old" {
      + result = (sensitive value)
      ...
```

- `(write-only attribute)`: sent at apply, never stored.
- The ephemeral resource isn't listed as a change: it isn't managed and isn't in state.

### Apply and inspect state

```bash
terraform apply -auto-approve
terraform state list
terraform state pull | jq -r '.resources[] | select(.type=="random_password") | .instances[0].attributes.result'
terraform state pull | jq '.resources[] | select(.type=="aws_secretsmanager_secret_version") | .instances[0].attributes | {secret_string, secret_string_wo, secret_string_wo_version}'
```

```
aws_secretsmanager_secret.db
aws_secretsmanager_secret_version.db
random_id.suffix
random_password.old

Q8vZ2mKp0xLr7TbN3wYc          # ← the OLD way: password readable from state

{
  "secret_string": "",
  "secret_string_wo": null,
  "secret_string_wo_version": 1
}
```

- No `ephemeral.random_password.db` in `state list`.
- The secret version stores **no value**.

Confirm the secret exists in AWS (with your admin credentials):

```bash
aws secretsmanager get-secret-value --secret-id "$(terraform output -raw secret_arn)" --query SecretString --output text
```

### Rotate

Bump `secret_string_wo_version = 2` and apply: a new ephemeral password is generated and written. Changing the ephemeral resource alone (without the version) does nothing, because Terraform can't see the old value to compare.

### Reading a secret ephemerally (for a provider)

```hcl
ephemeral "aws_secretsmanager_secret_version" "db" {
  secret_id = aws_secretsmanager_secret.db.id
}

provider "postgresql" {                          # e.g. the community postgresql provider
  host     = var.db_host
  username = "app_admin"
  password = ephemeral.aws_secretsmanager_secret_version.db.secret_string   # allowed: provider config
}
```

### The error that protects you

```hcl
resource "local_file" "leak" {
  filename = "pw.txt"
  content  = ephemeral.random_password.db.result
}
```

```
│ Error: Invalid use of ephemeral value
│
│   with local_file.leak,
│   on main.tf line 40, in resource "local_file" "leak":
│   40:   content  = ephemeral.random_password.db.result
│
│ Ephemeral values are not valid in resource arguments, because resource
│ instances must persist between Terraform phases.
```

### Clean up

```bash
terraform destroy -auto-approve
```

`recovery_window_in_days = 0` deletes the secret immediately (production keeps the default recovery window).

---

## Hands-on lab

1. Run the lab; compare `random_password.old` (readable in state) with the write-only secret (not stored).
2. Rotate by bumping the version; read the new value with the CLI.
3. Trigger `Invalid use of ephemeral value`.
4. Make a variable `variable "api_token" { type = string, ephemeral = true }` and pass it via `TF_VAR_api_token`. Try to use it in a normal resource argument (error), then in a write-only argument (`value_wo` of an `aws_ssm_parameter` with `value_wo_version`).
5. Read about `manage_master_user_password` on `aws_db_instance` in the provider docs; note which attributes appear in state (only the secret's ARN).
6. Destroy everything.

---

## Where you'll use it

- **Databases**: `manage_master_user_password` (best) or `password_wo` from an ephemeral value.
- **API keys and tokens** stored in Secrets Manager/SSM with write-only values.
- **Providers needing credentials** (database, Vault, Kubernetes) configured from ephemeral resources.
- **Security reviews**: "Is any secret in our state? Who can read state? Are plans stored?"

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `Invalid use of ephemeral value` | Ephemeral value in a normal argument. | Use a write-only argument, a provider config, or an ephemeral output. |
| Rotation "does nothing" | Write-only value changed, version not bumped. | Increment the `*_wo_version` argument. |
| `Unsupported argument "password_wo"` | Old provider or Terraform < 1.11. | Upgrade both. |
| `Unsupported block type "ephemeral"` | Terraform < 1.10, or the provider has no such ephemeral resource. | Upgrade; check the provider docs' "Ephemeral Resources" section. |
| Secret visible in CI logs | Printed outputs, `TRACE` logs, unmasked env vars. | Masking; don't print; rotate the secret. |
| Old secrets still in state history | S3 versions of past states contain them. | Rotate the secrets; lifecycle-expire old state versions (topic 35). |
| `You can't create this secret because a secret with this name is already scheduled for deletion` | The default 30-day recovery window. | Use a new name, or restore/force-delete the old one. |

---

## Quick recap

- Secrets leak through code, tfvars, **state**, **plan files**, logs and outputs. `sensitive` only hides printing.
- Prefer: **cloud-managed secrets** (RDS `manage_master_user_password`) > **ephemeral values + write-only arguments** (1.10/1.11+) > reading secret stores ephemerally > `sensitive`.
- Write-only arguments need a **version** argument to rotate.
- Still protect state: encryption, least-privilege access, versioning with expiry; OpenTofu adds client-side state encryption.

### Interview questions

1. **Q: Does `sensitive = true` keep a secret safe?**
   A: No. It only hides the value in CLI output. The value is still stored in plain text in the state and in plan files. Use ephemeral values, write-only arguments or cloud-managed secrets to keep it out of state, and protect state access.
2. **Q: What are ephemeral resources and write-only arguments?**
   A: Ephemeral resources (1.10+) produce values that exist only during a run and are never stored in state or plans. Write-only arguments (1.11+) accept values that are sent to the provider but never persisted; a version argument triggers updates. Together they let Terraform use a secret without storing it.
3. **Q: How would you manage an RDS master password with Terraform?**
   A: Set `manage_master_user_password = true` so RDS generates and rotates it in Secrets Manager and Terraform never sees it; apps read it via IAM. Alternatively, generate it with an ephemeral resource and pass it through `password_wo` with `password_wo_version`.
