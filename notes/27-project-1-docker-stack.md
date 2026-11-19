# 27 · Project 1: A Full App Stack in Docker

> **Level:** ★★ intermediate
>
> **You will learn:** how to put Parts 1–3 together in one real project: a three-tier app (**nginx** reverse proxy → **N Python app containers** → **PostgreSQL** with a persistent volume) built entirely with Terraform. You'll use variables, locals, `for_each`, `templatefile`, `random_password`, a locally **built** image that rebuilds when the code changes, health checks, a hidden dependency that needs `depends_on`, and outputs. Then you'll scale it, break it and fix it.
>
> **You need:** topics 14–26. Docker background: [Docker topic 17](../../Docker/notes/17-multi-stage-builds.md) (building images), [37](../../Docker/notes/37-compose-dependencies-health.md) (dependencies and health), [39a](../../Docker/notes/39a-tls-and-reverse-proxies.md) (reverse proxies).
>
> **Part 3 · Real infrastructure on your laptop** · Previous: [26 · Docker recap and the Docker provider](26-docker-provider.md) · Next: [28 · Terraform and Kubernetes (kind)](28-terraform-and-kubernetes.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A small but real system:

```
 browser ──▶ nginx (:8080) ──▶ app-a ─┐
                          └──▶ app-b ─┴──▶ postgres (volume: pgdata)
              all on one Docker network: shop-dev-net
```

Terraform builds the app image from `app/`, generates a DB password, starts Postgres, waits until it's healthy, starts one app container per name in `var.app_instances`, renders an nginx config listing them, and prints the URL.

**Picture it.** A small restaurant: a host at the door (nginx) seats guests at any free waiter (app containers), and all waiters write orders in the same ledger (Postgres) that stays in the safe overnight (the volume).

**The one thing to know.** `terraform apply -var='app_instances=["a","b","c"]'` scales the app. Watch the plan: one new app container, and nginx **replaced** because its config changed.

**Try it now:** build the project below, `terraform apply`, then `curl localhost:8080` a few times and watch `visits` go up and `served_by` change.

**You've got it if you can:** explain why the app needs `depends_on` on the database, and why nginx is replaced when you scale.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Project layout

```
~/tf-lab/27-stack/
├── versions.tf
├── variables.tf
├── main.tf
├── outputs.tf
├── terraform.tfvars
├── templates/
│   └── nginx.conf.tftpl
└── app/
    ├── Dockerfile
    ├── requirements.txt
    └── app.py
```

### What each piece demonstrates

| Piece | Concept | Topic |
|---|---|---|
| `var.app_instances` + `for_each` | Named, scalable instances | 22 |
| `local.name_prefix` | Naming convention | 16 |
| `random_password` | Generated secret (in state!) | 11, 51 |
| `docker_image` with `build` + `triggers` | Rebuild when source files change | 18 (`filesha1`, `fileset`) |
| `depends_on` on the DB container | A hidden dependency: the app reaches the DB by **name** | 20 |
| `replace_triggered_by` on the proxy | A runtime behaviour (nginx caches IPs) that Terraform can't see | 24 |
| `templatefile` for nginx | Config generated from live data | 18 |
| `healthcheck` + `wait = true` | Don't continue until healthy | 26 |
| Postgres volume | Data survives container replacement | 26 |
| Outputs | URL and a ready-to-use `psql` command | 15 |

---

## Why it exists

Single-resource labs teach syntax. Real work is about **how resources connect**: ordering, generated config, secrets, replacements that cascade, and data that must survive. This project gives you all of that for free on your laptop, before you spend money on AWS.

---

## Analogy

**A small restaurant opening for the night.**

- The **ledger** (Postgres) must be on the counter **and open** before waiters start (health check + `wait`).
- Waiters (app containers) know the ledger by its **name** ("db"), not by holding it, so nobody would notice the ordering requirement unless it's written down (`depends_on`).
- The **host** (nginx) needs the list of waiters on duty. Add a waiter and the host's list must be reprinted (nginx replaced).
- The **safe** (volume) keeps the ledger overnight, even if the counter is rebuilt.

---

## How it works inside

The dependency graph Terraform builds:

```
random_password.db ─────────────────────────────┐
docker_network.net ──┬──────────────┐            │
docker_volume.pgdata ┤              │            │
docker_image.postgres┴─▶ docker_container.db ◀───┘
                                    │  depends_on (hidden: DB reached by name)
docker_image.app ─────────▶ docker_container.app["a"], ["b"]   (parallel)
                                    │  references .name in the template
docker_image.nginx ───────▶ docker_container.proxy
```

- `docker_container.db` has `wait = true`: the provider polls Docker until the container's health status is `healthy` before reporting success. Only then do the app containers start.
- The proxy's `upload` content comes from `templatefile(...)` with the **app containers' names**, which is a real reference, so the proxy waits for all app containers, and any change to the list changes the content → **replacement**.

---

## Code / commands

### The app

`app/app.py`:

```python
import os
import socket

import psycopg
from flask import Flask

app = Flask(__name__)
DSN = os.environ["DATABASE_URL"]


@app.get("/health")
def health():
    return {"status": "ok"}


@app.get("/")
def index():
    with psycopg.connect(DSN) as conn:
        conn.execute(
            "CREATE TABLE IF NOT EXISTS visits (id serial PRIMARY KEY, at timestamptz DEFAULT now())"
        )
        conn.execute("INSERT INTO visits DEFAULT VALUES")
        count = conn.execute("SELECT count(*) FROM visits").fetchone()[0]
    return {
        "env": os.environ.get("APP_ENV", "dev"),
        "served_by": socket.gethostname(),
        "visits": count,
    }
```

`app/requirements.txt`:

```
flask==3.1.*
gunicorn==23.*
psycopg[binary]==3.2.*
```

`app/Dockerfile` (patterns from [Docker topic 20](../../Docker/notes/20-dockerfile-best-practices.md)):

```dockerfile
FROM python:3.13-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
RUN useradd --uid 10001 --no-create-home app
USER app
EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "2", "app:app"]
```

### The nginx template

`templates/nginx.conf.tftpl`:

```
upstream app {
%{ for name in app_hosts ~}
    server ${name}:8000;
%{ endfor ~}
}

server {
    listen 80;

    location / {
        proxy_pass http://app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

`$host` and `$remote_addr` are **nginx** variables, not Terraform ones: a `$` not followed by `{` is left alone by Terraform templates.

### Terraform

`versions.tf`:

```hcl
terraform {
  required_version = ">= 1.14"
  required_providers {
    docker = { source = "kreuzwerker/docker", version = "~> 4.7" }
    random = { source = "hashicorp/random", version = "~> 3.7" }
  }
}

provider "docker" {}          # default: local socket or DOCKER_HOST
```

`variables.tf`:

```hcl
variable "environment" {
  type        = string
  description = "Environment name, used in container names."
  default     = "dev"
}

variable "app_instances" {
  type        = set(string)
  description = "Names of app containers to run behind nginx."
  default     = ["a", "b"]

  validation {
    condition     = length(var.app_instances) >= 1 && length(var.app_instances) <= 5
    error_message = "Run between 1 and 5 app instances."
  }
}

variable "http_port" {
  type    = number
  default = 8080
}

variable "postgres_version" {
  type    = string
  default = "17-alpine"
}
```

`main.tf`:

```hcl
locals {
  name_prefix = "shop-${var.environment}"
  db_name     = "shop"
  db_user     = "shop"
  app_dir     = "${path.module}/app"

  # hash of every file in app/: changes whenever the code changes
  app_source_hash = sha1(join("", [
    for f in sort(fileset(local.app_dir, "**")) : filesha1("${local.app_dir}/${f}")
  ]))
}

# ── Network and storage ─────────────────────────────────────────────
resource "docker_network" "net" {
  name = "${local.name_prefix}-net"
}

resource "docker_volume" "pgdata" {
  name = "${local.name_prefix}-pgdata"

  lifecycle {
    prevent_destroy = false    # set to true for data you can't lose (topic 24)
  }
}

# ── Secrets ─────────────────────────────────────────────────────────
resource "random_password" "db" {
  length  = 24
  special = false              # keeps the connection URL simple
}

# ── Images ──────────────────────────────────────────────────────────
resource "docker_image" "postgres" {
  name         = "postgres:${var.postgres_version}"
  keep_locally = true
}

resource "docker_image" "nginx" {
  name         = "nginx:1.29"
  keep_locally = true
}

resource "docker_image" "app" {
  name = "${local.name_prefix}-app:${substr(local.app_source_hash, 0, 12)}"

  build {
    context = local.app_dir
  }

  triggers = {
    source = local.app_source_hash   # any change → rebuild (replace the image)
  }
}

# ── Database ────────────────────────────────────────────────────────
resource "docker_container" "db" {
  name    = "${local.name_prefix}-db"
  image   = docker_image.postgres.image_id
  restart = "unless-stopped"

  env = [
    "POSTGRES_DB=${local.db_name}",
    "POSTGRES_USER=${local.db_user}",
    "POSTGRES_PASSWORD=${random_password.db.result}",
  ]

  networks_advanced {
    name    = docker_network.net.name
    aliases = ["db"]
  }

  volumes {
    volume_name    = docker_volume.pgdata.name
    container_path = "/var/lib/postgresql/data"
  }

  healthcheck {
    test     = ["CMD-SHELL", "pg_isready -U ${local.db_user} -d ${local.db_name}"]
    interval = "5s"
    timeout  = "3s"
    retries  = 10
  }

  wait         = true      # don't report "created" until healthy
  wait_timeout = 90
}

# ── App instances ───────────────────────────────────────────────────
resource "docker_container" "app" {
  for_each = var.app_instances

  name    = "${local.name_prefix}-app-${each.key}"
  image   = docker_image.app.image_id
  restart = "unless-stopped"

  env = [
    "APP_ENV=${var.environment}",
    "DATABASE_URL=postgresql://${local.db_user}:${random_password.db.result}@db:5432/${local.db_name}",
  ]

  networks_advanced {
    name = docker_network.net.name
  }

  healthcheck {
    test     = ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health')"]
    interval = "10s"
    timeout  = "3s"
    retries  = 3
  }

  # The app reaches the database by the NAME "db", not through a Terraform
  # reference, so Terraform can't see this dependency. Without it, apps could
  # start before Postgres is ready.
  depends_on = [docker_container.db]
}

# ── Reverse proxy ───────────────────────────────────────────────────
resource "docker_container" "proxy" {
  name    = "${local.name_prefix}-proxy"
  image   = docker_image.nginx.image_id
  restart = "unless-stopped"

  ports {
    internal = 80
    external = var.http_port
  }

  networks_advanced {
    name = docker_network.net.name
  }

  upload {
    file = "/etc/nginx/conf.d/default.conf"
    content = templatefile("${path.module}/templates/nginx.conf.tftpl", {
      app_hosts = sort([for c in docker_container.app : c.name])   # real reference → waits for apps
    })
  }

  lifecycle {
    # nginx resolves upstream names to IPs only at startup. When app containers
    # are replaced they get new IPs, so restart (replace) the proxy too.
    replace_triggered_by = [docker_container.app]
  }
}
```

`outputs.tf`:

```hcl
output "url" {
  value = "http://localhost:${var.http_port}"
}

output "app_containers" {
  value = sort([for c in docker_container.app : c.name])
}

output "psql_command" {
  description = "Open a SQL shell in the database container."
  value       = "docker exec -it ${docker_container.db.name} psql -U ${local.db_user} -d ${local.db_name}"
}

output "db_password" {
  value     = random_password.db.result
  sensitive = true
}
```

### Apply

```bash
cd ~/tf-lab/27-stack
terraform init
terraform apply
```

Trimmed output, showing the order:

```
random_password.db: Creating...
docker_network.net: Creating...
docker_volume.pgdata: Creating...
docker_image.nginx: Creating...
docker_image.postgres: Creating...
docker_image.app: Creating...
...
docker_image.app: Still creating... [00m20s elapsed]
docker_container.db: Creating...
docker_container.db: Still creating... [00m10s elapsed]
docker_container.db: Creation complete after 12s [id=5d1e...]
docker_image.app: Creation complete after 31s [id=sha256:9a7c...shop-dev-app:3f1c2a9b8e7d]
docker_container.app["a"]: Creating...
docker_container.app["b"]: Creating...
docker_container.app["a"]: Creation complete after 1s [id=...]
docker_container.app["b"]: Creation complete after 1s [id=...]
docker_container.proxy: Creating...
docker_container.proxy: Creation complete after 1s [id=...]

Apply complete! Resources: 10 added, 0 changed, 0 destroyed.

Outputs:

app_containers = [
  "shop-dev-app-a",
  "shop-dev-app-b",
]
db_password = <sensitive>
psql_command = "docker exec -it shop-dev-db psql -U shop -d shop"
url = "http://localhost:8080"
```

- The images and the DB start in parallel with the network and volume; `db` waited **12s** for its health check (`wait = true`).
- The app containers started only after **both** the image build and the DB were done.
- The proxy started last, because its template references the app containers.

### Test it

```bash
for i in 1 2 3 4; do curl -s localhost:8080; echo; done
```

```
{"env":"dev","served_by":"5c0f8e2a91d4","visits":1}
{"env":"dev","served_by":"a17b3d6e0c55","visits":2}
{"env":"dev","served_by":"5c0f8e2a91d4","visits":3}
{"env":"dev","served_by":"a17b3d6e0c55","visits":4}
```

- `served_by` alternates: nginx load-balances across the two app containers (the hostname is the container ID).
- `visits` keeps counting: both apps share the database.

### Scale out

```bash
terraform plan -var='app_instances=["a","b","c"]'
```

```
  # docker_container.app["c"] will be created
  + resource "docker_container" "app" {
      + name = "shop-dev-app-c"
      ...
    }

  # docker_container.proxy must be replaced
-/+ resource "docker_container" "proxy" {
      ...
      - upload { # forces replacement
          - content = <<-EOT
                upstream app {
                    server shop-dev-app-a:8000;
                    server shop-dev-app-b:8000;
                }
                ...
            EOT -> null
          ...
        }
      + upload { # forces replacement
          + content = <<-EOT
                upstream app {
                    server shop-dev-app-a:8000;
                    server shop-dev-app-b:8000;
                    server shop-dev-app-c:8000;
                }
                ...
            EOT
          ...
        }
    }

Plan: 2 to add, 0 to change, 1 to destroy.
```

- `app["a"]` and `app["b"]` aren't touched (`for_each`).
- The new config is **fully known** at plan time: container names come from your config (`"${local.name_prefix}-app-${each.key}"`), not from Docker. Only provider-generated values like IDs are `(known after apply)`.
- The proxy is replaced because `upload` forces replacement. Expect a second or two of downtime on the proxy. (In production you'd reload nginx instead, or use a load balancer that discovers backends itself, like a Kubernetes Service.)

### Change the code → rebuild → roll

Edit `app/app.py` (e.g. add `"version": 2` to the response) and plan:

```
  # docker_image.app must be replaced
-/+ resource "docker_image" "app" {
      ~ name     = "shop-dev-app:3f1c2a9b8e7d" -> "shop-dev-app:b82e41c09d3a" # forces replacement
      ~ triggers = { # forces replacement
          ~ "source" = "3f1c2a9b8e7d..." -> "b82e41c09d3a..."
        }
    }

  # docker_container.app["a"] must be replaced
-/+ resource "docker_container" "app" {
      ~ image = "sha256:9a7c..." -> (known after apply) # forces replacement
      ...
  # docker_container.app["b"] must be replaced
  ...
  # docker_container.proxy will be replaced due to changes in replace_triggered_by
-/+ resource "docker_container" "proxy" {
      ...
Plan: 4 to add, 0 to change, 4 to destroy.
```

A full cascade, and every link has a different reason:

1. `docker_image.app`: the `triggers` hash changed → **forces replacement**.
2. `app["a"]`, `app["b"]`: their `image` is the new image's ID, unknown until it's built → **forces replacement** (cause 2 in topic 24).
3. `proxy`: its **config file is unchanged** (same names), so nothing in its own arguments forces anything. It's replaced only because of **`replace_triggered_by`**.

Why that last rule matters: without it, the proxy would keep running with the **old** app containers' IP addresses (nginx resolved them once, at startup), and every request would fail with `502 Bad Gateway` after the apply. Terraform can't see that kind of runtime behaviour. You have to know it and encode it. Try it in the lab.

### Data survives the database container

```bash
terraform apply -replace=docker_container.db -auto-approve
curl -s localhost:8080
```

```
{"env":"dev","served_by":"...","visits":5}
```

The DB container was recreated (`-replace`, topic 41), but `visits` continues from 4: the data lives in the `pgdata` **volume**.

---

## Hands-on lab

1. Build and apply the whole project. Hit the URL several times.
2. Look at the graph: `terraform graph | dot -Tpng > graph.png` (or `-format=mermaid`). Match it to the diagram above.
3. Scale to `["a","b","c"]`, then remove `"a"`. Each time, read the plan **before** applying and predict the changes.
4. Change the app code and watch the cascade. Then **remove the `replace_triggered_by`** block, change the code again, apply, and `curl localhost:8080`: you'll likely get `502 Bad Gateway` (nginx still points at the old IPs). `docker restart shop-dev-proxy` fixes it by hand. Put the block back.
5. **Break it 1: remove the `depends_on`**, destroy everything, and apply again. Do the apps fail at first? (Often they start fine because Flask only connects on a request, so the bug is hidden. That's why hidden dependencies are dangerous: they work until they don't.) Put it back.
6. **Break it 2: drift.** `docker stop shop-dev-app-b`, then `terraform plan`. Then `docker rm -f shop-dev-proxy` and plan again.
7. **Break it 3: secret in state.** `terraform output -raw db_password`, and `grep -c POSTGRES_PASSWORD terraform.tfstate`. The password is in state and in the container's env. Topic 51 shows better ways.
8. Use the `psql_command` output to open the database and run `SELECT count(*) FROM visits;`.
9. `terraform destroy`. Then `docker volume ls | grep shop`: the volume is gone too, with the data. Add `prevent_destroy = true` to the volume and see what `destroy` says.

---

## Where you'll use it

- The same **shape** (load balancer → app instances → database, plus secrets, health checks, generated config) appears in every AWS project: ALB (Application Load Balancer) → EC2/ECS → RDS. You'll build that version in the capstone.
- **Hidden-dependency thinking**: "does this resource talk to something by name rather than by reference?"
- **Image build triggers** are a pattern you'll see with Lambda packages and container images in CI.

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `container exited` / app restarting, logs say `connection refused` to `db` | Apps started before Postgres was ready. | `depends_on` + DB health check + `wait = true`. |
| Proxy exits: `host not found in upstream "shop-dev-app-a:8000"` | nginx resolves upstream names at start, and an app container doesn't exist (yet). | Make the proxy reference the app containers (as here), so it starts after them. |
| `timeout waiting for container ... to become healthy` | `wait = true` but the health check never passes. | `docker logs shop-dev-db`; check the health command, increase `wait_timeout`. |
| `failed to solve: ... requirements.txt: not found` during the image build | Wrong `build.context`. | Context must be the folder containing the Dockerfile and files. |
| `502 Bad Gateway` right after the app containers were replaced | nginx cached the old containers' IPs at startup. | Replace/restart the proxy with the apps (`replace_triggered_by`), or configure nginx's `resolver` for runtime DNS. |
| Image never rebuilds after code changes | No trigger tied to the source. | Use `triggers` with a hash of the files (as here). |
| Changing the DB password breaks the app | Postgres only reads `POSTGRES_PASSWORD` on **first** init of the volume. | Change it inside Postgres (`ALTER USER`), or recreate the volume (data loss). |
| `port is already allocated` | Port 8080 in use. | Change `http_port`. |

---

## Quick recap

- One configuration builds a three-tier app: image build, network, volume, secret, DB, N apps, proxy.
- `for_each` scales apps cleanly; generated config makes the proxy replace when the app list changes.
- Hidden dependencies (talking by name) need `depends_on`; health checks + `wait` make "created" mean "ready".
- Volumes keep data across container replacement; secrets end up in state.
- Reading the cascade in a plan tells you what each change really costs.

### Interview questions

1. **Q: Your app container starts before the database is ready. How do you fix it in Terraform?**
   A: Add `depends_on` to the database resource, because the app reaches it by name rather than a reference, and make the database resource wait for health (a health check plus the provider's wait option) so "created" means "ready".
2. **Q: How do you make Terraform rebuild an image when source code changes?**
   A: Put a hash of the source files into the image resource's `triggers` (and/or tag), e.g. `sha1` over `filesha1` of every file from `fileset`. A changed hash replaces the image, which cascades to containers using its ID.
3. **Q: Why did adding one app container replace the proxy?**
   A: The proxy's config file is rendered from the list of app container names. Adding a name changes the uploaded content, and changing an uploaded file forces replacement of the container.
4. **Q: After a deploy replaced the app containers, the proxy returns 502. Why, and how do you prevent it in Terraform?**
   A: The proxy resolved the app names to IPs only at startup, and the new containers have new IPs. Use `replace_triggered_by` on the proxy pointing at the app containers (or make the proxy re-resolve DNS at runtime).
