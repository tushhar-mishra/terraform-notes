# 17 · Types and Expressions

> **Level:** ★ foundation
>
> **You will learn:** Terraform's **type system**: primitive types (`string`, `number`, `bool`), collection types (`list`, `set`, `map`), structural types (`object`, `tuple`), `any`, `null`, and **`optional()`** object attributes with defaults; how **automatic type conversion** works; and the **expressions** you'll write every day: operators, the **conditional** `? :`, indexing and attribute access, **splat** `[*]`, string **templates** with `%{ if }` / `%{ for }` directives, and heredocs.
>
> **You need:** topics 14–16.
>
> **Part 2 · Making configuration flexible** · Previous: [16 · Locals](16-locals.md) · Next: [18 · Built-in functions and `terraform console`](18-functions-and-console.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Every value has a **type**. Declaring types on variables catches mistakes early and documents what's expected.

```hcl
variable "app" {
  type = object({
    name     = string
    replicas = optional(number, 1)          # may be omitted → 1
    ports    = list(number)
    labels   = optional(map(string), {})
  })
}
```

Everyday expressions:

```hcl
local.is_prod ? 3 : 1                       # conditional
var.app.ports[0]                            # index a list
var.app.labels["team"]                      # look up a map key
var.servers[*].name                         # splat: every element's name
"Hello, ${var.app.name}!"                   # interpolation
"%{ if local.is_prod }PROD%{ else }dev%{ endif }"   # template directive
```

**Picture it.** Types are the **shapes of sockets**: a round plug (number) won't fit a square socket (string list). An `object` type is a **form with named fields**.

**The one thing to know.** `list` = ordered, may repeat. `set` = unordered, unique. `map` = keys → values of **one** type. `object` = named fields of **different** types.

**Try it now:** open `terraform console` in any lab folder and type `[1, 2, 2]`, `toset([1, 2, 2])`, `{a = 1, b = "x"}`, `true ? "yes" : "no"`.

**You've got it if you can:** pick the right type for "a list of subnet CIDRs", "a set of usernames", "tags", and "one server's settings".

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Primitive types

| Type | Examples | Notes |
|---|---|---|
| `string` | `"web"`, `"10.0.0.0/16"` | Unicode text. Always double quotes. |
| `number` | `3`, `2.5`, `-1` | One numeric type for integers and decimals. |
| `bool` | `true`, `false` | No quotes. |

### `null`

`null` = "no value". Setting an argument to `null` is the same as **not setting it**: the provider's default (or the variable's default) is used. That makes `null` the tool for "set this only sometimes":

```hcl
file_permission = local.is_prod ? "0600" : null    # prod: 0600; otherwise provider default
```

### Collection types: all elements share one type

| Type | Ordered? | Duplicates? | Access | Literal |
|---|---|---|---|---|
| `list(T)` | Yes | Allowed | `x[0]` | `["a", "b"]` |
| `set(T)` | **No** | **No** (removed) | Can't index; iterate or convert | `toset(["a", "b"])` |
| `map(T)` | By key (shown sorted) | Keys unique | `x["key"]` or `x.key` | `{ a = 1, b = 2 }` |

`T` is the element type: `list(string)`, `map(number)`, `set(string)`, even `list(map(string))`.

### Structural types: each element can have its own type

| Type | Like | Literal |
|---|---|---|
| `object({ name = string, port = number })` | A form with named fields | `{ name = "web", port = 80 }` |
| `tuple([string, number, bool])` | A fixed-length list where each position has its own type | `["web", 80, true]` |

Important: the literal `{ ... }` and `[ ... ]` you write are **objects and tuples**. Terraform converts them to `map`/`list` automatically when a map or list is expected. That's why the console shows `tolist([])` or `tomap({...})`: it's telling you which type a value ended up as.

### `optional()` attributes (Terraform 1.3+)

Without `optional`, every attribute of an object type is required. With it:

```hcl
variable "app" {
  type = object({
    name     = string                         # required
    replicas = optional(number, 1)            # optional, default 1
    image    = optional(string)               # optional, default null
  })
}
```

This is how module authors (topic 44) design friendly inputs.

### `any`

`type = any` accepts any value. Use it sparingly: you lose the error checking that types give you.

### Expressions you'll use

| Expression | Example | Result |
|---|---|---|
| Arithmetic | `var.port + 1`, `10 % 3` | `8081`, `1` |
| Comparison | `var.count >= 2`, `var.env == "prod"`, `!=` | `true`/`false` |
| Logic | `a && b`, `a \|\| b`, `!a` | Since 1.12, `&&` and `\|\|` **short-circuit** (the right side isn't evaluated if the left decides) |
| Conditional | `cond ? a : b` | `a` and `b` should have the same type |
| Index | `var.list[0]`, `var.map["k"]` | Error if out of range or key missing |
| Attribute | `var.obj.name`, `local_file.x.id` | |
| Splat | `var.servers[*].name` | List of every element's `name` |
| Parentheses | `(a + b) * 2` | Grouping |

### Strings: templates and directives

```hcl
"Hello, ${var.name}"                              # interpolation
"%{ if var.debug }debug=on%{ else }debug=off%{ endif }"
<<-EOT
  %{ for p in var.ports ~}
  listen ${p}
  %{ endfor ~}
EOT
```

- `%{ if }...%{ else }...%{ endif }`: choose text.
- `%{ for x in list }...%{ endfor }`: repeat text.
- `~` next to a `%{ }` or `${ }` **strips the whitespace/newline** on that side. Without it, you get blank lines.
- `<<-EOT` (indented heredoc) removes the common leading indentation; `<<EOT` keeps it.

For anything longer than a few lines, put the template in a file and use `templatefile()` (topic 18).

---

## Why it exists

- **Types catch errors at plan time.** A typo like `replicas = "three"` fails immediately with a clear message, instead of producing a broken resource.
- **Types document intent.** `list(object({ name = string, cidr = string }))` tells a reader exactly what to pass.
- **Expressions** let one configuration adapt (dev vs prod) without copy-pasting.

---

## Analogy

**Containers in a kitchen.**

- `string`/`number`/`bool`: single items: a label, a weight, a yes/no tick.
- `list`: an **egg carton**: ordered slots, all eggs, duplicates fine.
- `set`: a **bag of unique stickers**: no order, no duplicates.
- `map`: a **spice rack with labels**: look up by name, every jar is a spice.
- `object`: a **lunchbox with compartments**: "sandwich", "fruit", "drink", each a different kind of thing.
- `optional(x, default)`: "if you don't pack a drink, I'll add water".
- `null`: an empty compartment: "use the default".

---

## How it works inside

**Type conversion** happens automatically when a value goes into a place with a declared type:

```
"5"           → number        5           (string that looks like a number)
5             → string        "5"
"true"        → bool          true
["a","b","a"] → set(string)   ["a","b"]   (duplicates dropped, order lost)
{a=1,b=2}     → map(number)   (object → map: all values must convert to one type)
{a=1,b="x"}   → map(string)   {a="1", b="x"}  (1 converted to "1")
```

If conversion is impossible (`"three"` → number), you get `Invalid value for input variable ... a number is required`.

**Unknowns and types:** during plan, a value can be **known type but unknown value** (`(known after apply)`), which is fine for most arguments, but not for things Terraform must know at plan time, like `count` and `for_each` keys (topics 21–22).

**Sets are sorted, not ordered:** when Terraform shows a set it sorts it, but you must never rely on any position, so you can't write `myset[0]`.

---

## Code / commands

All of these work in `terraform console` in an empty folder (no `init` needed for plain expressions):

```bash
mkdir -p ~/tf-lab/17-types && cd ~/tf-lab/17-types && terraform console
```

```
> [1, 2, 2]
[
  1,
  2,
  2,
]
> toset([1, 2, 2])
toset([
  1,
  2,
])
> { name = "web", port = 80 }
{
  "name" = "web"
  "port" = 80
}
> tomap({ a = 1, b = "x" })
tomap({
  "a" = "1"
  "b" = "x"
})
> 5 > 3 && "a" == "a"
true
> null == null
true
> [{ name = "a", ip = "10.0.0.1" }, { name = "b", ip = "10.0.0.2" }][*].ip
[
  "10.0.0.1",
  "10.0.0.2",
]
> "%{ if true }on%{ else }off%{ endif }"
"on"
> type(toset(["a"]))
set(string)
> exit
```

- `tomap({ a = 1, b = "x" })`: the number `1` became the string `"1"` so all values share one type.
- `[*].ip`: splat pulls `ip` out of each object.
- `type(x)` (console only) shows a value's type: very useful when debugging.

### A typed variable in a project

`~/tf-lab/17-types/` with `versions.tf` (`local` provider), then `variables.tf`:

```hcl
variable "app" {
  description = "The application to configure."
  type = object({
    name     = string
    replicas = optional(number, 1)
    ports    = list(number)
    labels   = optional(map(string), {})
    debug    = optional(bool, false)
  })
}
```

`main.tf`:

```hcl
resource "local_file" "nginx" {
  filename = "${path.module}/${var.app.name}.conf"
  content  = <<-EOT
    # ${var.app.name}: ${var.app.replicas} replica(s)
    %{ for p in var.app.ports ~}
    listen ${p};
    %{ endfor ~}
    %{ if var.app.debug ~}
    error_log /dev/stderr debug;
    %{ endif ~}
    %{ for k, v in var.app.labels ~}
    # label ${k}=${v}
    %{ endfor ~}
  EOT
}
```

`terraform.tfvars`:

```hcl
app = {
  name  = "shop"
  ports = [80, 443]
  labels = {
    team = "web"
  }
}
```

```bash
terraform init && terraform apply -auto-approve && cat shop.conf
```

```
# shop: 1 replica(s)
listen 80;
listen 443;
# label team=web
```

- `replicas` wasn't given → `optional(number, 1)` filled in `1`.
- `debug` defaulted to `false` → the `%{ if }` block produced nothing.
- `~` removed the blank lines the directives would otherwise leave.

### Type errors you'll see

`app = { name = "shop", ports = ["eighty"] }`:

```
│ Error: Invalid value for input variable
│
│   on terraform.tfvars line 1:
│    1: app = {
│
│ The given value is not suitable for var.app declared at variables.tf:1,1-15:
│ attribute "ports": element 0: a number is required.
```

`app = { ports = [80] }` (missing `name`):

```
│ The given value is not suitable for var.app declared at variables.tf:1,1-15:
│ attribute "name" is required.
```

The messages point at the exact attribute and element.

---

## Hands-on lab

1. Run every console example above. Then predict and check: `length(toset(["a","a","b"]))`, `["a","b"][2]`, `{a=1}["b"]`.
2. Build the typed project. Apply with the given `terraform.tfvars`.
3. Change `terraform.tfvars`: add `debug = true`, `replicas = 3`, a second label. Plan and read the content diff.
4. Remove the `~` characters from the template and apply. Look at the blank lines in `shop.conf`. Put them back.
5. Break types on purpose: `ports = "80"`, `replicas = "many"`, a missing `name`. Read each error.
6. Use `null`: add `file_permission = var.app.debug ? "0644" : null` to the resource, and check the plan for both values of `debug`.
7. `terraform destroy`.

---

## Where you'll use it

- **Module inputs** (topic 44): `object({...})` with `optional()` is the standard for clean module interfaces.
- **Conditional settings**: `is_prod ? ... : ...` and `null` to skip optional arguments.
- **Splat**: collecting IDs, e.g. `aws_subnet.private[*].id` (topic 21).
- **Templates**: `user_data` scripts, config files, IAM policies (but use `jsonencode` for JSON, topic 18).

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `Inconsistent conditional result types` | `cond ? a : b` where `a` and `b` have different types (e.g. a list vs a string). | Make both sides the same type: `["x"] : []`, or wrap with `tolist()`/`tostring()`. |
| `Invalid index` / `The given key does not identify an element in this collection value` | Index out of range or map key missing. | Check `length()`, or use `lookup(map, key, default)` / `try()` (topic 18). |
| `Elements of a set are identified only by their value and don't have any separate index or key` | Tried `myset[0]`. | Convert: `tolist(myset)[0]`, or iterate. |
| `attribute "x" is required` | Object type without `optional()`. | Pass it, or make it `optional(...)`. |
| `Unsupported attribute` on an object | Field name typo, or the field isn't in the type (extra attributes are dropped silently on conversion!). | Check spelling against the type. |
| Unexpected blank lines in rendered files | Template directives leave newlines. | Add `~` to strip whitespace. |

---

## Quick recap

- Primitives: `string`, `number`, `bool`; plus `null` = "not set / use default".
- Collections (one element type): `list` (ordered), `set` (unique, unordered), `map` (keyed).
- Structural: `object` (named fields of different types), `tuple`. `optional(type, default)` makes object fields optional.
- Terraform converts types automatically where it can; declare precise types to catch mistakes.
- Expressions: operators, `? :`, indexing, splat `[*]`, `${}` interpolation, `%{ if/for }` directives, `~` strips whitespace.

### Interview questions

1. **Q: What's the difference between a list, a set and a map?**
   A: A list is ordered and allows duplicates, accessed by index. A set is unordered with unique values and can't be indexed. A map has unique string keys pointing to values of one type.
2. **Q: Map vs object?**
   A: A map's values all share one type and keys are arbitrary. An object has a fixed set of named attributes, each with its own type. Object types are used to describe structured inputs.
3. **Q: How do you make an argument conditional?**
   A: Use a conditional expression that returns `null` when it shouldn't be set, e.g. `kms_key_id = var.encrypt ? var.key_id : null`. A null argument is treated as unset.
