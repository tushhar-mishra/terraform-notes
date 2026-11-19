# 15 · Outputs

> **Level:** ★ foundation
>
> **You will learn:** how to get values **out** of a configuration with `output` blocks: their settings (`value`, `description`, `sensitive`, `depends_on`, `precondition`), how they appear in plans ("Changes to Outputs"), where they're stored (state), how to read them with `terraform output` (`-raw`, `-json`), how scripts and pipelines use them, and the "Output refers to sensitive values" error.
>
> **You need:** topics 11 (references), 12 (state), 14 (variables, sensitive).
>
> **Part 2 · Making configuration flexible** · Previous: [14 · Input variables](14-input-variables.md) · Next: [16 · Locals](16-locals.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Variables are inputs; **outputs** are the configuration's **return values**: an IP address, a URL, a bucket name. Terraform prints them after `apply` and saves them in state, so people, scripts and other configurations can read them.

```hcl
output "config_path" {
  description = "Where the config file was written."
  value       = local_file.config.filename
}
```

```bash
terraform output                    # all outputs
terraform output -raw config_path   # just the value, no quotes: for scripts
terraform output -json | jq .       # everything as JSON
```

**Picture it.** The receipt the shop hands you after you pay: order number, collection time.

**The one thing to know.** Outputs are how Terraform hands information to the **next step**: an Ansible inventory, a CI job, a smoke test, another Terraform configuration.

**Try it now:** add the output above to your topic 14 lab, apply, then `cat "$(terraform output -raw config_path)"`.

**You've got it if you can:** explain the difference between `terraform output x` and `terraform output -raw x`, and why outputs from sensitive values need `sensitive = true`.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

```hcl
output "app_url" {                    # name: used in `terraform output app_url`
  description = "URL to open the app."
  value       = "http://localhost:${var.port}"   # any expression
  sensitive   = false                 # true = hidden in CLI output
}
```

| Setting | Purpose |
|---|---|
| `value` | **Required.** Any expression: a string, a number, a list, a whole resource object. |
| `description` | Explains it. Shows in docs and module registries. |
| `sensitive` | Hides it in plan/apply output and in `terraform output` (unless you ask for it by name or as JSON). Still stored in state. |
| `depends_on` | Rare. Makes the output wait for something it doesn't reference. |
| `precondition { }` | A check that must pass before the output is set (topic 25). |
| `ephemeral` | (1.10+) Only in **child modules**: pass a short-lived value without storing it (topic 51). |
| `type` | (1.15+) An explicit type constraint, mainly for module authors. |

Where outputs are used:

1. **Printed** after `apply` for humans.
2. **Stored** in state (`"outputs": {...}`, topic 12).
3. **Read by commands**: `terraform output`.
4. **Read by other configurations**: through `terraform_remote_state` (topic 38).
5. **Module return values**: a module's outputs are how its caller gets values (`module.network.vpc_id`, topic 43).

---

## Why it exists

After `apply`, you need things you couldn't know beforehand: the server's IP, the load balancer's DNS name, a generated bucket name. Without outputs you'd dig in the console or parse state. Outputs give a **stable, named interface**: scripts depend on `terraform output -raw app_url`, not on the internal structure of your resources. You can refactor the resources freely as long as the output keeps its name and meaning.

---

## Analogy

**A takeaway receipt.** You order (variables), the kitchen cooks (apply), and you get a receipt (outputs) with the order number and collection time. The delivery driver (the next pipeline step) only needs the receipt, not a tour of the kitchen. Some details on it are blurred out for privacy (`sensitive`), but the shop's own records (state) have them in full.

---

## How it works inside

```
plan:   evaluate each output's value → compare with the output stored in state
        → show "Changes to Outputs:"   (+ new, ~ changed, - removed)
        values not known yet → (known after apply)
apply:  outputs are evaluated LAST, after the resources they reference
        → written into state's "outputs" section with their sensitive flag
terraform output: reads state only. It never runs a plan or calls providers.
```

Two consequences:

- `terraform output` shows what was true **at the last apply**. If you add an output to code, it doesn't appear until you apply (or run `terraform apply -refresh-only`, which updates outputs and state without changing infrastructure).
- If a value includes anything sensitive and the output isn't marked `sensitive = true`, Terraform stops with an error. It won't let a secret slip out by accident.

---

## Code / commands

Extend your topic 14 project with `outputs.tf`:

```hcl
output "config_path" {
  description = "Where the config file was written."
  value       = local_file.config.filename
}

output "app_url" {
  description = "Where the app would be reachable."
  value       = "http://localhost:${var.port}"
}

output "summary" {
  description = "Everything about this deployment."
  value = {
    environment = var.environment
    port        = var.port
    features    = var.features
  }
}

output "api_token" {
  description = "The token (for the deploy script)."
  value       = var.api_token
  sensitive   = true
}
```

### In the plan

```
Changes to Outputs:
  + api_token   = (sensitive value)
  + app_url     = "http://localhost:8080"
  + config_path = "./dev.conf"
  + summary     = {
      + environment = "dev"
      + features    = []
      + port        = 8080
    }
```

### After apply

```
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

api_token = <sensitive>
app_url = "http://localhost:8080"
config_path = "./dev.conf"
summary = {
  "environment" = "dev"
  "features" = tolist([])
  "port" = 8080
}
```

- `<sensitive>`: hidden.
- `tolist([])`: Terraform shows the **type** of an empty collection so it isn't ambiguous.

### Reading outputs

```bash
terraform output app_url
terraform output -raw app_url
terraform output api_token
terraform output -raw api_token
terraform output -json | jq -r '.summary.value.port'
```

```
"http://localhost:8080"
http://localhost:8080
"dev-token-not-secret"
dev-token-not-secret
8080
```

| Form | Use |
|---|---|
| `terraform output NAME` | Human-readable, with quotes. |
| `terraform output -raw NAME` | Raw string/number, no quotes, no newline formatting. **Use in scripts.** Only works for strings, numbers and bools. |
| `terraform output -json` | All outputs with `value`, `type`, `sensitive` → for `jq`. **Includes sensitive values in clear text.** |

Terraform redacts sensitive outputs only when it prints **all** outputs (`terraform output`, and after `apply`). When you ask for one **by name**, or use `-raw` or `-json`, it prints the real value, because you asked explicitly. Be careful with these in CI logs.

`-json` output shape:

```json
{
  "app_url": {
    "sensitive": false,
    "type": "string",
    "value": "http://localhost:8080"
  },
  ...
}
```

### Using outputs in a script

```bash
#!/usr/bin/env bash
set -euo pipefail
URL=$(terraform output -raw app_url)
echo "Smoke-testing $URL"
curl -fsS --retry 5 "$URL/health" || { echo "health check failed"; exit 1; }
```

A pipeline step after `apply` would run this (topics 57–58).

### The sensitive-output error

Remove `sensitive = true` from `api_token` and plan:

```
╷
│ Error: Output refers to sensitive values
│
│   on outputs.tf line 21:
│   21: output "api_token" {
│
│ To reduce the risk of accidentally exporting sensitive data that was intended
│ to be only internal, Terraform requires that any root module output
│ containing sensitive data be explicitly marked as sensitive, to confirm your
│ intent.
│
│ If you do intend to export this data, annotate the output value as sensitive
│ by adding the following argument:
│     sensitive = true
╵
```

Terraform tracks sensitivity through every expression (topic 14), so it knows this output contains a sensitive variable.

---

## Hands-on lab

1. Add `outputs.tf` above to `~/tf-lab/14-vars`. `terraform apply -var-file=dev.tfvars`.
2. Read each output three ways: plain, `-raw`, `-json | jq`.
3. Write `smoke.sh` that reads `config_path` with `-raw` and prints the file with `cat`. Run it.
4. Remove `sensitive = true` from `api_token`, plan, read the error, put it back.
5. Add a new output `port_plus_one = var.port + 1` **without applying**, then run `terraform output`. It's missing. Run `terraform apply -refresh-only -var-file=dev.tfvars` (approve), and check again.
6. Look at the outputs in state: `jq '.outputs' terraform.tfstate`. Note `api_token` is there in clear text.
7. `terraform destroy -var-file=dev.tfvars`, then `terraform output`: `No outputs found`.

---

## Where you'll use it

- **Pipelines**: the deploy job reads `cluster_name`, `ecr_repository_url`, `alb_dns_name` from outputs.
- **Ansible**: Terraform outputs the server IPs; a script writes them into an inventory (or the `local_file` resource writes it directly).
- **Kubeconfig**: `aws eks update-kubeconfig --name "$(terraform output -raw cluster_name)"`.
- **Other configurations**: the network team's outputs (`vpc_id`, `subnet_ids`) are read by the app team (topic 38).
- **Modules**: outputs are a module's API (topic 44).

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `Output refers to sensitive values` | The value includes something sensitive. | Add `sensitive = true` (or don't output it). |
| `terraform output` doesn't show a new output | Outputs come from state; you haven't applied since adding it. | `apply` (or `apply -refresh-only`). |
| `Warning: No outputs found` | Nothing applied in this state, or wrong folder/workspace/backend. | Check where you are. |
| `-raw` fails: `Unsupported value for raw output ... the -raw option only supports strings, numbers, and boolean values` | The output is a list/map/object. | Use `-json` with `jq`. |
| Script gets `"value"` with quotes | Used `terraform output NAME` instead of `-raw`. | Use `-raw`. |
| A secret printed in CI logs | Someone used `terraform output -json` or `-raw` on a sensitive output in a logged step. | Don't echo it; mask it in CI ([GitHub Actions topic 17](../../GitHub-Actions-Basics-To-Production/notes/17-secrets-and-variables.md)); rotate it. |

---

## Quick recap

- `output` blocks are the configuration's return values; `value` is any expression.
- They're shown in plans ("Changes to Outputs"), printed after apply and **stored in state**.
- `terraform output` reads state: `-raw` for scripts, `-json` for structured data.
- Outputs containing sensitive values must be marked `sensitive = true`; they're still clear text in state and in `-json`.

### Interview questions

1. **Q: What are Terraform outputs used for?**
   A: Exposing values after apply: for humans, for scripts and pipelines (`terraform output -raw/-json`), as a module's return values, and for other configurations through `terraform_remote_state`.
2. **Q: Does `terraform output` call the cloud?**
   A: No. It reads output values from the current state, as of the last apply or refresh.
3. **Q: Why does Terraform require `sensitive = true` on some outputs?**
   A: Terraform tracks sensitive values through expressions. If a root output contains one, you must mark it explicitly so secrets aren't exported by accident.
