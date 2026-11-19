# 28 · Terraform and Kubernetes (kind)

> **Level:** ★★ intermediate
>
> **You will learn:** a short Kubernetes recap, then how Terraform manages things **inside** a cluster with the **`kubernetes`** provider (v3: `kubernetes_namespace_v1`, `kubernetes_deployment_v1`, `kubernetes_service_v1`, `kubernetes_config_map_v1`, `kubernetes_manifest`) and the **`helm`** provider (v3: `helm_release`), on a free **kind** cluster. You'll also learn the "provider configured from a cluster in the same run" problem, drift with `kubectl`, Terraform vs `kubectl apply` vs Helm vs GitOps, and the rule of thumb: **Terraform builds the cluster and its platform add-ons; GitOps deploys the apps.**
>
> **You need:** topics 10 (providers, aliases), 13 (provider configuration happens early), 20, 24 (`ignore_changes`), 26–27. Kubernetes background: [Kubernetes topics 06](../../CKA-Certification-Course-2025/notes/06-the-cluster-at-a-glance.md), [11](../../CKA-Certification-Course-2025/notes/11-lab-setup-kind-and-minikube.md), [12](../../CKA-Certification-Course-2025/notes/12-kubectl-kubeconfig-and-contexts.md), [25](../../CKA-Certification-Course-2025/notes/25-deployments.md), [30](../../CKA-Certification-Course-2025/notes/30-services-clusterip.md), [65](../../CKA-Certification-Course-2025/notes/65-helm.md), [68](../../CKA-Certification-Course-2025/notes/68-gitops.md).
>
> **Part 3 · Real infrastructure on your laptop** · Previous: [27 · Project 1: a full app stack in Docker](27-project-1-docker-stack.md) · Next: [29 · AWS basics for Terraform](29-aws-basics.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** The `kubernetes` provider turns Kubernetes objects into Terraform resources. The `helm` provider installs Helm charts. Both find the cluster through a **kubeconfig** file, like `kubectl` does.

```hcl
provider "kubernetes" {
  config_path    = "~/.kube/config"
  config_context = "kind-tf-lab"
}

resource "kubernetes_namespace_v1" "shop" {
  metadata {
    name = "shop"
  }
}

resource "helm_release" "podinfo" {
  name       = "podinfo"
  repository = "https://stefanprodan.github.io/podinfo"
  chart      = "podinfo"
  namespace  = kubernetes_namespace_v1.shop.metadata[0].name
}
```

**Picture it.** Kubernetes is a self-running factory with its own managers (controllers). Terraform is the **architect** who builds the factory and installs the big machines. The **daily production schedule** (which app version runs today) is better left to the factory's own planning office (GitOps).

**The one thing to know.** Manage the **cluster** and its **platform add-ons** (ingress controller, cert-manager, monitoring, Argo CD) with Terraform. Deploy **applications** with Helm/GitOps. And don't create a cluster and configure its `kubernetes` provider in the **same** configuration.

**Try it now:** `kind create cluster --name tf-lab`, then apply the code above and run `kubectl -n shop get pods`.

**You've got it if you can:** explain why `kubectl scale` shows up as drift in the next `terraform plan`, and who should own an app's replica count.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Kubernetes recap (only what Terraform needs)

| Idea | Recap | Kubernetes notes |
|---|---|---|
| **API server** | Everything in Kubernetes is an object you create through the API server. `kubectl` is just an API client. | [06](../../CKA-Certification-Course-2025/notes/06-the-cluster-at-a-glance.md), [14](../../CKA-Certification-Course-2025/notes/14-the-kubernetes-api.md) |
| **kubeconfig / context** | A file (`~/.kube/config`) with cluster URLs, credentials and **contexts** (cluster + user + namespace). | [12](../../CKA-Certification-Course-2025/notes/12-kubectl-kubeconfig-and-contexts.md) |
| **Declarative objects** | You submit desired state (YAML); controllers keep reality matching it, **all the time**. | [15](../../CKA-Certification-Course-2025/notes/15-imperative-vs-declarative.md) |
| **Namespace, Deployment, Service, ConfigMap** | The basic app objects. | [26](../../CKA-Certification-Course-2025/notes/26-namespaces.md), [25](../../CKA-Certification-Course-2025/notes/25-deployments.md), [30](../../CKA-Certification-Course-2025/notes/30-services-clusterip.md), [34](../../CKA-Certification-Course-2025/notes/34-configmaps.md) |
| **Helm** | A package manager: a **chart** = templated YAML; a **release** = one installed copy with values. | [65](../../CKA-Certification-Course-2025/notes/65-helm.md) |
| **CRD** (Custom Resource Definition) | Adds new object types (e.g. `Certificate` from cert-manager). | [66](../../CKA-Certification-Course-2025/notes/66-crds-and-custom-controllers.md) |
| **GitOps** | A controller in the cluster (Argo CD, Flux) syncs YAML from Git continuously. | [68](../../CKA-Certification-Course-2025/notes/68-gitops.md) |
| **kind** | Kubernetes IN Docker: each "node" is a Docker container. Free local clusters. | [11](../../CKA-Certification-Course-2025/notes/11-lab-setup-kind-and-minikube.md) |

### The `kubernetes` provider (`hashicorp/kubernetes`, v3)

- **Typed resources**, one per kind: `kubernetes_namespace_v1`, `kubernetes_deployment_v1`, `kubernetes_service_v1`, `kubernetes_config_map_v1`, `kubernetes_secret_v1`... The `_v1` names are the current ones; the old names without `_v1` are ⚠️ deprecated in v3. Their blocks mirror the YAML: `metadata { }` and `spec { }`.
- **`kubernetes_manifest`**: takes **any** object as an HCL map (`manifest = { apiVersion = ..., kind = ..., ... }`), including custom resources. It must reach the cluster **at plan time** (to read the object's schema), and the CRD must already exist.
- Provider settings: `config_path`, `config_context`, or explicit `host` + `cluster_ca_certificate` + `token`/exec plugin (used for EKS: the capstone's optional EKS extension, topics 59–61).

### The `helm` provider (`hashicorp/helm`, v3)

- One main resource: **`helm_release`** = `helm upgrade --install`.
- v3 changed the syntax: the cluster settings are an **attribute** (`kubernetes = { ... }`), and values are lists of objects (`set = [ { name = "...", value = "..." } ]`). Old v2 examples with `kubernetes { }` blocks and repeated `set { }` blocks need converting.

### Who should own what

| Layer | Example | Best owner |
|---|---|---|
| Cloud infrastructure | VPC, EKS cluster, node groups, IAM roles | **Terraform** |
| Cluster bootstrap / platform add-ons | Namespaces, RBAC for teams, ingress controller, cert-manager, external-dns, monitoring stack, **Argo CD itself** | **Terraform** (often via `helm_release`) |
| Applications | Your services' Deployments, Services, ConfigMaps, versions | **GitOps** (Argo CD/Flux) or Helm in CI |

Why not apps in Terraform? App releases happen many times a day; Terraform runs are heavier, hold everything in state, and don't watch the cluster continuously. GitOps controllers do.

---

## Why it exists

- A cluster isn't useful until it has namespaces, RBAC (Role-Based Access Control), an ingress controller, certificates and monitoring. Doing that by hand with `kubectl` and `helm install` after each new cluster is ClickOps again.
- Terraform lets **one** plan create the cloud cluster (in one configuration) and bootstrap it (in a second), with the same review and state as all other infrastructure.
- The `kubernetes` provider also shows Terraform's **limits**: inside a cluster, controllers change objects constantly, which teaches you `ignore_changes` and ownership boundaries.

---

## Analogy

**Building a factory.**

- The **architect** (Terraform) builds the factory building (cluster), installs the big machines (ingress controller, cert-manager, monitoring), and hires the **planning office** (Argo CD).
- The **planning office** (GitOps) decides every day which products (app versions) are made, reading the order book (Git) continuously.
- If the architect also tried to schedule daily production, they'd be on site 50 times a day and keep undoing the planning office's decisions (drift fights).
- The factory's **floor managers** (Kubernetes controllers) adjust machine speed all day (HPA changing replicas). The architect must accept that (`ignore_changes`).

---

## How it works inside

### How a `kubernetes_deployment_v1` is applied

```
terraform core ──gRPC──▶ kubernetes provider
                           │ reads kubeconfig: server URL, CA, client cert/token for context kind-tf-lab
                           │ POST https://127.0.0.1:<port>/apis/apps/v1/namespaces/shop/deployments
                           ▼
                       kube-apiserver → authn/authz/admission → etcd       (your Kubernetes topic 10 trace)
                           │
                           ▼ controllers create ReplicaSet → Pods → scheduler → kubelet ...
provider (wait_for_rollout = true): polls the Deployment status until the rollout is complete
                           ← object as stored (with defaults filled in) → Terraform state
```

Terraform **only** talks to the API server. Everything after that (ReplicaSets, Pods) is the cluster's own work, and those child objects are **not** in Terraform state.

### The "same run" problem

```hcl
# ⚠️ fragile: cluster and its provider configuration in ONE configuration
resource "aws_eks_cluster" "main" { ... }

provider "kubernetes" {
  host = aws_eks_cluster.main.endpoint     # unknown until the cluster exists
  ...
}
```

Topic 13: providers are configured **early** in the run. On the first apply the endpoint is unknown; during later changes (e.g. replacing the cluster) the provider may be configured with stale or unknown values, and plans fail or try to talk to the wrong cluster. The standard fix: **two configurations** (two states): one creates the cluster and outputs its details; the other reads them (topic 38) and manages the in-cluster objects. The capstone's optional EKS extension (topics 59–61) does exactly this.

---

## Code / commands

### Step 0: a kind cluster

```bash
# kind install: Kubernetes topic 11. Then:
kind create cluster --name tf-lab
kubectl config current-context
```

```
kind-tf-lab
```

We create the cluster **outside** this configuration (the "same run" rule).

### Step 1: providers

`~/tf-lab/28-k8s/versions.tf`:

```hcl
terraform {
  required_version = ">= 1.14"
  required_providers {
    kubernetes = { source = "hashicorp/kubernetes", version = "~> 3.5" }
    helm       = { source = "hashicorp/helm", version = "~> 3.3" }
  }
}

variable "kube_context" {
  type    = string
  default = "kind-tf-lab"
}

provider "kubernetes" {
  config_path    = "~/.kube/config"
  config_context = var.kube_context
}

provider "helm" {
  kubernetes = {                      # v3 syntax: an attribute, not a block
    config_path    = "~/.kube/config"
    config_context = var.kube_context
  }
}
```

Always set `config_context` explicitly. Otherwise the provider uses whatever context is current, and one day that's production.

### Step 2: namespace, ConfigMap, Deployment, Service

`main.tf`:

```hcl
resource "kubernetes_namespace_v1" "shop" {
  metadata {
    name = "shop"
    labels = {
      "app.kubernetes.io/managed-by" = "terraform"
    }
  }
}

resource "kubernetes_config_map_v1" "web" {
  metadata {
    name      = "web-content"
    namespace = kubernetes_namespace_v1.shop.metadata[0].name
  }
  data = {
    "index.html" = "<h1>Hello from a Terraform-managed Deployment</h1>\n"
  }
}

resource "kubernetes_deployment_v1" "web" {
  metadata {
    name      = "web"
    namespace = kubernetes_namespace_v1.shop.metadata[0].name
    labels    = { app = "web" }
  }

  spec {
    replicas = 2

    selector {
      match_labels = { app = "web" }
    }

    template {
      metadata {
        labels = { app = "web" }
        annotations = {
          # roll the Pods when the content changes (ConfigMap edits don't restart Pods by themselves)
          "checksum/content" = sha256(jsonencode(kubernetes_config_map_v1.web.data))
        }
      }

      spec {
        container {
          name  = "nginx"
          image = "nginx:1.29"

          port {
            container_port = 80
          }

          volume_mount {
            name       = "content"
            mount_path = "/usr/share/nginx/html"
          }

          readiness_probe {
            http_get {
              path = "/"
              port = 80
            }
          }

          resources {
            requests = { cpu = "50m", memory = "64Mi" }
            limits   = { memory = "128Mi" }
          }
        }

        volume {
          name = "content"
          config_map {
            name = kubernetes_config_map_v1.web.metadata[0].name
          }
        }
      }
    }
  }
}

resource "kubernetes_service_v1" "web" {
  metadata {
    name      = "web"
    namespace = kubernetes_namespace_v1.shop.metadata[0].name
  }
  spec {
    selector = { app = "web" }
    port {
      port        = 80
      target_port = 80
    }
    type = "ClusterIP"
  }
}
```

- `metadata[0].name`: `metadata` is a nested block, exposed as a **list** with one element, so you index it.
- The `checksum/content` annotation is a classic trick: changing the ConfigMap changes the Pod template, which triggers a rolling update.
- Nested blocks (`container`, `port`, `volume_mount`...) mirror the YAML field names in `snake_case`.

### Step 3: a Helm chart

```hcl
resource "helm_release" "podinfo" {
  name       = "podinfo"
  repository = "https://stefanprodan.github.io/podinfo"
  chart      = "podinfo"
  version    = "6.9.2"                       # pin chart versions like provider versions
  namespace  = kubernetes_namespace_v1.shop.metadata[0].name

  set = [                                    # v3 syntax: a list of objects
    { name = "replicaCount", value = "2" },
    { name = "ui.message", value = "Installed by Terraform" },
  ]

  wait    = true                              # wait until the release's resources are ready
  timeout = 300
}
```

(Check the latest chart version with `helm search repo podinfo` after `helm repo add podinfo https://stefanprodan.github.io/podinfo`.)

### Apply and check

```bash
terraform init
terraform apply
```

```
kubernetes_namespace_v1.shop: Creating...
kubernetes_namespace_v1.shop: Creation complete after 0s [id=shop]
kubernetes_config_map_v1.web: Creating...
kubernetes_service_v1.web: Creating...
helm_release.podinfo: Creating...
kubernetes_config_map_v1.web: Creation complete after 0s [id=shop/web-content]
kubernetes_service_v1.web: Creation complete after 0s [id=shop/web]
kubernetes_deployment_v1.web: Creating...
kubernetes_deployment_v1.web: Creation complete after 8s [id=shop/web]
helm_release.podinfo: Still creating... [00m10s elapsed]
helm_release.podinfo: Creation complete after 14s [id=podinfo]

Apply complete! Resources: 5 added, 0 changed, 0 destroyed.
```

- IDs are `namespace/name` for namespaced objects.
- The Deployment took 8s: `wait_for_rollout` (default `true`) waited for Pods to be ready.

```bash
kubectl -n shop get deploy,svc,pods
kubectl -n shop port-forward svc/web 8081:80 &
curl -s localhost:8081; kill %1
helm -n shop list
```

```
<h1>Hello from a Terraform-managed Deployment</h1>
NAME     NAMESPACE  REVISION  UPDATED   STATUS    CHART           APP VERSION
podinfo  shop       1         ...       deployed  podinfo-6.9.2   6.9.2
```

`helm list` sees the Terraform-installed release: the Helm provider uses Helm's normal storage (a Secret in the namespace).

### Drift with `kubectl`

```bash
kubectl -n shop scale deploy/web --replicas=5
terraform plan
```

```
kubernetes_deployment_v1.web: Refreshing state... [id=shop/web]

Note: Objects have changed outside of Terraform
  # kubernetes_deployment_v1.web has changed
  ~ resource "kubernetes_deployment_v1" "web" {
      ~ spec {
          ~ replicas = "2" -> "5"
        }
    }
...
  # kubernetes_deployment_v1.web will be updated in-place
  ~ resource "kubernetes_deployment_v1" "web" {
      ~ spec {
          ~ replicas = "5" -> "2"
        }
    }
```

Terraform wants to scale back to 2. If an **HPA** (Horizontal Pod Autoscaler, [Kubernetes topic 48](../../CKA-Certification-Course-2025/notes/48-metrics-server-and-hpa.md)) owns the replica count, tell Terraform to leave it alone:

```hcl
resource "kubernetes_deployment_v1" "web" {
  # ...
  lifecycle {
    ignore_changes = [spec[0].replicas]
  }
}
```

### `kubernetes_manifest` for anything else

```hcl
resource "kubernetes_manifest" "quota" {
  manifest = {
    apiVersion = "v1"
    kind       = "ResourceQuota"
    metadata = {
      name      = "shop-quota"
      namespace = "shop"
    }
    spec = {
      hard = {
        pods = "20"
      }
    }
  }
}
```

The `manifest` is an HCL **object** (an attribute with `=`), the YAML converted 1:1. Use it for kinds without a typed resource, especially CRDs. Convert YAML quickly with `yamldecode(file("quota.yaml"))`. (The CRD itself must already be installed before the plan, so CRD + custom resource in one apply fails. Install CRDs in an earlier stage.)

---

## Hands-on lab

1. Create the kind cluster and apply everything. Check with `kubectl` and `helm`.
2. Change the ConfigMap content and plan: see the Deployment update because of the checksum annotation. Apply and `curl` again.
3. Drift: `kubectl scale`, plan; then add `ignore_changes` and plan again.
4. Delete the namespace with `kubectl delete ns shop` and plan. Everything inside is "deleted"; Terraform plans to recreate all of it.
5. Set `config_context = "does-not-exist"` and plan: read the error. Put it back.
6. Change `version` of the podinfo chart to an older one and plan: `helm_release` shows an in-place update (`helm upgrade`).
7. Compare: write the same Deployment as YAML and `kubectl apply -f` it in another namespace. Which shows a preview? Which notices drift? (`kubectl diff` exists, but nothing records ownership the way state does.)
8. `terraform destroy`, then `kind delete cluster --name tf-lab`.

---

## Where you'll use it

- **EKS/AKS/GKE bootstrap**: after Terraform creates a cluster, a second configuration installs the AWS Load Balancer Controller, cert-manager, external-dns, metrics-server, Prometheus/Grafana and Argo CD with `helm_release`.
- **Multi-tenant platforms**: namespaces, ResourceQuotas, LimitRanges and RBAC per team, generated with `for_each` from a list of teams.
- **Argo CD bootstrap**: Terraform installs Argo CD and creates its root "Application", then Argo CD manages all apps from Git (the hand-off point).

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `Kubernetes cluster unreachable` / `connection refused 127.0.0.1:xxxxx` | kind cluster not running, or wrong context. | `kind get clusters`, `kubectl config get-contexts`; set `config_context`. |
| `context "x" does not exist` | Context name typo. | kind contexts are `kind-<name>`. |
| Plans fail / try the wrong cluster after the cluster was replaced | Provider configured from a resource in the same configuration. | Split into two configurations (cluster, then workloads). |
| `kubernetes_manifest`: `cannot select exact GV from REST mapper` / `no matches for kind` | The CRD isn't installed yet at plan time. | Install CRDs in an earlier apply/configuration. |
| Constant diffs on `replicas` | HPA/manual scaling. | `ignore_changes = [spec[0].replicas]`. |
| `Error: cannot re-use a name that is still in use` (helm) | A Helm release with that name exists outside Terraform. | `helm uninstall`, or import the release (topic 39). |
| v2 examples fail: `Blocks of type "kubernetes" are not expected here` / `set` errors | Helm provider v3 syntax. | `kubernetes = { ... }` and `set = [ { ... } ]`. |
| `Deprecated Resource` warning for `kubernetes_deployment` | v3 deprecated the non-`_v1` names. | Rename with a `moved` block to `kubernetes_deployment_v1` (topic 40). |
| Apply hangs until timeout on a Deployment | Pods never become ready (image pull error, failing probe). | `kubectl -n shop describe pod`; [Kubernetes topic 21](../../CKA-Certification-Course-2025/notes/21-debugging-pods.md). |

---

## Quick recap

- `kubernetes` provider: typed `_v1` resources mirroring YAML, plus `kubernetes_manifest` for anything (CRDs must pre-exist). `helm` provider: `helm_release` with v3 attribute syntax.
- Both use kubeconfig; always pin `config_context`.
- Don't configure a provider from a cluster created in the same configuration: use two configurations.
- Terraform for clusters and platform add-ons; GitOps for apps. Use `ignore_changes` where controllers own fields.

### Interview questions

1. **Q: Would you deploy your microservices with Terraform?**
   A: Usually not. I'd use Terraform for the cluster and platform add-ons (ingress controller, cert-manager, monitoring, Argo CD), and GitOps or Helm in CI for app deployments, which change often and benefit from continuous reconciliation.
2. **Q: Why is it a problem to create an EKS cluster and configure the Kubernetes provider from it in the same configuration?**
   A: Providers are configured before resources are applied. On the first run the cluster's endpoint and credentials are unknown, and when the cluster is replaced the provider can be configured with stale values. Separate the cluster and the in-cluster resources into two configurations.
3. **Q: `kubernetes_deployment_v1` vs `kubernetes_manifest`?**
   A: The typed resource has a fixed schema, validation and good diffs, and doesn't need the cluster at plan time. `kubernetes_manifest` accepts any object including custom resources, but needs API access during plan and the CRD already installed.
