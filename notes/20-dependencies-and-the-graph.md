# 20 · Dependencies and the Graph

> **Level:** ★★ intermediate
>
> **You will learn:** how Terraform decides **order**: **implicit** dependencies (from references) and **explicit** ones (`depends_on`), when `depends_on` is truly needed and what it costs, the **dependency graph** as a DAG (Directed Acyclic Graph), how to view it with `terraform graph`, how **parallelism** and **destroy order** follow from it, and how to recognise and break **dependency cycles**.
>
> **You need:** topics 11 (references), 13 (graph walk), 19 (data sources).
>
> **Part 2 · Making configuration flexible** · Previous: [19 · Data sources](19-data-sources.md) · Next: [21 · `count`](21-count.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Terraform builds a **graph**: every resource, data source, variable, local, output and provider is a **node**; every reference is an **edge** ("B needs A"). It creates things in edge order, destroys them in **reverse** order, and runs everything without a path between them **in parallel** (10 at a time).

```hcl
resource "local_file" "a" { ... }
resource "local_file" "b" {
  content = local_file.a.content      # implicit dependency: a → b
}
resource "local_file" "c" {
  depends_on = [local_file.a]         # explicit dependency: a → c, no value used
  ...
}
```

**Picture it.** Cooking a meal: you can't ice a cake before baking it (dependency), but you can boil potatoes while it bakes (parallel). Washing up happens in reverse order.

**The one thing to know.** Prefer **references**. Use `depends_on` only for a dependency Terraform **can't see**, and add a comment saying why. A **cycle** (A needs B, B needs A) is an error you must design away.

**Try it now:** `terraform graph` in your topic 19 lab; find which nodes have no arrows between them.

**You've got it if you can:** give a real example where `depends_on` is necessary, and explain how to fix `Error: Cycle`.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Implicit dependencies

Any reference creates an edge. All of these do:

```hcl
content  = local_file.a.content                   # resource attribute
filename = "${local_file.a.filename}.bak"         # inside interpolation
content  = jsonencode({ x = random_pet.p.id })    # inside a function
content  = local.uses_a                           # through a local that references a
```

Terraform follows the chain through locals, variables in modules and outputs.

### Explicit dependencies: `depends_on`

```hcl
resource "aws_instance" "app" {
  # The app reads from this bucket at boot using the instance role.
  # Nothing here references the policy attachment, but without it
  # the first boot fails with AccessDenied.
  depends_on = [aws_iam_role_policy_attachment.app_s3_read]
  ...
}
```

- A **list of addresses**: resources, data sources or whole modules (`module.network`). No attributes, just the object.
- Available on resources, data sources, modules and outputs.
- It means "**don't start me until those are fully done** (created or updated)". It carries no value.

**When you actually need it** (a "hidden dependency"):

| Situation | Why Terraform can't see it |
|---|---|
| A VM's software uses permissions granted by a separate policy attachment | The VM references the **role**, not the attachment |
| An app (or an AWS Lambda function: AWS's run-code-without-servers service) needs a DNS record to exist before starting | The app reads the name from config, not from Terraform |
| A Kubernetes resource needs a CRD (Custom Resource Definition) installed by a Helm release | The manifest references the CRD by name string, not by Terraform address |
| A slow external system must be ready (`time_sleep` after creating something) | Time isn't a value |

**What it costs:**

- Less parallelism (things wait that might not need to).
- On **data sources** and **modules**, `depends_on` makes Terraform defer reads until apply whenever the dependency has **any** change pending, which can make values unknown and cause **unexpected replacements** (topic 19).
- It hides the real reason. Always add a comment.

### The graph

The graph is a **DAG = Directed Acyclic Graph**:

- **Directed**: edges have a direction (A → B: "B depends on A").
- **Acyclic**: no loops allowed.
- Nodes include resources, data sources, providers, variables, locals, outputs, module boundaries. Every resource also depends on its **provider** node (the provider must be configured first).

### Parallelism

During a walk, Terraform runs every node whose dependencies are finished, up to **10 at once** (`-parallelism=N`). Unrelated resources are created simultaneously. That's why a 50-resource apply can finish in the time of its longest chain, not the sum of all.

### Destroy order

Destroying goes **backwards**: if B depends on A, B is destroyed **first**. State stores each resource's `dependencies` (topic 12), so Terraform knows the order even after you've deleted the code.

Replacement order (topic 24):
- default `-/+`: destroy the old object, then create the new one, while dependents wait;
- `create_before_destroy`: create new, update dependents to point at it, then destroy old.

---

## Why it exists

Real infrastructure has strict order: a network before the server in it, a role before the policy attached to it, a database before the app that connects to it. Writing this order by hand (like a script) is error-prone and slow. Deriving it from references means:

- you **can't forget** an order your code depends on,
- independent work runs **in parallel**,
- deletion order is **automatically** the reverse.

---

## Analogy

**Cooking Sunday dinner.**

- The **recipe steps** form a graph: "make the dough" → "bake the pie" → "serve". "Peel potatoes" → "boil potatoes" → "mash".
- Pie and potatoes have **no link**, so two cooks work on them at once (parallelism).
- **Implicit dependency**: "spread the jam (made earlier) on the cake": the jam is *used*, so obviously it comes first.
- **Explicit dependency**: "preheat the oven before baking". The cake doesn't *contain* the oven's heat, but it still has to happen first. You write that down because it's not obvious from the ingredients.
- **Washing up** goes in reverse: you clear the plates before you wash the pan they were served from.
- **A cycle**: "make the sauce using the gravy" and "make the gravy using the sauce". Impossible; you have to rewrite the recipe.

---

## How it works inside

How core builds the graph (simplified):

```
1. Add a node for every resource/data/variable/local/output/module/provider in config
   AND for every resource in state that's no longer in config (orphans → delete nodes).
2. For each node, find every reference in its expressions → add an edge.
3. Add depends_on edges.
4. Connect each resource to its provider node; provider nodes to the variables they use.
5. Check for cycles → if any: "Error: Cycle: ..." and stop.
6. Transitive reduction: remove redundant edges (A→B→C makes A→C unnecessary).
7. Walk: a node starts when all its incoming edges are done; up to N at once.
```

For an apply, Terraform builds a graph of the **changes**, and for destroy actions it **reverses** the edges.

---

## Code / commands

`~/tf-lab/20-graph/` with `versions.tf` declaring `local`, `random` and `time` (`hashicorp/time ~> 0.13`).

```hcl
resource "random_pet" "name" {}

resource "local_file" "config" {                       # implicit: needs random_pet.name
  filename = "${path.module}/config.txt"
  content  = "name=${random_pet.name.id}\n"
}

resource "time_sleep" "warmup" {                       # pretend a service takes 10s to warm up
  create_duration = "10s"
  depends_on      = [local_file.config]                # explicit: start after config exists
}

resource "local_file" "ready" {
  filename = "${path.module}/ready.txt"
  content  = "ready at ${time_sleep.warmup.id}\n"     # implicit: needs time_sleep
}

resource "local_file" "independent" {                  # no edges: runs in parallel
  filename = "${path.module}/independent.txt"
  content  = "I don't wait for anyone\n"
}
```

### The apply order

```
$ terraform apply -auto-approve
random_pet.name: Creating...
local_file.independent: Creating...
random_pet.name: Creation complete after 0s [id=eager-moth]
local_file.independent: Creation complete after 0s [id=6f1a...]
local_file.config: Creating...
local_file.config: Creation complete after 0s [id=2b9c...]
time_sleep.warmup: Creating...
time_sleep.warmup: Still creating... [00m10s elapsed]
time_sleep.warmup: Creation complete after 10s [id=2026-11-19T11:02:03Z]
local_file.ready: Creating...
local_file.ready: Creation complete after 0s [id=d41e...]
```

- `random_pet.name` and `local_file.independent` start **together**: neither depends on anything.
- The chain `name → config → warmup → ready` runs strictly in order.

### View the graph

```bash
terraform graph
```

```
digraph G {
  rankdir = "RL";
  node [shape = rect, fontname = "sans-serif"];
  "local_file.config" [label="local_file.config"];
  "local_file.independent" [label="local_file.independent"];
  "local_file.ready" [label="local_file.ready"];
  "random_pet.name" [label="random_pet.name"];
  "time_sleep.warmup" [label="time_sleep.warmup"];
  "local_file.config" -> "random_pet.name";
  "local_file.ready" -> "time_sleep.warmup";
  "time_sleep.warmup" -> "local_file.config";
}
```

- Arrows point from a node **to what it depends on**.
- `local_file.independent` has no arrows: free to run any time.
- `terraform graph -type=plan` shows the full internal graph (providers, variables...): much bigger, useful for deep debugging.
- Draw it: `terraform graph | dot -Tsvg > graph.svg` (Graphviz), or `terraform graph -format=mermaid` (1.16+).

### Destroy order

```
$ terraform destroy -auto-approve
local_file.independent: Destroying... [id=6f1a...]
local_file.ready: Destroying... [id=d41e...]
local_file.independent: Destruction complete after 0s
local_file.ready: Destruction complete after 0s
time_sleep.warmup: Destroying... [id=2026-11-19T11:02:03Z]
time_sleep.warmup: Destruction complete after 0s
local_file.config: Destroying... [id=2b9c...]
local_file.config: Destruction complete after 0s
random_pet.name: Destroying... [id=eager-moth]
random_pet.name: Destruction complete after 0s
```

Exactly the reverse chain: `ready → warmup → config → name`.

### A cycle

```hcl
resource "local_file" "a" {
  filename = "${path.module}/a.txt"
  content  = "b says: ${local_file.b.content}"
}
resource "local_file" "b" {
  filename = "${path.module}/b.txt"
  content  = "a says: ${local_file.a.content}"
}
```

```
$ terraform plan
╷
│ Error: Cycle: local_file.b, local_file.a
╵
```

The message lists the nodes in the loop. With modules and locals, the list can be long; look for the two (or more) **resources** that reference each other.

**How to break cycles:**

| Pattern | Fix |
|---|---|
| Two resources need each other's IDs | Create both without the cross-reference, then link them with a **third** resource. AWS example: two security groups (AWS's virtual firewalls, topic 32) that allow each other → create both groups with no inline rules, then add `aws_vpc_security_group_ingress_rule` resources that reference both. |
| A resource references its own attribute | Compute the value in a local from inputs instead. |
| `depends_on` added on both sides "to be safe" | Remove the unnecessary one. |
| Provider configured from a resource that depends on that provider | Split into two configurations (topic 28). |

The AWS version, which you'll meet in topic 32:

```hcl
# ❌ cycle: each inline rule references the other group
resource "aws_security_group" "app" {
  ingress { security_groups = [aws_security_group.db.id] ... }
}
resource "aws_security_group" "db" {
  ingress { security_groups = [aws_security_group.app.id] ... }
}

# ✅ no cycle: groups first, rules as separate resources
resource "aws_security_group" "app" { name = "app" }
resource "aws_security_group" "db"  { name = "db" }

resource "aws_vpc_security_group_ingress_rule" "db_from_app" {
  security_group_id            = aws_security_group.db.id
  referenced_security_group_id = aws_security_group.app.id
  ip_protocol                  = "tcp"
  from_port                    = 5432
  to_port                      = 5432
}
```

---

## Hands-on lab

1. Build the project above. Apply and watch the order and the parallel start.
2. `terraform graph`, draw it by hand, then render it with Graphviz (`sudo apt install -y graphviz`) or Mermaid.
3. Run `terraform destroy` and check the order is reversed.
4. Apply again with `-parallelism=1`. Everything runs one by one. Compare with the default.
5. Remove the `depends_on` from `time_sleep.warmup` and apply after a destroy. Now `warmup` starts at the same time as `name`. Explain why the result is still correct here, and why it wouldn't be if a real service needed `config.txt` first.
6. Create the `a`/`b` cycle, read the error, and fix it by making `b` not reference `a`.
7. `terraform graph -type=plan | grep -c '\->'` vs `terraform graph | grep -c '\->'`: the full graph has many more edges (providers, variables...).
8. `terraform destroy`.

---

## Where you'll use it

- **Understanding plans and apply order** in every project.
- **IAM timing**: `depends_on` on the policy attachment before an instance or Lambda uses it.
- **Kubernetes**: CRDs before custom resources; namespaces before objects in them (references usually handle it).
- **Breaking cycles**: separate security group rule resources are the standard AWS pattern.
- **Performance tuning** (topic 63): the critical path (longest chain) sets the minimum apply time.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `Error: Cycle: x, y` | Two or more nodes depend on each other. | Remove one reference; use a third "link" resource; don't put `depends_on` on both sides. |
| Resource created too early → `AccessDenied`, `not found` in the API error, then works on re-apply | A **hidden dependency**. | Add a reference or `depends_on` (with a comment). |
| Plan shows many `(known after apply)` and replacements after adding `depends_on` | `depends_on` on a data source or module deferred reads. | Replace `depends_on` with real references, or only depend on the specific resource. |
| Apply is slow even though resources are unrelated | Unnecessary `depends_on`, or `-parallelism` too low. | Remove needless edges; check flags. |
| Destroy fails: `DependencyViolation` (AWS) | Something **outside** Terraform still uses the resource (e.g. a manually created ENI, Elastic Network Interface, i.e. a virtual network card, in the subnet), or Terraform's state lost the dependency. | Find and remove the outside user; re-apply to record dependencies. |
| `Error: Invalid depends_on reference` | `depends_on` must list objects, not attributes: `[aws_instance.web.id]` is wrong. | Use `[aws_instance.web]`. |

---

## Quick recap

- References create **implicit** dependencies; `depends_on` creates **explicit** ones for hidden needs.
- The graph is a DAG; nodes without a path between them run **in parallel** (10 by default).
- Destroy runs in **reverse** dependency order, using the dependencies stored in state.
- `depends_on` has costs (less parallelism, deferred reads); comment why you use it.
- Cycles are errors; break them with separate link resources or by removing a reference.

### Interview questions

1. **Q: Implicit vs explicit dependencies?**
   A: Implicit dependencies come from references: if B uses an attribute of A, A comes first. Explicit ones are declared with `depends_on` for dependencies Terraform can't infer, like a VM needing an IAM policy attachment it doesn't reference.
2. **Q: How does Terraform know the order to destroy resources you deleted from the code?**
   A: Each resource's dependencies are recorded in state. Terraform builds the destroy graph from that and destroys dependents first.
3. **Q: How do you fix a dependency cycle between two security groups?**
   A: Create the groups without inline rules that reference each other, then define the cross-group rules as separate rule resources that reference both group IDs.
