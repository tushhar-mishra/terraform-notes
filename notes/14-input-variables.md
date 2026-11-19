# 14 · Input Variables

> **Level:** ★ foundation
>
> **You will learn:** how to make one configuration work for many situations with **input variables**: the `variable` block and its settings (`type`, `default`, `description`, `validation`, `sensitive`, `nullable`, `deprecated`), the six ways to give a variable a value (`default`, `terraform.tfvars`, `*.auto.tfvars`, `-var-file`, `-var`, `TF_VAR_*`, and the prompt), the exact **precedence order** when several are set, and the errors you'll meet.
>
> **You need:** topics 08 (HCL), 09 (commands), 11 (references).
>
> **Part 2 · Making configuration flexible** · Previous: [13 · What happens inside `plan` and `apply`](13-inside-plan-and-apply.md) · Next: [15 · Outputs](15-outputs.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **variable** is an input to your configuration, like a function parameter. You declare it once, use it as `var.<name>`, and give it a different value per environment.

```hcl
variable "environment" {
  type        = string
  description = "Which environment this is."
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev, staging or prod."
  }
}

resource "local_file" "config" {
  filename = "${path.module}/${var.environment}.conf"
  content  = "env=${var.environment}\n"
}
```

```bash
terraform apply                             # uses the default: dev
terraform apply -var="environment=prod"     # overrides it
```

**Picture it.** A coffee machine: same machine (code), different buttons (variables) for espresso or latte.

**The one thing to know.** When a variable is set in several places, the **later** one in this list wins: `default` → `TF_VAR_` env var → `terraform.tfvars` → `*.auto.tfvars` → `-var`/`-var-file` on the command line.

**Try it now:** set `environment` with `TF_VAR_environment=staging` **and** `-var="environment=prod"` at the same time. Which wins? (`prod`: the command line beats environment variables.)

**You've got it if you can:** list the precedence order and write a variable with a type, a default and a validation.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### The `variable` block

```hcl
variable "instance_count" {     # the name: used as var.instance_count
  type        = number          # what kind of value is allowed (topic 17)
  description = "How many app instances to run."
  default     = 1               # makes it optional. No default = required
  nullable    = false           # null is not accepted (default: true)
  sensitive   = false           # true = hide the value in plan/apply output

  validation {                  # custom rule, can repeat
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "instance_count must be between 1 and 10."
  }
}
```

| Setting | What it does |
|---|---|
| `type` | Restricts the value: `string`, `number`, `bool`, `list(string)`, `map(string)`, `object({...})`... (topic 17). Values are converted automatically when possible (`"3"` → `3`). |
| `default` | Used when nothing else sets it. **No default = required**: Terraform asks for it or errors. |
| `description` | For humans and documentation tools. Always write one. |
| `validation` | A `condition` that must be `true` and an `error_message`. Can refer to the variable itself and, since 1.9, to other variables and data. |
| `sensitive` | Hides the value in CLI output (`(sensitive value)`). **It's still stored in state and plan files.** |
| `nullable` | `false` = passing `null` is an error; with a default, `null` means "use the default". |
| `ephemeral` | (1.10+) The value is never stored in state or plan. Used for passwords and tokens; topic 51. |
| `deprecated` | (1.15+) A message shown if someone sets this variable, so you can retire it gently. Mostly for modules. |

Inside the configuration you read it as **`var.<name>`**. Variables are **read-only**: you can't change `var.x` in code. (Computed values go in `locals`, topic 16.)

Every variable in the **root** configuration is an input you supply. In modules (topic 43), variables are how the caller passes values in.

### The ways to set a value

| # | Where | Example | Loaded automatically? |
|---|---|---|---|
| 1 | `default` in the block | `default = "dev"` | Yes |
| 2 | Environment variable | `export TF_VAR_environment=staging` | Yes |
| 3 | `terraform.tfvars` / `terraform.tfvars.json` | `environment = "staging"` | Yes, if in the working directory |
| 4 | `*.auto.tfvars` / `*.auto.tfvars.json` | `common.auto.tfvars` | Yes, in **alphabetical** order |
| 5 | `-var-file=FILE` | `-var-file=prod.tfvars` | No, only when you pass it |
| 6 | `-var 'NAME=VALUE'` | `-var 'environment=prod'` | No, only when you pass it |
| 7 | Interactive prompt | `var.environment  Enter a value:` | Only if still unset and input is allowed |

A **`.tfvars` file** contains only `name = value` lines (no blocks):

```hcl
# prod.tfvars
environment    = "prod"
instance_count = 3
tags = {
  team = "platform"
}
```

### Precedence (who wins)

From **lowest** to **highest**. Later items override earlier ones:

```
default
  < TF_VAR_<name> environment variable
  < terraform.tfvars
  < terraform.tfvars.json
  < *.auto.tfvars / *.auto.tfvars.json   (alphabetical; later file wins)
  < -var and -var-file on the command line (in the ORDER you typed them; last wins)
```

The prompt only appears if a **required** variable is still unset after all of these.

---

## Why it exists

Without variables, you'd copy the whole configuration for dev, staging and prod, then fix the same bug three times. Variables let **one** codebase describe **many** environments, with only the values differing. Validation catches bad values (`"prdo"`) at plan time instead of after half the infrastructure is built. Types catch `"three"` where a number is needed.

---

## Analogy

**A coffee machine.**

- The **machine** is your code.
- **Variables** are the buttons and dials: size, strength, milk.
- The **default** is what you get if you press nothing.
- **Validation** is the machine refusing "size: 5 litres".
- **Precedence**: the factory setting (`default`) < the café's house settings (`terraform.tfvars`) < what you tap on the screen right now (`-var`). The most specific, most recent choice wins.
- **Sensitive** is a privacy screen on the display: the barista can't see your PIN, but the receipt (state) still records the payment.

---

## How it works inside

When does Terraform resolve variables? At the very start of `plan`/`apply`, before the graph walk:

```
1. collect values: defaults → TF_VAR_* → terraform.tfvars → *.auto.tfvars → -var/-var-file
2. still missing a required one?  input allowed → prompt;  -input=false → ERROR
3. convert each value to its declared type        ("3" → 3; [1,2] → list(string) ["1","2"])
   can't convert → "Invalid value for input variable"
4. null + nullable=false → error;  null + default → default
5. run validation blocks → any condition false → "Invalid value for variable" + your message
6. var.* values become graph nodes; everything that references them depends on them
```

Values passed with `-var` and `TF_VAR_` are strings on the command line; Terraform parses them as HCL for complex types:

```bash
terraform plan -var='zones=["a","b"]' -var='tags={team="platform"}'
export TF_VAR_zones='["a","b"]'
```

Root variable values **used in the plan** are saved inside a saved plan file, so `apply tfplan` uses exactly the same values (one more reason plan files are sensitive).

---

## Code / commands

`~/tf-lab/14-vars/versions.tf`: the usual `local` provider block (topic 10).

`variables.tf`:

```hcl
variable "environment" {
  type        = string
  description = "Which environment this is (dev, staging, prod)."
  # no default → required

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev, staging or prod."
  }
}

variable "port" {
  type        = number
  description = "Port the app listens on."
  default     = 8080

  validation {
    condition     = var.port >= 1024 && var.port <= 65535
    error_message = "port must be between 1024 and 65535 (no privileged ports)."
  }
}

variable "features" {
  type        = list(string)
  description = "Feature flags to enable."
  default     = []
}

variable "api_token" {
  type        = string
  description = "Token for the upstream API."
  sensitive   = true
  default     = "dev-token-not-secret"
}
```

`main.tf`:

```hcl
resource "local_file" "config" {
  filename = "${path.module}/${var.environment}.conf"
  content  = <<-EOT
    env=${var.environment}
    port=${var.port}
    features=${join(",", var.features)}
    token=${var.api_token}
  EOT
}
```

`join(",", list)` glues list items into one string (functions: topic 18).

### Missing required variable

```
$ terraform plan
var.environment
  Which environment this is (dev, staging, prod).

  Enter a value: dev
```

- Terraform prompts with your `description`. That's why descriptions matter.

In CI, with `-input=false`:

```
╷
│ Error: No value for required variable
│
│   on variables.tf line 1:
│    1: variable "environment" {
│
│ The root module input variable "environment" is not set, and has no default
│ value. Use a -var or -var-file command line argument to provide a value for
│ this variable.
╵
```

### Validation failure

```
$ terraform plan -var="environment=qa"
╷
│ Error: Invalid value for variable
│
│   on variables.tf line 1:
│    1: variable "environment" {
│     ├────────────────
│     │ var.environment is "qa"
│
│ environment must be dev, staging or prod.
│
│ This was checked by the validation rule at variables.tf:5,3-13.
╵
```

- `var.environment is "qa"`: the value it received.
- Then **your** error message. Write messages that say what's allowed.

### Sensitive in a plan

```
$ terraform plan -var="environment=dev"
  # local_file.config will be created
  + resource "local_file" "config" {
      + content              = (sensitive value)
      + filename             = "./dev.conf"
      ...
```

`content` includes `var.api_token`, so the **whole** `content` is marked sensitive. Sensitivity **spreads** to anything built from a sensitive value, just like "unknown" does.

### Precedence experiment

```bash
echo 'environment = "staging"' > terraform.tfvars
echo 'environment = "dev"'     > a.auto.tfvars
export TF_VAR_environment=prod
terraform plan -var-file=terraform.tfvars | grep filename
```

```
      + filename             = "./staging.conf"
```

Walk it: `TF_VAR` = prod → overridden by `terraform.tfvars` = staging → overridden by `a.auto.tfvars` = dev → overridden by `-var-file=terraform.tfvars` (command line, highest) = **staging**.

```bash
terraform plan -var-file=terraform.tfvars -var="environment=prod" | grep filename
```

```
      + filename             = "./prod.conf"
```

Command-line options are applied **in order**; the last one wins. Clean up: `unset TF_VAR_environment; rm a.auto.tfvars`.

### Type conversion error

```
$ terraform plan -var="environment=dev" -var="port=eighty"
╷
│ Error: Invalid value for input variable
│
│ The given value is not suitable for var.port declared at variables.tf:11,1-16:
│ a number is required.
╵
```

---

## Hands-on lab

1. Build the project above in `~/tf-lab/14-vars`. Run `terraform plan` with no values and answer the prompt.
2. Create `dev.tfvars` and `prod.tfvars` with different `environment`, `port` and `features`. Run `terraform apply -var-file=dev.tfvars`, then `terraform plan -var-file=prod.tfvars`. Read the plan: why is it `-/+`? (The `filename` changes, and `local_file` replaces on any change.)
3. Break validation three ways: `environment=qa`, `port=80`, `port=eighty`. Read each error.
4. Do the precedence experiment. Predict the winner **before** running each command.
5. Pass a list: `terraform plan -var-file=dev.tfvars -var='features=["beta","dark-mode"]'`.
6. Show that sensitive ≠ secret: after apply, `jq -r '.resources[0].instances[0].attributes.content' terraform.tfstate` shows the token in plain text.
7. Run `terraform plan -input=false` without values to see the CI error. Then `terraform destroy -var-file=dev.tfvars`.

---

## Where you'll use it

- **Per-environment values**: `dev.tfvars`, `prod.tfvars` (topic 49): sizes, counts, CIDR (Classless Inter-Domain Routing) ranges, domain names.
- **CI pipelines**: values from pipeline variables via `TF_VAR_*` or `-var-file` (topics 57–58).
- **Modules**: every module input is a variable (topic 44). Validation and good descriptions are the module's user interface.
- **Secrets**: `sensitive = true` (or `ephemeral = true`) on passwords, coming from a secret store via `TF_VAR_*` in CI, never from a committed `.tfvars` (topic 51).

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| CI job hangs | Waiting at the prompt for a required variable. | Use `-input=false` and pass all values. |
| `No value for required variable` | Not set anywhere. | Add it to a `.tfvars` file, `-var`, or `TF_VAR_`. |
| `Value for undeclared variable` (warning or error) | A `.tfvars` file sets a name with no `variable` block (typo, or a variable was deleted). | Fix the name or delete the line. With `-var`, it's an error. |
| A value you set "isn't used" | Something higher in the precedence list overrides it (a forgotten `*.auto.tfvars` or `TF_VAR_`). | `env | grep TF_VAR_`, `ls *.auto.tfvars`, check command-line order. |
| `Invalid value for input variable ... a number is required` | Wrong type. | Fix the value. Quote complex values in the shell: `-var='list=["a"]'`. |
| `Variables not allowed` | A variable used in a place that needs a literal, e.g. inside a `backend` block. | Use partial backend config (topic 35). (Since 1.15 module `source`/`version` may use variables.) |
| Secret shows in Git | Put in a committed `.tfvars` file. | Remove it, rotate it, and pass it from a secret store (topic 51). |

---

## Quick recap

- `variable` blocks declare inputs; read them as `var.<name>`. No default = required.
- Settings: `type`, `default`, `description`, `validation`, `sensitive`, `nullable` (+ `ephemeral`, `deprecated`).
- Precedence: default < `TF_VAR_` < `terraform.tfvars` < `*.auto.tfvars` < command line (last wins).
- `sensitive` hides output only; the value is still in state.
- Use `-input=false` in automation.

### Interview questions

1. **Q: What is the variable precedence order in Terraform?**
   A: From lowest: the default, `TF_VAR_` environment variables, `terraform.tfvars`, `terraform.tfvars.json`, `*.auto.tfvars` in alphabetical order, then `-var` and `-var-file` in the order given on the command line. The last one set wins.
2. **Q: Does `sensitive = true` protect a secret?**
   A: Only from being printed in CLI output. The value is still written to state and plan files in plain text. Protect those, or use ephemeral variables and write-only arguments so the value is never stored.
3. **Q: How do you stop someone passing an invalid value?**
   A: A precise `type` plus `validation` blocks with a condition and a clear error message. Validation runs at plan time, before any change.
