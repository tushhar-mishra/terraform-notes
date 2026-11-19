# 18 · Built-in Functions and `terraform console`

> **Level:** ★★ intermediate
>
> **You will learn:** how to test any expression interactively with **`terraform console`**, and the ~35 built-in functions you'll actually use, grouped by job: strings, collections, maps, types and errors (`try`, `can`), encoding (`jsonencode`, `yamldecode`, `base64encode`), files and templates (`file`, `templatefile`), networking (`cidrsubnet`, `cidrhost`), time and hashing. Plus **provider-defined functions** (`provider::...`), and the rule that you **can't write your own** functions in HCL.
>
> **You need:** topics 14–17.
>
> **Part 2 · Making configuration flexible** · Previous: [17 · Types and expressions](17-types-and-expressions.md) · Next: [19 · Data sources](19-data-sources.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Terraform has a fixed library of **built-in functions**: `name(arg1, arg2)`. You can't define new ones. You'll use about thirty regularly. Test them in **`terraform console`** before putting them in code.

```bash
terraform console
> upper("web")
"WEB"
> join("-", ["shop", "dev", "api"])
"shop-dev-api"
> merge({a = 1}, {b = 2})
{ "a" = 1, "b" = 2 }
> cidrsubnet("10.0.0.0/16", 8, 1)
"10.0.1.0/24"
> try(var.missing.thing, "fallback")
"fallback"
```

**Picture it.** A kitchen drawer of gadgets: peeler, grater, zester. You don't build your own; you learn which gadget does what, and you test it on one carrot first (the console).

**The one thing to know.** When an expression doesn't do what you expect, **don't guess in a plan**: copy it into `terraform console` and look at the actual value and its `type()`.

**Try it now:** open `terraform console` and run every line above.

**You've got it if you can:** split `10.0.0.0/16` into four `/18` subnets with `cidrsubnet`, and build a JSON policy string with `jsonencode`.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### `terraform console`

An interactive prompt (a **REPL**: Read-Eval-Print Loop) that evaluates Terraform expressions using:

- your configuration's **variables** (from `.tfvars`, `-var`...),
- **locals**,
- **state** (resource attributes, outputs, data sources already read).

It **never changes** anything. But it **holds the state lock for as long as it's open**, so exit it (`exit` or Ctrl-D) before running `plan`/`apply` in the same folder.

```bash
terraform console                           # in a project folder
terraform console -var-file=prod.tfvars     # with specific variable values
echo 'local.name_prefix' | terraform console    # non-interactive, for scripts
```

### The functions you'll actually use

**Strings**

| Function | Example | Result |
|---|---|---|
| `lower`, `upper`, `title` | `lower("Web")` | `"web"` |
| `trimspace` | `trimspace("  x \n")` | `"x"` |
| `replace(s, find, repl)` | `replace("a_b_c", "_", "-")` | `"a-b-c"` (find can be `/regex/`) |
| `split(sep, s)` | `split(",", "a,b")` | `["a", "b"]` |
| `join(sep, list)` | `join(",", ["a","b"])` | `"a,b"` |
| `format(fmt, ...)` | `format("%s-%03d", "web", 7)` | `"web-007"` |
| `substr(s, offset, len)` | `substr("production", 0, 4)` | `"prod"` |
| `startswith`, `endswith`, `strcontains` | `startswith("prod-db", "prod")` | `true` |
| `regex`, `regexall` | `regex("[0-9]+", "v12")` | `"12"` |

**Collections**

| Function | Example | Result |
|---|---|---|
| `length` | `length(["a","b"])` | `2` |
| `concat` | `concat(["a"], ["b","c"])` | `["a","b","c"]` |
| `contains` | `contains(["dev","prod"], "qa")` | `false` |
| `distinct` | `distinct(["a","a","b"])` | `["a","b"]` |
| `flatten` | `flatten([["a"], ["b", ["c"]]])` | `["a","b","c"]` |
| `element(list, i)` | `element(["a","b"], 3)` | `"b"` (wraps around: 3 mod 2 = 1) |
| `index(list, value)` | `index(["a","b"], "b")` | `1` |
| `slice(list, start, end)` | `slice(["a","b","c"], 0, 2)` | `["a","b"]` |
| `sort`, `reverse` | `sort(["b","a"])` | `["a","b"]` |
| `range` | `range(3)` | `[0, 1, 2]` |
| `one(list)` | `one(["x"])` / `one([])` | `"x"` / `null` (error if 2+) |
| `toset`, `tolist`, `tomap`, `tostring`, `tonumber`, `tobool` | `toset(["a","a"])` | Type conversion |
| `setunion`, `setintersection`, `setsubtract` | `setsubtract(["a","b"], ["a"])` | `["b"]` |
| `sum`, `min`, `max` | `max(1, 5, 3)` / `max([1,5]...)` | `5` (`...` expands a list into arguments) |

**Maps**

| Function | Example | Result |
|---|---|---|
| `merge` | `merge({a=1}, {a=2, b=3})` | `{a=2, b=3}` (later wins) |
| `lookup(map, key, default)` | `lookup({a=1}, "b", 0)` | `0` |
| `keys`, `values` | `keys({b=1, a=2})` | `["a","b"]` (sorted) |
| `zipmap(keys, values)` | `zipmap(["a","b"], [1,2])` | `{a=1, b=2}` |

**Types and errors**

| Function | Use |
|---|---|
| `try(expr1, expr2, ...)` | Returns the first expression that **doesn't error**. `try(var.cfg.port, 80)` |
| `can(expr)` | `true` if the expression evaluates without error. Mostly in `validation`: `can(regex("^[a-z]+$", var.name))` |
| `coalesce(a, b, ...)` | First argument that isn't `null` or `""`. |
| `nonsensitive(x)` / `sensitive(x)` | Remove / add the sensitive mark. Use `nonsensitive` only when you're sure. |
| `type(x)` | Console only: shows the type. |

**Encoding**

| Function | Use |
|---|---|
| `jsonencode(value)` / `jsondecode(string)` | HCL ↔ JSON. **Always** build JSON (IAM policies, configs) with `jsonencode`, not by hand in a heredoc: it handles quotes and commas for you. |
| `yamlencode` / `yamldecode` | HCL ↔ YAML (e.g. read a YAML config file, write a Kubernetes manifest). |
| `base64encode` / `base64decode` | e.g. some `user_data` fields want base64. |
| `urlencode`, `csvdecode` | Less common. |

**Files and templates**

| Function | Use |
|---|---|
| `file(path)` | Reads a file's text. |
| `fileexists(path)` | `true`/`false`. |
| `templatefile(path, vars)` | Reads a template file and renders `${}` / `%{}` with the given variables. The standard way to build scripts and config files. |
| `fileset(dir, pattern)` | Set of file names matching a glob, e.g. `fileset(path.module, "configs/*.yaml")`. |
| `abspath`, `dirname`, `basename`, `pathexpand` | Path helpers. |

Paths are relative to where you run `terraform`. Use `path.module` (this module's folder), `path.root` (the root module's folder) or `path.cwd` (current directory) to be explicit.

**Networking**

| Function | Example | Result |
|---|---|---|
| `cidrsubnet(prefix, newbits, num)` | `cidrsubnet("10.0.0.0/16", 8, 2)` | `"10.0.2.0/24"` |
| `cidrsubnets(prefix, newbits...)` | `cidrsubnets("10.0.0.0/16", 8, 8, 4)` | `["10.0.0.0/24","10.0.1.0/24","10.0.16.0/20"]` |
| `cidrhost(prefix, n)` | `cidrhost("10.0.1.0/24", 10)` | `"10.0.1.10"` |
| `cidrnetmask` | `cidrnetmask("10.0.0.0/16")` | `"255.255.0.0"` |

`cidrsubnet` recap from [Linux topic 059](../../linux-mastery-rhel/notes/059-ip-addressing-and-subnetting.md): `newbits` = how many bits to add to the prefix (`/16` + 8 = `/24`), `num` = which of the resulting subnets (0, 1, 2...).

**Time and hashing**

| Function | Note |
|---|---|
| `timestamp()` | ⚠️ Current time, **different on every run**: anything using it changes on every plan. Prefer the `time_static` resource from the `time` provider when you need a fixed timestamp. |
| `formatdate`, `timeadd`, `timecmp` | Formatting and maths on timestamps. |
| `md5`, `sha256`, `filesha256`, `uuid` | ⚠️ `uuid()` is also different every run. |

### Provider-defined functions (Terraform 1.8+)

Providers can ship their own functions, called with a namespace:

```hcl
provider::aws::arn_parse("arn:aws:iam::123456789012:role/admin").account_id   # "123456789012"
```

They only work if that provider is in `required_providers`.

### No user-defined functions

HCL has no `function` keyword. To reuse logic, you use **locals** (a value) or **modules** (a group of resources, topic 43). This is deliberate: it keeps configurations readable.

---

## Why it exists

Real inputs are messy: comma-separated strings, YAML files, nested lists, CIDR ranges to slice up. Functions let you reshape data **declaratively** without leaving HCL or calling external scripts. The console exists because guessing what `flatten(...)` returns by running a plan every time is slow and error-prone.

---

## Analogy

**A kitchen drawer of gadgets.** You can't invent new gadgets in this kitchen. You learn the drawer: the grater (`split`), the mixer (`merge`), the sieve (`distinct`), the scale (`length`), the label maker (`format`). Before using a new gadget on the whole meal, you try it on one carrot on the side (`terraform console`).

---

## How it works inside

- Functions are **pure** (same input → same output) with a few exceptions: `timestamp()`, `uuid()`, `bcrypt()` and file reads. Impure functions make plans change every time.
- Functions run during **plan** (and again at apply when inputs were unknown). If an argument is `(known after apply)`, the result is too.
- File functions (`file`, `templatefile`) read files **at plan time** from your machine/runner, not from the target server. They can't read files created by a resource in the same run.
- Errors inside functions point at the exact call: `Invalid function argument ... Invalid value for "num" parameter`.

---

## Code / commands

### A realistic lab: generating config from a template and a YAML file

`~/tf-lab/18-functions/` with `versions.tf` (`local` provider).

`services.yaml`, a file another team maintains:

```yaml
services:
  - name: api
    port: 8080
    public: true
  - name: worker
    port: 9090
    public: false
  - name: Admin UI
    port: 3000
    public: true
```

`templates/nginx.conf.tftpl`:

```
# generated by Terraform for ${environment}
%{ for s in services ~}
upstream ${s.id} { server 127.0.0.1:${s.port}; }
%{ endfor ~}
```

`main.tf`:

```hcl
variable "environment" {
  type    = string
  default = "dev"
}

locals {
  raw = yamldecode(file("${path.module}/services.yaml"))

  # Clean names into IDs: "Admin UI" → "admin-ui"
  services = [
    for s in local.raw.services : {
      id     = replace(lower(trimspace(s.name)), " ", "-")
      port   = s.port
      public = s.public
    }
  ]

  public_ids = [for s in local.services : s.id if s.public]

  # One /24 subnet per service from 10.20.0.0/16
  subnets = zipmap(
    local.services[*].id,
    [for i, s in local.services : cidrsubnet("10.20.0.0/16", 8, i)]
  )
}

resource "local_file" "nginx" {
  filename = "${path.module}/out/nginx.conf"
  content = templatefile("${path.module}/templates/nginx.conf.tftpl", {
    environment = var.environment
    services    = local.services
  })
}

resource "local_file" "inventory" {
  filename = "${path.module}/out/services.json"
  content = jsonencode({
    environment = upper(var.environment)
    public      = local.public_ids
    subnets     = local.subnets
    generated   = formatdate("YYYY-MM-DD", plantimestamp())
  })
}
```

- `yamldecode(file(...))`: read and parse YAML into an HCL object.
- `[for s in ... : {...}]`: a **`for` expression**: build a new list from an old one. Topic 23 covers these fully; here, read it as "for each service, make this object".
- `[for ... : s.id if s.public]`: the same, with a **filter**.
- `[for i, s in ...]`: `i` is the index (0, 1, 2).
- `templatefile(path, {vars})`: the template sees `environment` and `services` as variables.
- `plantimestamp()` (1.5+): the time the **plan** was made. Unlike `timestamp()`, it's known during plan, but it still changes every run, so here the file content changes daily. That's acceptable for a demo; avoid it in real resources.

### Explore it in the console first

```bash
terraform init
terraform console
```

```
> local.services
[
  {
    "id" = "api"
    "port" = 8080
    "public" = true
  },
  {
    "id" = "worker"
    "port" = 9090
    "public" = false
  },
  {
    "id" = "admin-ui"
    "port" = 3000
    "public" = true
  },
]
> local.public_ids
[
  "api",
  "admin-ui",
]
> local.subnets
{
  "admin-ui" = "10.20.2.0/24"
  "api" = "10.20.0.0/24"
  "worker" = "10.20.1.0/24"
}
> cidrhost(local.subnets["api"], 10)
"10.20.0.10"
> exit
```

Now apply and check:

```bash
terraform apply -auto-approve
cat out/nginx.conf; jq . out/services.json
```

```
# generated by Terraform for dev
upstream api { server 127.0.0.1:8080; }
upstream worker { server 127.0.0.1:9090; }
upstream admin-ui { server 127.0.0.1:3000; }
```

```json
{
  "environment": "DEV",
  "generated": "2026-11-19",
  "public": ["api", "admin-ui"],
  "subnets": {
    "admin-ui": "10.20.2.0/24",
    "api": "10.20.0.0/24",
    "worker": "10.20.1.0/24"
  }
}
```

### `try` and `can` in practice

```hcl
locals {
  # Use the YAML's optional "timeout" field if present, else 30
  timeouts = { for s in local.raw.services : s.name => try(s.timeout, 30) }
}

variable "bucket_name" {
  type = string
  validation {
    condition     = can(regex("^[a-z0-9-]{3,63}$", var.bucket_name))
    error_message = "Bucket names: 3-63 chars, lowercase letters, digits and hyphens."
  }
}
```

### A function error

```
> cidrsubnet("10.0.0.0/16", 20, 1)
╷
│ Error: Error in function call
│
│   on <console-input> line 1:
│   (source code not available)
│
│ Call to function "cidrsubnet" failed: insufficient address space to extend
│ prefix of 16 by 20.
╵
```

`/16` + 20 bits = `/36`, more than the 32 bits an IPv4 address has.

---

## Hands-on lab

1. Build the project above. Explore every local in `terraform console` **before** applying.
2. Add a fourth service to `services.yaml` with `timeout: 60`. Plan: what changes in each file?
3. In the console, answer with one expression each:
   - the total of all ports (`sum(local.services[*].port)`),
   - the IDs sorted in reverse (`reverse(sort(local.services[*].id))`),
   - four `/18` subnets from `10.0.0.0/16` (`cidrsubnets("10.0.0.0/16", 2, 2, 2, 2)`),
   - the first non-empty of `""`, `null`, `"x"` (`coalesce("", "x")`).
4. Replace `jsonencode` with a hand-written JSON heredoc and add a service name containing a `"` character. See the broken JSON. Switch back. (That's why `jsonencode` matters.)
5. Add `formatdate("hh:mm:ss", timestamp())` to `services.json`, run `plan` twice, and see it never settle. Remove it.
6. `terraform destroy`.

---

## Where you'll use it

- **`cidrsubnet`/`cidrsubnets`**: carving VPC subnets (topic 32).
- **`jsonencode`**: IAM policies, ECS (Elastic Container Service) task definitions, any JSON argument.
- **`templatefile`**: EC2 `user_data`, cloud-init, config files (topic 33).
- **`merge`**: tags (`merge(local.common_tags, { Name = "web" })`).
- **`try`/`can`/`lookup`**: optional fields in inputs; validation rules.
- **`yamldecode`/`fileset`**: data-driven configs, e.g. "one GitHub repo per YAML file".

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `Call to unknown function` | Typo, or a function from a newer Terraform version, or a provider function without the provider. | Check the name and your version; add the provider for `provider::` functions. |
| `Invalid function argument` | Wrong type or value. | Check in the console with `type(...)`. |
| `no file exists at "templates/x.tftpl"` | Path relative to the wrong folder. | Use `"${path.module}/templates/x.tftpl"`. |
| Plan shows a change **every run** | `timestamp()`, `uuid()` or another impure function in a resource argument. | Use `time_static`, `random_uuid`, or `lifecycle.ignore_changes` (topic 24). |
| `Invalid template interpolation value` | A list or map put into `${}` in a template. | Use `join()`, a `%{ for }` loop, or `jsonencode()`. |
| `templatefile` complains a variable is missing | The template uses a name not passed in the map. | Pass every variable the template uses. |
| Console hangs or errors with a lock | Another command holds the state lock. | Close other `terraform` commands; exit the console before applying. |

---

## Quick recap

- `terraform console` evaluates expressions against your variables, locals and state, read-only. Test there first.
- Learn the drawer: strings (`join`, `split`, `replace`, `format`), collections (`length`, `concat`, `flatten`, `distinct`, `toset`), maps (`merge`, `lookup`, `zipmap`), errors (`try`, `can`, `coalesce`), encoding (`jsonencode`, `yamldecode`), files (`file`, `templatefile`), networking (`cidrsubnet`, `cidrhost`).
- Avoid impure functions (`timestamp`, `uuid`) in resource arguments.
- No custom functions in HCL; providers can ship `provider::name::fn()` functions.

### Interview questions

1. **Q: How do you test an expression without running a plan?**
   A: `terraform console`, which evaluates expressions using the configuration's variables, locals and current state, without changing anything.
2. **Q: Why use `jsonencode` instead of writing JSON in a heredoc?**
   A: It produces valid JSON from HCL values every time (escaping, commas, types), lets you use variables and expressions naturally, and Terraform shows the diff structurally in plans.
3. **Q: What does `cidrsubnet("10.0.0.0/16", 8, 3)` return?**
   A: `10.0.3.0/24`: extend the /16 by 8 bits to /24 and take subnet number 3.
