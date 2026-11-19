# 19 · Data Sources

> **Level:** ★ foundation
>
> **You will learn:** how Terraform **reads** information it doesn't manage, with `data` blocks: the syntax and the `data.TYPE.NAME.ATTR` reference, **when** data sources are read (at plan time, or **during apply** as `<= read`), how they appear in state, and three examples you can run locally: `local_file` (read a file), `http` (call a web API), and `external` (run a script, ⚠️ last resort). AWS examples like "find the latest Ubuntu AMI" follow in Part 4.
>
> **You need:** topics 11 (resources, references), 13 (plan walk), 17–18.
>
> **Part 2 · Making configuration flexible** · Previous: [18 · Built-in functions and `terraform console`](18-functions-and-console.md) · Next: [20 · Dependencies and the graph](20-dependencies-and-the-graph.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **resource** says "create and manage this". A **data source** says "**look this up** for me". Terraform reads it, never creates, changes or deletes it.

```hcl
data "http" "my_ip" {                          # TYPE "http", NAME "my_ip"
  url = "https://checkip.amazonaws.com"
}

resource "local_file" "ip" {
  filename = "${path.module}/my-ip.txt"
  content  = data.http.my_ip.response_body     # data.TYPE.NAME.ATTRIBUTE
}
```

**Picture it.** Resources are things you **build**. Data sources are things you **look up in the phone book**. You don't own the phone book; you just read a number from it.

**The one thing to know.** References start with `data.`: `data.http.my_ip.response_body`. Data sources are read **every** plan, so they always reflect the current world.

**Try it now:** run the code above (it needs the `hashicorp/http` provider in `required_providers`), then `cat my-ip.txt`.

**You've got it if you can:** explain resource vs data source, and why a data source sometimes shows `<= read (data resources)` in a plan.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

```hcl
data "<TYPE>" "<NAME>" {
  # query arguments: what to look for
}
# reference: data.<TYPE>.<NAME>.<ATTRIBUTE>
```

- **Type** comes from a provider, like resources: `aws_ami`, `aws_vpc`, `http`, `local_file`, `kubernetes_secret`... Many types exist as both a resource and a data source (`local_file` does).
- **Arguments** are the **query**: filters, names, IDs, URLs.
- **Attributes** are the **result**.
- They're shown in state with `"mode": "data"` (topic 12).

What you **can't** do with a data source: create, update or delete anything. If the thing doesn't exist, most data sources **fail** with an error ("no matching AMI found"), which is often what you want: fail early.

### Common data sources

| Provider | Data source | Looks up |
|---|---|---|
| `hashicorp/aws` | `aws_ami` | Latest image matching filters (topic 31) |
| `hashicorp/aws` | `aws_vpc`, `aws_subnets` | Network created by another team (topic 38) |
| `hashicorp/aws` | `aws_caller_identity`, `aws_region` | Current account ID and region |
| `hashicorp/aws` | `aws_availability_zones` | Zones available in the region |
| `hashicorp/aws` | `aws_iam_policy_document` | Builds a JSON policy in HCL |
| `hashicorp/aws` | `aws_secretsmanager_secret_version` | A secret (careful: stored in state; topic 51) |
| built-in `terraform` | `terraform_remote_state` | Another configuration's outputs (topic 38) |
| `hashicorp/http` | `http` | Any HTTP GET (or other method) |
| `hashicorp/local` | `local_file` | A file's content |
| `hashicorp/external` | `external` | Output of a script |

---

## Why it exists

- You need values that **someone else** owns: the company VPC (Virtual Private Cloud), a shared DNS zone, the latest approved image, the account ID.
- Hard-coding them (`subnet_id = "subnet-0a1b2c..."`) breaks when they change and differs per account/region. A lookup stays correct.
- They let configurations **cooperate** without one managing the other's resources.

---

## Analogy

**The phone book vs your own address book.**

- Your **resources** are entries you write yourself in your own address book: you add, change and delete them.
- **Data sources** are lookups in the public phone book: you read "the pizza shop's number" each time you need it. If the shop changes its number, your next lookup gets the new one. You can't edit the phone book, and if the shop doesn't exist, the lookup fails.

---

## How it works inside

**When is a data source read?**

```
plan walk reaches data.X
   │
   ├── all its arguments are KNOWN now, and it doesn't depend on a resource
   │   with pending changes
   │       → read it NOW, during plan (ReadDataSource on the provider)
   │       → its attributes are known; the plan uses real values
   │
   └── an argument is (known after apply), or it depends (depends_on or reference)
       on a resource that has changes planned
           → DEFER: plan shows " <= read (data resources)"
           → it's read during APPLY, after its dependencies
           → everything using its attributes is (known after apply) in the plan
```

- Data sources are **re-read on every plan/apply**. The result is saved in state, but the saved copy is just the last read: it's never trusted as the current truth.
- `terraform destroy` doesn't "destroy" data sources; they simply disappear from state.
- ⚠️ `depends_on` on a data source (topic 20) often forces it to be read during apply, which makes everything downstream unknown and can cause unexpected replacements. Prefer real references.

---

## Code / commands

`~/tf-lab/19-data/versions.tf`:

```hcl
terraform {
  required_version = ">= 1.14"
  required_providers {
    local    = { source = "hashicorp/local", version = "~> 2.5" }
    http     = { source = "hashicorp/http", version = "~> 3.5" }
    external = { source = "hashicorp/external", version = "~> 2.3" }
  }
}
```

### 1. `local_file`: read a file someone else manages

Create the file **outside** Terraform:

```bash
mkdir -p ~/tf-lab/19-data && cd ~/tf-lab/19-data
echo "team-platform" > owner.txt
```

```hcl
data "local_file" "owner" {
  filename = "${path.module}/owner.txt"
}
```

### 2. `http`: call a web API

```hcl
data "http" "my_ip" {
  url = "https://checkip.amazonaws.com"

  request_headers = {
    Accept = "text/plain"
  }

  lifecycle {
    postcondition {                          # fail if the API misbehaves (topic 25)
      condition     = self.status_code == 200  # self = this data source's own result
      error_message = "checkip returned ${self.status_code}."
    }
  }
}
```

### 3. `external`: run a script (last resort)

The script must read a JSON object on stdin and print a JSON object **of strings** on stdout:

`git-info.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
# stdin is the JSON "query"; we ignore it here
sha=$(git rev-parse --short HEAD 2>/dev/null || echo "no-git")
printf '{"sha":"%s"}\n' "$sha"
```

```hcl
data "external" "git" {
  program = ["bash", "${path.module}/git-info.sh"]
}
```

⚠️ `external` runs **any** program on the machine running Terraform, at every plan, with no state of its own. It breaks on machines without the same tools (Windows laptops, slim CI images). Use it only when no provider or function can do the job.

### Using them

```hcl
resource "local_file" "report" {
  filename = "${path.module}/report.txt"
  content  = <<-EOT
    owner:  ${trimspace(data.local_file.owner.content)}
    my ip:  ${trimspace(data.http.my_ip.response_body)}
    commit: ${data.external.git.result.sha}
  EOT
}
```

- `data.external.git.result` is a map of the strings the script printed.

### Plan when everything is known

```
$ terraform init && terraform plan
data.external.git: Reading...
data.local_file.owner: Reading...
data.http.my_ip: Reading...
data.local_file.owner: Read complete after 0s [id=4d1c...]
data.external.git: Read complete after 0s [id=-]
data.http.my_ip: Read complete after 1s [id=https://checkip.amazonaws.com]

Terraform will perform the following actions:

  # local_file.report will be created
  + resource "local_file" "report" {
      + content  = <<-EOT
            owner:  team-platform
            my ip:  203.0.113.45
            commit: no-git
        EOT
      ...
Plan: 1 to add, 0 to change, 0 to destroy.
```

- `Reading... / Read complete`: data sources were read **during plan**, so the plan shows real values.
- The data sources themselves aren't in the "will perform" list: they aren't changed.

### Plan when a data source depends on something unknown

Add a resource that **writes** a file, and a data source that **reads** it back:

```hcl
resource "local_file" "generated" {
  filename = "${path.module}/generated.txt"
  content  = "build-42"
}

data "local_file" "generated" {
  filename = local_file.generated.filename      # reference → depends on the resource
}

output "generated_back" {
  value = data.local_file.generated.content
}
```

```
  # data.local_file.generated will be read during apply
  # (config refers to values not yet known)
 <= data "local_file" "generated" {
      + content  = (known after apply)
      + filename = "./generated.txt"
      + id       = (known after apply)
      ...
    }

  # local_file.generated will be created
  + resource "local_file" "generated" {
...
Changes to Outputs:
  + generated_back = (known after apply)
```

- `<=` = **read during apply**.
- `(config refers to values not yet known)`: Terraform's reason for deferring.
- Once `local_file.generated` exists and has no pending changes, the next plan reads the data source at plan time again.

### See data sources in state

```bash
terraform apply -auto-approve
terraform state list
```

```
data.external.git
data.http.my_ip
data.local_file.generated
data.local_file.owner
local_file.generated
local_file.report
```

`data.` addresses are in state too, with `"mode": "data"`.

### A failing lookup

```bash
rm owner.txt && terraform plan
```

```
╷
│ Error: Read local file data source error
│
│   with data.local_file.owner,
│   on main.tf line 1, in data "local_file" "owner":
│    1: data "local_file" "owner" {
│
│ The file at given path cannot be read.
│
│ +Original Error: open ./owner.txt: no such file or directory
╵
```

The plan stops. That's a feature: Terraform refuses to plan with missing inputs.

---

## Hands-on lab

1. Build the project with all three data sources and `report`. Plan, read the `Reading...` lines, apply, `cat report.txt`.
2. Change `owner.txt` to `team-web` **outside** Terraform. Plan: `report` changes, because the data source re-reads every time.
3. Add the `generated` resource + data source pair. Find the `<=` line in the plan. Apply, then plan again: no `<=` any more.
4. Run `git init && git add . && git commit -m lab` in the folder and plan again: the commit SHA appears.
5. Delete `owner.txt` and read the error. Put it back.
6. Put a wrong URL in the `http` data source (e.g. `https://checkip.amazonaws.com/nope`) and see your `postcondition` message.
7. `terraform destroy`. Note no data source is "destroyed".

---

## Where you'll use it

- **Latest AMI**: `data "aws_ami"` with filters, so you don't hard-code image IDs (topic 31).
- **Shared networks**: look up the VPC and subnets created by the network team (topic 38).
- **Account and region**: `data.aws_caller_identity.current.account_id` in bucket names and policies.
- **Availability zones**: `data.aws_availability_zones.available.names` to spread subnets (topic 32).
- **IAM policies**: `data "aws_iam_policy_document"` to write policies in HCL.
- **Health checks**: `http` data source inside a `check` block (topic 25).

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `Your query returned no results` / `no matching ... found` | The lookup found nothing: wrong filters, wrong region/account, or the thing was deleted. | Fix the filters; check you're in the right region and account. |
| `multiple ... matched; use additional constraints` | The data source expects exactly one result. | Add filters (or use `most_recent = true` where offered). |
| Plan shows `<= read during apply` and many downstream `(known after apply)` / replacements | The data source depends on something with pending changes, often via `depends_on`. | Remove unnecessary `depends_on`; reference only what you need; split into two applies if unavoidable. |
| Value changes on every plan | The source changes (e.g. "latest" AMI, an API returning a timestamp). | Pin it, or use `ignore_changes` on the consumer (topic 24). |
| `external` fails in CI: `bash: not found` / `git: command not found` | The script depends on tools the runner doesn't have. | Avoid `external`; or install the tools in the runner image. |
| `Program output must be a JSON encoded map of string keys and string values` | The `external` script printed non-strings or extra text. | Print only one JSON object of strings; send logs to stderr. |

---

## Quick recap

- `data "TYPE" "NAME" { query }` **reads** existing things; reference with `data.TYPE.NAME.ATTR`.
- Read during **plan** when inputs are known, otherwise **during apply** (`<=`), making dependents unknown.
- Re-read every run; stored in state with `mode = "data"`; never created or destroyed.
- `http` and `local_file` are handy locally; ⚠️ `external` is a last resort.

### Interview questions

1. **Q: Resource vs data source?**
   A: A resource is something Terraform creates and manages through its life cycle. A data source only reads existing information (from a provider or another state) for use in the configuration; Terraform never changes it.
2. **Q: Why might a data source be read during apply instead of plan?**
   A: Its arguments depend on values not known until apply, or it depends on a resource with pending changes (including via `depends_on`). Terraform defers the read, showing `<= read during apply`, and anything using it becomes unknown in the plan.
3. **Q: How would you avoid hard-coding an AMI ID?**
   A: Use the `aws_ami` data source with owner and name filters and `most_recent = true`, and reference `data.aws_ami.x.id`.
