# 05 · Linux, Git and Cloud API Recap for Terraform

> **Level:** ★ foundation
>
> **You will learn:** exactly the Linux, Git and API knowledge Terraform depends on, and where Terraform uses each piece: `PATH` and environment variables (how credentials and `TF_*` settings reach Terraform), files and permissions (state files hold secrets), Git (what to commit and what to **never** commit), and **APIs** (what an API call is, HTTP methods, JSON, tokens). You'll finish by creating a Docker container **with raw API calls**, which is exactly what a Terraform provider does for you.
>
> **You need:** topics 01–04. Recaps of your [Linux notes](../../linux-mastery-rhel/INDEX.md), [Docker notes](../../Docker/INDEX.md) and [GitHub Actions Git recap](../../GitHub-Actions-Basics-To-Production/notes/02-git-and-github-recap.md).
>
> **Part 0 · Foundations** · Previous: [04 · The license change and OpenTofu](04-license-change-and-opentofu.md) · Next: [06 · Lab setup](06-lab-setup.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Terraform is "a program that reads files and calls APIs". So you need four things from earlier notes:

1. **Linux shell**: `PATH` (where the shell finds the `terraform` binary), **environment variables** (how you pass credentials and settings like `AWS_PROFILE` and `TF_LOG`), files and permissions.
2. **Git**: your `.tf` files live in Git. **State files, `.terraform/` folders and secret `.tfvars` files never do.**
3. **HTTP/REST APIs**: every cloud action is an HTTP request with a method (GET/POST/PUT/DELETE), a URL, a JSON body and a credential. Providers make these calls.
4. **JSON**: state files, plan output for machines (`-json`) and API bodies are all JSON.

**Picture it.** Terraform is a **very polite robot clerk**: it reads your forms (files), finds its ID badge in its pocket (environment variables), walks to the right office (API URL), and files requests (HTTP calls). Git is the filing cabinet for the forms, not for the clerk's receipts (state).

**The one thing to know.** Talk to an API yourself once. With Docker running in WSL2:

```bash
curl -s --unix-socket /var/run/docker.sock http://localhost/version | jq .Version
```

That's what Terraform's Docker provider does under the hood.

**Try it now:** run `echo $PATH`, `env | grep -i aws`, and the `curl` line above.

**You've got it if you can:** explain why `terraform.tfstate` must be in `.gitignore`, and what HTTP method creating a resource usually uses.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### 1. Linux shell pieces Terraform relies on

**`PATH`** ([Linux topic 012](../../linux-mastery-rhel/notes/012-variables-and-environment.md)) is the list of folders the shell searches when you type a command name. `terraform` works only if the folder holding the binary (usually `/usr/bin` or `/usr/local/bin`) is in `PATH`.

**Environment variables** are `NAME=value` pairs every process inherits from its parent shell. Terraform and its providers read a lot of them:

| Variable | Read by | Purpose | Topic |
|---|---|---|---|
| `TF_LOG`, `TF_LOG_PATH` | Terraform core | Debug logging | 64 |
| `TF_VAR_<name>` | Terraform core | Set an input variable | 14 |
| `TF_CLI_ARGS_plan` etc. | Terraform core | Extra flags for a command | 09 |
| `TF_IN_AUTOMATION` | Terraform core | Shorter output in CI | 56 |
| `TF_PLUGIN_CACHE_DIR` | Terraform core | Share downloaded providers between projects | 10 |
| `AWS_PROFILE`, `AWS_REGION`, `AWS_ACCESS_KEY_ID`... | AWS provider | Credentials and region | 30 |
| `DOCKER_HOST` | Docker provider | Where the Docker daemon is | 26 |
| `KUBE_CONFIG_PATH` | Kubernetes provider | Which kubeconfig file | 28 |

Set one for a single command, or for the whole session:

```bash
TF_LOG=DEBUG terraform plan     # only this command sees it
export AWS_PROFILE=lab          # every later command in this shell sees it
```

**Files and permissions** ([Linux topic 026](../../linux-mastery-rhel/notes/026-file-permissions-and-umask.md)): Terraform writes `terraform.tfstate` in your project folder. It can contain **passwords and keys in plain text** (topic 51). On a shared machine it should be readable only by you (`chmod 600`), and in teams it should live in an encrypted remote backend (topic 35), not on laptops.

**Exit codes** ([Linux topic 045](../../linux-mastery-rhel/notes/045-robust-scripts.md)): every command returns a number; `0` = success. CI pipelines stop on a non-zero exit code. `terraform plan -detailed-exitcode` returns `0` (no changes), `1` (error) or `2` (changes present), which pipelines and drift checks use (topics 37, 56).

### 2. Git pieces Terraform relies on

From your [GitHub Actions Git recap](../../GitHub-Actions-Basics-To-Production/notes/02-git-and-github-recap.md) and [Jenkins Git essentials](../../Jenkins/notes/05-git-essentials-for-ci-cd.md): a **repository** holds files and their history; a **commit** is a saved snapshot with an author and message; a **branch** is a line of work; a **PR (Pull Request)** asks to merge a branch after review; **tags** name a commit (e.g. `v1.2.0`), which is how you'll version modules (topic 45).

For Terraform, Git holds the **desired state**. Some files must **never** be committed:

| Path | What it is | Commit? |
|---|---|---|
| `*.tf` | Your configuration | ✅ Yes |
| `.terraform.lock.hcl` | Exact provider versions and checksums (topic 10) | ✅ **Yes** (people often get this wrong) |
| `.terraform/` | Downloaded providers and modules (big binaries, rebuilt by `init`) | ❌ No |
| `terraform.tfstate`, `terraform.tfstate.backup` | State: may contain secrets; must live in a backend | ❌ **Never** |
| `*.tfplan`, `tfplan` | Saved plan files: may contain secrets | ❌ No |
| `*.tfvars` with secrets | Variable values | ❌ No (non-secret ones like `dev.tfvars` are fine) |
| `crash.log`, `crash.*.log` | Crash dumps | ❌ No |
| `.terraformrc`, `terraform.rc` | CLI config, may contain tokens | ❌ No |

### 3. APIs: what providers actually do

An **API (Application Programming Interface)** is a way for one program to ask another program to do something. Cloud APIs are almost all **HTTP APIs**, often in **REST** style (REST = Representational State Transfer: each "thing" has a URL, and you use HTTP methods on it). Recap from [Linux topic 070](../../linux-mastery-rhel/notes/070-http-and-curl.md):

| HTTP method | Meaning | Terraform action that uses it |
|---|---|---|
| `GET` | Read | Refresh: "does this still exist, what are its settings?" |
| `POST` | Create | Create a resource |
| `PUT` / `PATCH` | Replace / partly change | Update in place |
| `DELETE` | Delete | Destroy |

Every request has:

- a **URL** (which thing),
- a **method** (what to do),
- **headers** (including the **credential**: a token, or an AWS **signature** computed from your secret key),
- often a **JSON body** (the settings).

The response has a **status code** (`200 OK`, `201 Created`, `400 Bad Request`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `429 Too Many Requests`) and a JSON body. Terraform errors like `403 AccessDenied` or `429 Throttling` are just these codes passed back by the provider (topic 65).

**JSON** (JavaScript Object Notation) is text like `{"name": "web", "ports": [80, 443]}`. `jq` is the command-line tool for reading it; you'll use it with `terraform output -json` and `terraform show -json`.

---

## Why it exists

Terraform hides API calls behind HCL, but when something fails, the error **is** an API error: a 403 from AWS, a 409 from Docker, a socket permission error from Linux. If you know what's underneath, you can debug. If you don't, every error looks like magic.

---

## Analogy

A **courier company**:

- **Environment variables** = the courier's ID badge and the day's instructions in their pocket.
- **`PATH`** = the list of depots where the courier looks for their van.
- **The API** = the reception desk at each customer's office: you hand in a form (JSON body) of a certain kind (method) at a certain desk (URL), show your badge (credential), and get a receipt (status code).
- **Git** = the company archive of order forms. Receipts with customers' private details (state) go in a locked safe, not the public archive.

---

## How it works inside

What happens between `terraform apply` and Docker creating a container:

```
terraform (core)
   │ reads env: TF_LOG, TF_VAR_*...
   │ starts provider process: .terraform/providers/.../terraform-provider-docker
   ▼
Docker provider (child process; inherits the environment: DOCKER_HOST...)
   │ POST /containers/create {"Image":"nginx:1.29", ...}
   │ POST /containers/<id>/start
   ▼  over the Unix socket /var/run/docker.sock (Linux permissions apply:
dockerd      your user must be in the docker group, as in Docker topic 04)
   │ 201 Created {"Id":"7f3c..."}
   ▼
provider returns the ID → core writes it into terraform.tfstate
```

For AWS it's the same shape, but over HTTPS to `https://ec2.<region>.amazonaws.com/` with a signed request.

---

## Code / commands

### Environment and PATH

```bash
which terraform          # where the shell found it (after topic 06)
echo "$PATH" | tr ':' '\n'
env | grep -E '^(TF_|AWS_)' || echo "no TF_/AWS_ vars set"
```

```
/usr/bin/terraform
/usr/local/sbin
/usr/local/bin
/usr/sbin
/usr/bin
...
no TF_/AWS_ vars set
```

- `which` prints the first match in `PATH`. If you have two Terraform versions, this tells you which one runs.
- `|| echo ...` runs only if `grep` found nothing (exit code 1).

### A Terraform `.gitignore`

```gitignore
# Local .terraform directories (providers and modules, rebuilt by terraform init)
**/.terraform/*

# State files: never in Git
*.tfstate
*.tfstate.*

# Saved plans
*.tfplan
tfplan

# Crash logs
crash.log
crash.*.log

# Variable files that hold secrets: our convention is to name them *.secret.tfvars
*.secret.tfvars
*.secret.tfvars.json

# CLI config
.terraformrc
terraform.rc

# Override files (local-only tweaks)
override.tf
override.tf.json
*_override.tf
*_override.tf.json
```

Notes:

- `.terraform.lock.hcl` is **not** ignored. Commit it.
- GitHub's official `Terraform.gitignore` template ignores **all** `*.tfvars` files. That's the safest default when you don't know what people will put in them. These notes commit non-secret files like `dev.tfvars` (topic 49) and keep secrets out of `.tfvars` entirely (topic 51), so they only ignore the `*.secret.tfvars` naming convention. Pick one rule per repo and write it down.

### Calling an API by hand: Docker

Docker's daemon exposes a REST API on a Unix socket ([Docker topic 03](../../Docker/notes/03-docker-architecture.md)). The `docker` CLI and the Terraform Docker provider both use it.

```bash
# GET: read the daemon version
curl -s --unix-socket /var/run/docker.sock http://localhost/version | jq '{Version, ApiVersion}'
```

```json
{
  "Version": "29.0.2",
  "ApiVersion": "1.52"
}
```

```bash
# POST: create a container (like "terraform apply" creating docker_container)
docker pull -q nginx:1.29
curl -s --unix-socket /var/run/docker.sock \
  -H 'Content-Type: application/json' \
  -d '{"Image":"nginx:1.29"}' \
  -X POST "http://localhost/containers/create?name=api-demo" | jq .
```

```json
{
  "Id": "7f3c9a1e2b4d...",
  "Warnings": []
}
```

- `-X POST`: the method, "create".
- `?name=api-demo`: the container name, in the URL.
- `-d '{...}'`: the JSON body: the settings.
- The response `Id` is what a provider would save into state.

```bash
# POST: start it, then GET its state, then DELETE it (like "terraform destroy")
curl -s -o /dev/null -w '%{http_code}\n' --unix-socket /var/run/docker.sock -X POST http://localhost/containers/api-demo/start
curl -s --unix-socket /var/run/docker.sock http://localhost/containers/api-demo/json | jq .State.Status
curl -s -o /dev/null -w '%{http_code}\n' --unix-socket /var/run/docker.sock -X DELETE "http://localhost/containers/api-demo?force=true"
```

```
204
"running"
204
```

- `204` = "No Content": success with no body.
- Run the create command twice and the second gives `409` (`Conflict. The container name "/api-demo" is already in use`). That's the non-idempotent API underneath. A Terraform provider handles this by reading state first and only creating what's missing.

---

## Hands-on lab

1. Check your environment: `echo $SHELL`, `echo $PATH`, `id` (is your user in the `docker` group? If not, see [Docker topic 04](../../Docker/notes/04-installing-docker.md)).
2. Run the four Docker API calls above. Then run the create call twice to see the `409`.
3. Make a Git repo for your labs:

   ```bash
   mkdir -p ~/tf-lab && cd ~/tf-lab
   git init
   # paste the .gitignore above into ~/tf-lab/.gitignore
   echo '{}' > terraform.tfstate
   git status --short       # terraform.tfstate must NOT appear
   rm terraform.tfstate
   ```

4. If you have an AWS account already, run `aws sts get-caller-identity --debug 2>&1 | grep -E 'POST|https://'` to see the CLI's HTTPS request. (Otherwise skip; topic 29 sets up AWS.)

---

## Where you'll use it

- **Credentials** for every provider come through environment variables or files (topic 30).
- **CI pipelines** ([GitHub Actions topic 04](../../GitHub-Actions-Basics-To-Production/notes/04-linux-shell-recap-for-runners.md)) run `terraform` in a Linux shell and depend on exit codes and environment variables.
- **Scripts** read `terraform output -json` with `jq`.
- **Debugging** (topic 64): `TF_LOG=DEBUG` shows the HTTP requests providers send.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `terraform: command not found` | Binary not installed, or its folder isn't in `PATH`. | Install (topic 06), or check `PATH`. |
| `permission denied while trying to connect to the Docker daemon socket` | Your user can't access `/var/run/docker.sock`. | Add yourself to the `docker` group and log in again (Docker topic 04). |
| `terraform.tfstate` shows up in `git status` | Missing `.gitignore` rule. | Add it. If it was **already committed**, remove it from Git **and rotate any secrets in it**: they're in history forever. |
| `.terraform.lock.hcl` ignored by mistake | Teammates and CI get different provider versions. | Remove it from `.gitignore` and commit it. |
| `jq: command not found` | Not installed. | `sudo apt install -y jq` (Ubuntu) / `sudo dnf install -y jq` (RHEL). |

---

## Quick recap

- Terraform finds itself via `PATH`, and gets settings and credentials from **environment variables** (`TF_*`, `AWS_*`...).
- **Commit** `*.tf` and `.terraform.lock.hcl`. **Never commit** state, `.terraform/`, plans or secret `.tfvars`.
- Providers make **HTTP API calls**: GET to read, POST to create, PUT/PATCH to update, DELETE to destroy. Errors are status codes passed back.
- State, plans and API bodies are **JSON**; use `jq` to read them.

### Interview questions

1. **Q: Which Terraform files go in Git and which don't?**
   A: Commit `.tf` files and `.terraform.lock.hcl`. Don't commit `.terraform/`, state files, saved plans, crash logs or `.tfvars` with secrets. State belongs in a remote backend.
2. **Q: How does a Terraform provider create a resource?**
   A: It translates the resource's arguments into API requests (usually an HTTP POST with a JSON body, authenticated with your credentials) to the platform's API, then returns the new object's ID and attributes to core, which saves them in state.
3. **Q: What does `terraform plan -detailed-exitcode` return?**
   A: 0 for no changes, 1 for an error, 2 for a successful plan with changes. Useful for drift detection and CI logic.
