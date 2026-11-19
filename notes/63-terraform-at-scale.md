# 63 · Terraform at Scale

> **Level:** ★★★ advanced
>
> **You will learn:** why Terraform gets **slow** and **fragile** as estates grow (thousands of resources, hundreds of states, many teams), how to **measure** where time goes, the levers (**splitting state**, **refresh** options, **`-parallelism`**, provider **retries** and API **rate limits**, plugin caching in CI), why ⚠️ **`-target`** is an emergency tool only, and the organisational patterns that keep large Terraform estates healthy: ownership, a module catalogue, orchestration between states, and automation.
>
> **You need:** topics 13 (graph walk, parallelism), 34 (refresh), 38, 47, 49, 50, 57.
>
> **Part 10 · Running it in production** · Previous: [62 · Upgrades: Terraform, providers and modules](62-upgrades.md) · Next: [64 · Debugging Terraform](64-debugging-terraform.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A plan's time is roughly: **init** (download providers/modules) + **refresh** (one or more API reads **per resource**) + **graph walk** (10 at a time). A 3,000-resource state can mean thousands of API calls on every plan, slow PRs, API throttling, and a huge blast radius.

| Problem | Lever |
|---|---|
| Plans take 10+ minutes | **Split the state** (by component/lifecycle/team) |
| `ThrottlingException` / `Rate exceeded` | Lower `-parallelism`; provider retries; split states |
| Slow `init` in CI | Plugin cache / cached `.terraform` keyed by the lock file |
| Need a quick plan of one tiny thing | ⚠️ `-target` only in emergencies; otherwise smaller states |
| Trusting refresh to be skipped | `-refresh=false` only when you know nothing drifted |

**Picture it.** A single warehouse inventory for a whole country: counting every shelf before every change takes all day. Regional warehouses, each with its own inventory, can each be counted in minutes.

**The one thing to know.** The fix for slow, scary Terraform is almost always **smaller states with clear ownership**, not clever flags.

**Try it now:** `time terraform plan` in your capstone `envs/dev/web`, then `time terraform plan -refresh=false`. The difference is your refresh cost.

**You've got it if you can:** explain why `-target` is dangerous as a habit, and how you'd split a 2,000-resource state.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Where the time goes

| Phase | Cost driver | Topic |
|---|---|---|
| `init` | Downloading providers (the AWS provider is hundreds of MB) and modules | 10, 45 |
| Provider configure | Credential/STS calls; one per provider configuration (aliases multiply it) | 30 |
| **Refresh** | ≥ 1 read API call per resource **instance** (some resources need several); data sources too | 34 |
| Plan walk | Provider `PlanResourceChange` per instance (local, fast) | 13 |
| Apply | Real create/update/delete calls + waits ("Still creating...") | 13 |
| State I/O | Downloading/uploading a big state file | 35 |

### Levers

**1. Split state** (the big one). By:
- **lifecycle**: network (rarely changes) vs apps (daily),
- **ownership**: one team per state,
- **environment/region/account**,
- **blast radius**: stateful data in its own state with stricter approvals.

Rule of thumb: if a state routinely takes more than a couple of minutes to plan or has more than ~several hundred resources, look for a seam. Wire states together with outputs/SSM (topic 38) and orchestrate their order (pipelines, Terragrunt, Stacks, platforms).

**2. Refresh options** (topic 34):
- `-refresh=false`: skip reading reality; fast, **blind to drift**. Acceptable in tightly controlled pipelines where drift is checked separately (scheduled drift jobs) and only for plans you understand.
- `-refresh-only`: the opposite: only drift.
- Terraform 1.17 adds a `-minimal-refresh` plan option: it **only refreshes resources that have proposed changes**, instead of every resource in the state. That makes big plans much faster, but drift on untouched resources goes unnoticed in that run, so keep a full (normal) plan in scheduled drift detection (topic 37).

**3. `-parallelism=N`** (default 10):
- **Raise** (e.g. 20–30) when many independent resources are slow to create and the API allows it.
- **Lower** (e.g. 3–5) when you hit rate limits.

**4. API rate limits and retries**: providers retry throttled calls. The AWS provider has `max_retries` (default 25) and respects `AWS_RETRY_MODE`; very large applies across many states at once can still exceed account-level API quotas. Stagger pipelines; request quota increases for heavy APIs.

**5. CI caching**: set `TF_PLUGIN_CACHE_DIR` and cache it keyed on `.terraform.lock.hcl` (e.g. `actions/cache`), so each job doesn't re-download providers.

### ⚠️ `-target`

`terraform plan/apply -target=ADDRESS` plans **only** that resource and its dependencies. It prints a warning every time:

```
│ Warning: Resource targeting is in effect
│
│ You are creating a plan with the -target option, which means that the result
│ of this plan may not represent all of the changes requested by the current
│ configuration.
│
│ The -target option is not for routine use, and is provided only for
│ exceptional situations such as recovering from errors or mistakes, or when
│ Terraform specifically suggests to use it as part of an error message.
```

Why not as a habit: the rest of the configuration silently **drifts** from state (unapplied changes pile up), dependent resources may be left inconsistent, and the next full plan surprises someone. Legitimate uses: recovering from a broken apply, breaking a `count`/`for_each` unknown-value deadlock (topic 21), emergency fixes, always followed by a **full** plan/apply. (OpenTofu also has `-exclude`, the inverse.)

### Organisational patterns at scale

| Pattern | Why |
|---|---|
| **Clear ownership per state** (`CODEOWNERS`, team-owned repos) | Someone is responsible for every state's drift, upgrades and incidents |
| **Module catalogue** (private registry, versioned, tested) | Consistency without copy-paste (topics 44–45, 55) |
| **Golden pipeline** (one reusable workflow / platform) | Same checks and safety everywhere (topic 60) |
| **Change detection** in monorepos | Only plan roots affected by a change |
| **Orchestration across states** | Dependency order for applies (pipelines with `needs`, Terragrunt `run --all`, HCP Stacks) |
| **Policy as code** centrally maintained | Guardrails scale better than reviews (topic 54) |
| **Inventory and drift dashboards** | Know what's unmanaged or drifting (topic 37) |
| **State hygiene** | Remove dead resources, avoid huge data-only states, expire old state versions |

---

## Why it exists

Terraform works the same at 10 resources and 10,000, but the **costs** (time, API calls, risk, coordination) grow. Teams that don't adapt end up with 20-minute plans nobody reads, applies that fail on throttling, and "don't touch that state" folklore. Scaling Terraform is mostly about **architecture and ownership**, with a few flags for tuning.

---

## Analogy

**A national retail chain's inventory.** One giant inventory for the whole country takes days to count and one mistake affects every shop. Regional warehouses (split states), each with a manager (owner), standard shelving (module catalogue), the same audit checklist (golden pipeline), and a head office that schedules deliveries between regions in the right order (orchestration).

---

## How it works inside

```
time ≈ init + Σ(refresh calls per instance) / parallelism × latency
         + Σ(create/update waits on the critical path)
```

- Refresh and plan for **independent** resources run concurrently up to `-parallelism`.
- The **critical path** (the longest dependency chain, e.g. VPC → subnets → RDS (10+ min) → app) sets the minimum apply time no matter how high parallelism goes.
- Throttling turns into retries with backoff, which looks like "Terraform is hanging".

---

## Code / commands

### Measure

```bash
time terraform plan -input=false > /dev/null
time terraform plan -input=false -refresh=false > /dev/null
terraform state list | wc -l
terraform state list | cut -d. -f1-2 | sed 's/\[.*//' | sort | uniq -c | sort -rn | head
```

```
real    1m42.3s
real    0m6.8s
1843
    412 module.app.aws_route53_record.service
    388 module.app.aws_cloudwatch_metric_alarm.svc
    ...
```

- 95 s of the plan is refresh: that's API reads for 1,843 instances.
- The type breakdown shows where the bulk is (here, DNS records and alarms: candidates for their own states).

Find slow resources in a debug log:

```bash
TF_LOG=DEBUG TF_LOG_PATH=plan.log terraform plan >/dev/null
grep -E 'ReadResource|throttl|Rate exceeded|retrying' plan.log | head
```

### CI plugin caching (GitHub Actions)

```yaml
env:
  TF_PLUGIN_CACHE_DIR: ${{ github.workspace }}/.terraform.d/plugin-cache
steps:
  - run: mkdir -p "$TF_PLUGIN_CACHE_DIR"
  - uses: actions/cache@v4
    with:
      path: ${{ env.TF_PLUGIN_CACHE_DIR }}
      key: tf-plugins-${{ runner.os }}-${{ hashFiles('**/.terraform.lock.hcl') }}
```

### Tuning parallelism

```bash
terraform apply -parallelism=20      # big batch of independent, slow resources
terraform apply -parallelism=4       # API throttling
```

### Splitting a state (outline)

1. Decide the seam (e.g. move all `aws_cloudwatch_metric_alarm` + SNS into `envs/prod/monitoring`).
2. In the **new** root: write the config + `import` blocks for each resource (topic 39); plan → only imports.
3. In the **old** root: delete those blocks + `removed { destroy = false }` (topic 40); plan → only forgets.
4. Apply old (forget), then new (import). Replace cross-references with SSM/outputs.
5. Both plans: `No changes`. (Topic 42, scenario J.)

---

## Hands-on lab

1. Measure your capstone plans with and without refresh; count resources per state.
2. Create 200 cheap resources in a lab (`random_pet` with `count = 200`) and time `plan` with `-parallelism=1`, `10`, `50`.
3. Use `-target` on one lab resource and read the warning; then run a full plan and see what the targeted plan didn't show.
4. Add plugin caching to your capstone workflow and compare `init` times between runs.
5. Design (on paper) how you'd split a single state containing network, three databases, 20 services and monitoring.

---

## Where you'll use it

- **Platform teams** managing hundreds of states across dozens of accounts.
- **Performance tickets**: "PR plans take 15 minutes".
- **Incidents**: throttling during a big apply; a `-target` recovery followed by a full apply.
- **Architecture interviews**: "How would you structure Terraform for 50 teams?"

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| Plans take many minutes | Huge state; slow refresh | Split state; measure; refresh options carefully |
| `ThrottlingException: Rate exceeded` | Too many parallel API calls (or many pipelines at once) | Lower `-parallelism`; stagger runs; quota increases |
| Apply "hangs" | Retries with backoff, or a slow resource (RDS, CloudFront) | `TF_LOG=DEBUG` to see retries; wait; check timeouts |
| Next full plan shows surprising changes | Someone used `-target` and left the rest unapplied | Always follow targeted applies with a full plan/apply |
| `init` slow in CI | Providers downloaded every run | Plugin cache keyed on the lock file |
| Nobody owns a state | Organisational gap | Ownership in `CODEOWNERS`; drift alerts routed to the owner |

---

## Quick recap

- Plan time ≈ init + refresh (API reads per instance) + walk; apply time is bounded by the critical path.
- Biggest lever: **split states** by lifecycle/ownership/blast radius, wired with outputs/SSM and orchestrated.
- Tune with `-parallelism`, careful `-refresh=false`, provider retries, CI plugin caching.
- ⚠️ `-target` is for emergencies and recovery, always followed by a full plan.
- At scale, ownership, a module catalogue, a golden pipeline, policy and drift visibility matter more than flags.

### Interview questions

1. **Q: Your Terraform plans take 15 minutes. What do you do?**
   A: Measure first (plan with and without refresh, resource counts per type, debug logs for throttling). Then split the state along lifecycle and ownership seams using removed and import blocks, cache providers in CI, tune parallelism, and only skip refresh where drift is covered by scheduled checks.
2. **Q: When is `-target` appropriate?**
   A: Only in exceptional cases: recovering from a failed or broken apply, resolving an unknown-value deadlock Terraform itself suggests targeting for, or an emergency fix. Always follow it with a full plan and apply so the rest of the configuration doesn't drift.
3. **Q: How do you handle API rate limiting during large applies?**
   A: Lower `-parallelism`, rely on and tune provider retries, stagger pipelines that hit the same account, request quota increases for heavy APIs, and reduce the size of each state.
