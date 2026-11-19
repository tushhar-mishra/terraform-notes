# 26 · Docker Recap and the Docker Provider

> **Level:** ★ foundation
>
> **You will learn:** a short recap of Docker images, containers, networks, volumes and the daemon socket from your Docker notes, then how Terraform manages them with the community **`kreuzwerker/docker`** provider (v4): the provider block, `docker_image`, `docker_container`, `docker_network`, `docker_volume`, what happens on the Docker socket during `apply`, and how Terraform-managed containers differ from `docker run` and Docker Compose. This is your first "real infrastructure" provider, and it's free.
>
> **You need:** topics 05 (calling the Docker API by hand), 10 (providers), 20–24. Docker background: [Docker topics 03](../../Docker/notes/03-docker-architecture.md), [06](../../Docker/notes/06-images-tags-and-registries.md), [24](../../Docker/notes/24-volumes.md), [27](../../Docker/notes/27-docker-network-drivers.md), [35](../../Docker/notes/35-compose-basics.md).
>
> **Part 3 · Real infrastructure on your laptop** · Previous: [25 · Custom conditions and `check` blocks](25-conditions-and-checks.md) · Next: [27 · Project 1: a full app stack in Docker](27-project-1-docker-stack.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** The Docker provider calls the same Docker API you called with `curl` in topic 05. Instead of `docker pull` and `docker run`, you declare an image and a container:

```hcl
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"    # community provider, NOT hashicorp/docker
      version = "~> 4.7"
    }
  }
}

provider "docker" {
  host = "unix:///var/run/docker.sock"
}

resource "docker_image" "nginx" {
  name         = "nginx:1.29"
  keep_locally = true                   # don't delete the image on destroy
}

resource "docker_container" "web" {
  name  = "tf-web"
  image = docker_image.nginx.image_id   # reference → pull before run
  ports {
    internal = 80
    external = 8080
  }
}
```

**Picture it.** `docker run` is ordering a meal by shouting at the kitchen. Terraform is a written order that the kitchen keeps, checks against what's on the table, and can cancel completely.

**The one thing to know.** Terraform now **owns** this container. Change something with `docker` commands and the next plan shows drift. Remove the block and Terraform removes the container.

**Try it now:** apply the code above, then `curl -s localhost:8080 | head -4`, then `docker rm -f tf-web` and run `terraform plan`.

**You've got it if you can:** explain what `image = docker_image.nginx.image_id` does for ordering, and what `keep_locally` changes.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Docker recap (only what Terraform needs)

From your Docker notes:

| Docker idea | Recap | Docker notes |
|---|---|---|
| **Daemon** (`dockerd`) | The background service that does the work. Clients talk to it through an API on the Unix socket `/var/run/docker.sock`. | [03](../../Docker/notes/03-docker-architecture.md) |
| **Image** | A read-only template (layers + config), named `repo:tag` and identified by an ID (`sha256:...`). | [06](../../Docker/notes/06-images-tags-and-registries.md) |
| **Container** | A running (or stopped) instance of an image, with its own name, network and writable layer. | [08](../../Docker/notes/08-container-lifecycle.md) |
| **Published port** | `-p 8080:80`: host port 8080 forwards to container port 80. | [28](../../Docker/notes/28-publishing-ports.md) |
| **User-defined bridge network** | Containers on it can reach each other **by name** (Docker's embedded DNS). | [27](../../Docker/notes/27-docker-network-drivers.md), [29](../../Docker/notes/29-container-dns.md) |
| **Named volume** | Storage managed by Docker that outlives containers. | [24](../../Docker/notes/24-volumes.md) |
| **Health check** | A command Docker runs to mark a container `healthy`/`unhealthy`. | [32](../../Docker/notes/32-restart-and-healthchecks.md) |
| **Compose** | A YAML file describing several containers, run with `docker compose up`. | [35](../../Docker/notes/35-compose-basics.md) |

### The provider

- Source: **`kreuzwerker/docker`** (a community provider; HashiCorp's own Docker provider was archived years ago). Current major version: **4.x**. Upgrading from 3.x needs almost no code changes; the only removed argument is `check_duplicate` on `docker_network`.
- The `provider "docker"` block's `host` says where the daemon is. Default: the local socket (or `DOCKER_HOST` if set). It can also be `ssh://user@server` or `tcp://host:2376` with TLS for a remote daemon.

### The four core resources

| Resource | Equivalent command | Key arguments | Key attributes |
|---|---|---|---|
| `docker_image` | `docker pull` / `docker build` | `name`, `keep_locally`, `build { context }` | `image_id` |
| `docker_container` | `docker run` | `name`, `image`, `ports {}`, `env`, `networks_advanced {}`, `volumes {}`, `healthcheck {}`, `restart`, `command`, `upload {}` | `id`, `network_data` |
| `docker_network` | `docker network create` | `name`, `driver`, `ipam_config {}` | `id` |
| `docker_volume` | `docker volume create` | `name`, `driver` | `mountpoint` |

Notes:

- `docker_container.image` should be **`docker_image.x.image_id`**: the reference makes Terraform pull first, and when the tag points to a new image ID, the container is **replaced** to use it.
- `env` is a **set of strings** like `["KEY=value"]`.
- `ports`, `volumes`, `networks_advanced`, `healthcheck`, `upload` are **nested blocks** (good `dynamic` candidates, topic 23).
- `keep_locally = true` means `terraform destroy` won't `docker rmi` the image (useful if other containers use it).

---

## Why it exists

Why manage containers with Terraform at all, when Compose exists?

- **Learning**: it's the cheapest way to practise Terraform with real infrastructure, real drift and real replacements, with zero cloud cost.
- **One workflow**: in some setups, Terraform provisions a VM, then the Docker provider (pointed at that VM over SSH) runs its containers, all in one plan.
- **State and plan**: you get previews, drift detection and clean destruction, which Compose only partly offers.

When **not** to use it: Compose is simpler for local development, and Kubernetes (with Helm/GitOps) is the production standard for many containers. Topic 27 compares them.

---

## Analogy

**Ordering at a restaurant.**

- `docker run` = shouting an order at the kitchen. It gets cooked, but there's no ticket. Tomorrow nobody knows who ordered what.
- `docker compose` = a printed party menu: easy to repeat, but the kitchen doesn't keep track if a waiter quietly swaps a dish.
- Terraform + Docker provider = a **ticketed order**: the kitchen keeps the ticket (state), compares the table with the ticket before every change (refresh), shows you what it'll change (plan), and can clear the whole table (destroy).

---

## How it works inside

What happens on the socket during `terraform apply` for the Core example (compare with your topic 05 `curl` calls):

```
docker_image.nginx    Create
   provider → GET  /images/nginx:1.29/json           (already present?)
            → POST /images/create?fromImage=nginx&tag=1.29   (pull if not)
            ← image ID sha256:4e1b...              → state: image_id
docker_container.web  Create   (waits for the image: reference = dependency)
   provider → POST /containers/create?name=tf-web
                {"Image":"sha256:4e1b...","HostConfig":{"PortBindings":{"80/tcp":[{"HostPort":"8080"}]}}}
            → POST /containers/<id>/start
            → GET  /containers/<id>/json            (read back the real settings)
            ← id, network_data, ...                → state
```

On every later plan, **refresh** calls `GET /containers/<id>/json`. If the container is gone, the provider removes it from state and the plan says `will be created`. If settings differ, you get an update or replacement.

Most `docker_container` arguments **force replacement**, because Docker can't change a container's image, ports or env after creation. That matches Docker's own model: containers are immutable; you recreate them.

---

## Code / commands

`~/tf-lab/26-docker/`:

`versions.tf`:

```hcl
terraform {
  required_version = ">= 1.14"
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 4.7"
    }
  }
}

provider "docker" {
  host = "unix:///var/run/docker.sock"
}
```

`main.tf`:

```hcl
resource "docker_network" "lab" {
  name = "tf-lab-net"
}

resource "docker_volume" "html" {
  name = "tf-lab-html"
}

resource "docker_image" "nginx" {
  name         = "nginx:1.29"
  keep_locally = true
}

resource "docker_container" "web" {
  name    = "tf-web"
  image   = docker_image.nginx.image_id
  restart = "unless-stopped"

  ports {
    internal = 80
    external = 8080
  }

  networks_advanced {
    name    = docker_network.lab.name
    aliases = ["web"]               # other containers on the network can use "web"
  }

  volumes {
    volume_name    = docker_volume.html.name
    container_path = "/usr/share/nginx/html"
  }

  upload {                          # copy a file into the container at create time
    file    = "/usr/share/nginx/html/index.html"
    content = "<h1>Served by a Terraform-managed container</h1>\n"
  }

  healthcheck {
    # the nginx image has no curl; bash's /dev/tcp opens a TCP connection to port 80
    test     = ["CMD", "bash", "-c", "exec 3<>/dev/tcp/127.0.0.1/80"]
    interval = "10s"
    timeout  = "3s"
    retries  = 3
  }
}

output "url" {
  value = "http://localhost:${docker_container.web.ports[0].external}"
}
```

### Plan (trimmed)

```
$ terraform init && terraform plan
  # docker_container.web will be created
  + resource "docker_container" "web" {
      + id       = (known after apply)
      + image    = (known after apply)
      + name     = "tf-web"
      + restart  = "unless-stopped"
      ...
      + healthcheck {
          + interval = "10s"
          + retries  = 3
          + test     = [
              + "CMD",
              + "bash",
              + "-c",
              + "exec 3<>/dev/tcp/127.0.0.1/80",
            ]
          + timeout  = "3s"
        }
      + networks_advanced {
          + aliases = [
              + "web",
            ]
          + name    = "tf-lab-net"
        }
      + ports {
          + external = 8080
          + internal = 80
          + ip       = "0.0.0.0"
          + protocol = "tcp"
        }
      + upload { ... }
      + volumes {
          + container_path = "/usr/share/nginx/html"
          + volume_name    = "tf-lab-html"
        }
    }

  # docker_image.nginx will be created
  + resource "docker_image" "nginx" {
      + id           = (known after apply)
      + image_id     = (known after apply)
      + keep_locally = true
      + name         = "nginx:1.29"
    }

  # docker_network.lab will be created
  ...
  # docker_volume.html will be created
  ...
Plan: 4 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + url = "http://localhost:8080"
```

- `image = (known after apply)`: the image ID isn't known until it's pulled. That's the reference that orders the two resources.
- `ip = "0.0.0.0"`, `protocol = "tcp"`: provider **defaults** for the port block. `0.0.0.0` means "all host interfaces" (from Docker topic 28: bind to `127.0.0.1` with `ip = "127.0.0.1"` to keep it local).

### Apply and check

```bash
terraform apply -auto-approve
curl -s localhost:8080
docker ps --filter name=tf-web --format '{{.Names}}  {{.Image}}  {{.Status}}'
```

```
<h1>Served by a Terraform-managed container</h1>
tf-web  sha256:4e1b...  Up 15 seconds (healthy)
```

- The **Image** column shows the image **ID**, because we passed `image_id`, not `nginx:1.29`.

### Drift: change the container behind Terraform's back

```bash
docker rm -f tf-web
terraform plan
```

```
docker_container.web: Refreshing state... [id=8c2f...]

Note: Objects have changed outside of Terraform
  # docker_container.web has been deleted
  ...
  # docker_container.web will be created
  ...
Plan: 1 to add, 0 to change, 0 to destroy.
```

Apply to put it back. The **volume** wasn't touched, so if you'd changed `index.html` inside it, the change would survive: volumes outlive containers.

### A replacement

Change `external = 8080` to `8081` and plan:

```
  # docker_container.web must be replaced
-/+ resource "docker_container" "web" {
      ...
      ~ ports { # forces replacement
          ~ external = 8080 -> 8081 # forces replacement
            # (3 unchanged attributes hidden)
        }
    }
```

Docker can't change a running container's port mappings, so it's destroy + create. For a web server, that's a few seconds of downtime. `create_before_destroy` wouldn't help here: two containers can't share the name `tf-web` (topic 24's trap).

### Terraform vs `docker run` vs Compose

| | `docker run` | `docker compose` | Terraform + Docker provider |
|---|---|---|---|
| Record of what was created | None | The compose file | Code + **state** |
| Preview before change | No | No (`compose up` just does it) | **Plan** |
| Detects manual changes | No | Partly (recreates on config change) | **Yes**, on refresh |
| Removes things you deleted from the file | No | `--remove-orphans` | **Yes** |
| Best for | One-off commands | Local development | Learning Terraform; Docker hosts managed alongside other infrastructure |

---

## Hands-on lab

1. Build the project. `terraform apply`, open `http://localhost:8080` (from Windows too: WSL2 forwards localhost).
2. Compare Terraform's view with Docker's: `terraform state show docker_container.web` vs `docker inspect tf-web | jq '.[0].HostConfig.PortBindings'`.
3. Drift: `docker rm -f tf-web` → plan → apply. Then `docker stop tf-web` → plan: what does the provider say about a stopped container? (In 4.x, `must_run` defaults to `true`, so the plan proposes to start it again.)
4. Replacement: change the port to `8081`. Then change `nginx:1.29` to `nginx:1.30` and see **both** the image and the container change.
5. Add a second container `tf-client` from the image `curlimages/curl:8.11.1` with `command = ["sleep", "infinity"]` on the same network. Then `docker exec tf-client curl -s web` (resolved by name through the network alias).
6. Watch the API calls: `TF_LOG=DEBUG terraform apply 2>&1 | grep -E 'POST|GET' | head`.
7. `terraform destroy`. Check that `docker images nginx` still shows the image (`keep_locally = true`) and the volume is gone (`docker volume ls`).

---

## Where you'll use it

- **This course**: Docker is your free lab for topics 27 and 28 (kind runs on Docker too).
- **Single-host deployments**: small teams sometimes run a VM with Docker and manage its containers with Terraform over SSH (`host = "ssh://deploy@10.0.1.5"`).
- **CI test environments**: spin up a database container for integration tests, destroy it after.
- **Understanding every other provider**: create/read/replace/drift work the same way with AWS; only the API is different.

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `Could not retrieve the list of available versions for provider hashicorp/docker` | No `required_providers` entry, so Terraform guessed `hashicorp/docker` (doesn't exist). | Declare `source = "kreuzwerker/docker"`. |
| `Error pinging Docker server: ... permission denied ... docker.sock` | Your user can't use the socket. | Add yourself to the `docker` group and re-login ([Docker topic 04](../../Docker/notes/04-installing-docker.md)). |
| `Cannot connect to the Docker daemon at unix:///var/run/docker.sock` | Docker isn't running in WSL2. | `sudo systemctl start docker` (or `sudo service docker start`). |
| `Conflict. The container name "/tf-web" is already in use` | A container with that name exists outside Terraform. | Remove it, or import it (topic 39). |
| `Bind for 0.0.0.0:8080 failed: port is already allocated` | Another process/container uses the port. | `ss -ltnp | grep 8080` ([Linux topic 058](../../linux-mastery-rhel/notes/058-how-networks-work.md)); pick another port. |
| `container exited immediately` / `container ... is not running` after create | The process in the container ended (bad command, crash). | `docker logs <name>`; fix the command/env. |
| Every plan shows changes on `docker_container` | An attribute normalised differently by Docker (e.g. env order, default values). | Set the value explicitly as Docker reports it, or `ignore_changes` for that attribute. |
| `An argument named "check_duplicate" is not expected here` after upgrading | Removed in provider 4.0. | Delete the argument. |

---

## Quick recap

- The **`kreuzwerker/docker`** provider (v4) manages images, containers, networks and volumes through the Docker socket API.
- Use `image = docker_image.x.image_id` so pulls happen first and new image IDs replace containers.
- Most container settings **force replacement**, like Docker itself (containers are immutable).
- Terraform adds state, plans, drift detection and clean destroy on top of what `docker run`/Compose do.

### Interview questions

1. **Q: How does Terraform's Docker provider talk to Docker?**
   A: Through the Docker Engine API, by default on the Unix socket `/var/run/docker.sock` (or `DOCKER_HOST`, SSH or TCP with TLS). It makes the same API calls the `docker` CLI makes.
2. **Q: Why does changing a container's port replace it?**
   A: Docker can't change port bindings, image or env on an existing container. The provider marks those arguments as forcing replacement, so Terraform destroys and recreates it.
3. **Q: Would you use Terraform or Compose to run containers locally?**
   A: Compose for day-to-day local development: simpler and faster. Terraform when the containers are part of infrastructure you manage alongside other resources, or when you need plan, state and drift detection.
