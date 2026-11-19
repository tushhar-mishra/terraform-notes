# 02 · The IaC Tool Landscape

> **Level:** ★ foundation
>
> **You will learn:** the two jobs IaC tools do (**provisioning** vs **configuration management**), the main tools (Terraform, OpenTofu, CloudFormation, ARM/Bicep, Pulumi, CDK, Crossplane, Ansible, Packer) and what each is good at, and why Terraform became the default answer for multi-cloud provisioning.
>
> **You need:** topic 01 (declarative, idempotent, mutable vs immutable).
>
> **Part 0 · Foundations** · Previous: [01 · What is IaC?](01-what-is-iac.md) · Next: [03 · What is Terraform?](03-what-is-terraform.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** IaC tools do one of two jobs:

- **Provisioning** = *create the infrastructure itself*: networks, VMs, databases, DNS records, Kubernetes clusters. **Terraform** is the best-known tool here.
- **Configuration management** = *set up the inside of servers that already exist*: install packages, write config files, start services. **Ansible** is the best-known tool here.

Many teams use both: Terraform builds the VM, Ansible configures what runs on it. With containers, the second job mostly moves into the **Dockerfile** and Kubernetes.

**Picture it.** Provisioning builds the house (walls, plumbing, electricity). Configuration management furnishes the rooms.

**The one thing to know.** Terraform's big strength is **one tool and one language for every platform** (AWS, Azure, GCP, Kubernetes, GitHub, Cloudflare, Datadog...) through plugins called **providers**. CloudFormation only does AWS. Bicep only does Azure.

**Try it now:** write down every tool your current or target company uses (check job ads). Sort each into "provisioning" or "configuration". You'll usually find Terraform + Ansible, or Terraform + Kubernetes + Helm.

**You've got it if you can:** answer "Terraform or Ansible?" with "different jobs: Terraform provisions, Ansible configures, and they're often used together".

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### The two jobs

| | Provisioning | Configuration management |
|---|---|---|
| **Question it answers** | "Does this infrastructure exist, with these settings?" | "Is this server set up correctly inside?" |
| **Works on** | Cloud APIs | The operating system (over SSH, Secure Shell, or an agent) |
| **Examples of what it manages** | VPC (Virtual Private Cloud, your private network in AWS), VM, load balancer, S3 bucket, IAM (Identity and Access Management) role | `nginx` package installed, `/etc/nginx/nginx.conf` content, `nginx` service running |
| **Tools** | Terraform, OpenTofu, CloudFormation, Bicep, Pulumi, CDK, Crossplane | Ansible, Chef, Puppet, Salt |
| **Typical style** | Declarative, keeps a record ("state") of what it created | Ansible: mostly declarative modules, run in order. Chef/Puppet: agent on every server |

Two more categories you'll meet:

- **Image building**: **Packer** (HashiCorp) builds VM images (AMIs) with your software already installed. That's the immutable approach from topic 01: bake the image, then Terraform launches VMs from it. A **Dockerfile** does the same job for containers ([Docker topic 14](../../Docker/notes/14-dockerfile-basics.md)).
- **Kubernetes-native IaC**: **Crossplane** runs inside a Kubernetes cluster and creates cloud resources from Kubernetes YAML, with a controller that keeps reconciling, like a Deployment does for Pods.

### The main provisioning tools

| Tool | Made by | Language | Clouds | State kept where | Notes |
|---|---|---|---|---|---|
| **Terraform** | HashiCorp (an IBM company) | **HCL** (HashiCorp Configuration Language) | Any (thousands of providers) | A state file you store (local, S3, HCP Terraform...) | The industry default. License changed in 2023 (topic 04). |
| **OpenTofu** | Linux Foundation community | HCL (same as Terraform) | Any (same providers) | Same as Terraform | Open-source fork of Terraform, near drop-in replacement (topic 04). |
| **AWS CloudFormation** | AWS | YAML or JSON | AWS only | Managed by AWS ("stacks") | No state file to manage. AWS-only. Sometimes gets new AWS features first. |
| **Azure Bicep / ARM** | Microsoft | Bicep (compiles to ARM JSON) | Azure only | Azure itself | ARM = Azure Resource Manager. |
| **Pulumi** | Pulumi | Real programming languages (TypeScript, Python, Go, C#, Java) | Any | Pulumi Cloud or self-managed | Same idea as Terraform but in a general language. Can reuse Terraform providers. |
| **AWS CDK** | AWS | TypeScript, Python, Java... | AWS only | Generates CloudFormation | CDK = Cloud Development Kit. You write code; it generates CloudFormation templates. (A Terraform version, **CDKTF**, existed but HashiCorp deprecated it in late 2025. Don't start new work on it.) |
| **Crossplane** | CNCF project | Kubernetes YAML | Any | The Kubernetes API (etcd) | Continuous reconciliation, like a Kubernetes controller. |

> **CNCF** = Cloud Native Computing Foundation, the Linux Foundation group that also hosts Kubernetes.

---

## Why it exists (why so many tools?)

Each tool was born from a different pain:

- **Chef/Puppet (≈2005–2009)**: data centres had thousands of physical servers to keep configured. Provisioning was a human racking hardware.
- **Ansible (2012)**: Chef/Puppet needed agents and a server. Ansible needed only SSH and YAML.
- **CloudFormation (2011)**: AWS gave its own customers a way to describe AWS stacks.
- **Terraform (2014)**: companies used *several* clouds and SaaS (Software as a Service) products. They wanted **one** workflow for all of them, not one tool per vendor. Terraform's plugin design (providers) made that possible.
- **Pulumi/CDK (2018–2019)**: developers wanted loops, classes and unit tests from real languages instead of a configuration language.
- **Crossplane (2018)**: Kubernetes teams wanted cloud resources to behave like Kubernetes objects, reconciled all the time.

### Why Terraform became the default

1. **Multi-provider.** One tool, one language for AWS + Cloudflare DNS + GitHub repos + Datadog monitors + Kubernetes.
2. **The plan.** Terraform shows an exact preview of every change before making it. That makes reviews in PRs possible.
3. **Huge ecosystem.** Thousands of providers and modules on the public Terraform Registry.
4. **Simple language.** HCL is easier to read than a general programming language, especially for people who review infrastructure but don't write code all day.
5. **Hiring.** Because everyone uses it, everyone hires for it.

---

## Analogy

**Building and running a hotel.**

- **Provisioning (Terraform)** = the construction company: builds floors, rooms, plumbing, wiring, lifts.
- **Image building (Packer / Dockerfile)** = a factory that makes **pre-furnished room pods** you just slot in.
- **Configuration management (Ansible)** = the housekeeping team that walks into each room and puts the right sheets, towels and TV channels in place.
- **Kubernetes** = the hotel manager who keeps assigning guests to rooms and moves them if a room breaks.
- **CloudFormation** = a construction company that only works for one hotel chain.

---

## How it works inside

How the two jobs reach their targets:

```
 PROVISIONING (Terraform)                    CONFIGURATION (Ansible)

 your .tf files                              your playbook.yml
      │                                            │
 terraform ──HTTPS API calls──▶ AWS/Azure/GCP      ansible ──SSH──▶ server 1 (runs Python
      │      "create VM", "create subnet"          │                 modules on the server:
      ▼                                            │                 dnf install nginx...)
 state file: "I created VM i-0abc..."              └──SSH──▶ server 2, server 3...
```

- Terraform never logs **into** your servers (unless you misuse provisioners, topic 52). It talks to the **control plane API** of the platform.
- Ansible logs into each server and changes files and packages inside it.

That's why they fit together: Terraform outputs the new server IPs (topic 15), and Ansible uses them as its inventory (the list of hosts to configure).

---

## Code / commands

The **same S3 bucket** in four tools, so you can see the difference in style. Don't run these; just compare.

**Terraform (HCL):**

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "acme-app-logs-2026"
}
```

**CloudFormation (YAML):**

```yaml
Resources:
  LogsBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: acme-app-logs-2026
```

**Pulumi (Python):**

```python
import pulumi_aws as aws
logs = aws.s3.Bucket("logs", bucket="acme-app-logs-2026")
```

**Ansible (YAML, using the AWS collection):**

```yaml
- name: Ensure the logs bucket exists
  amazon.aws.s3_bucket:
    name: acme-app-logs-2026
    state: present
```

What to notice:

- All four are **declarative**: "this bucket exists".
- Terraform/CloudFormation/Pulumi **remember** what they created (state). If you delete the block, they **delete the bucket**.
- Ansible **doesn't remember**. If you delete the task, the bucket just stays. To remove it, you must write `state: absent`. That's the main reason Ansible is a poor provisioning tool: it can't clean up what it no longer manages.

---

## Hands-on lab

A thinking lab (no installs yet):

1. Go to the Terraform Registry at `registry.terraform.io/browse/providers` in your browser. Count how many categories there are (cloud, networking, monitoring, version control...). Find the providers for **AWS**, **Kubernetes**, **Docker**, **GitHub** and **Cloudflare**.
2. For each of these tasks, decide **provisioning**, **configuration**, **image building** or **Kubernetes**:
   - create an EKS (Elastic Kubernetes Service) cluster
   - install `nginx` on 50 existing VMs
   - build an AMI with Java 21 preinstalled
   - create a GitHub repository with branch protection
   - roll out a new version of a web app to the cluster
   - create a Route 53 DNS record (Route 53 = AWS's DNS service)

   <details><summary>Answers</summary>

   Provisioning (Terraform): EKS cluster, GitHub repo, DNS record. Configuration (Ansible): nginx on existing VMs. Image building (Packer): the AMI. Kubernetes (Deployment/Helm/GitOps): rolling out the app version.
   </details>
3. Look at three DevOps job ads. Note which provisioning tool each asks for.

---

## Where you'll use it

- **Terraform/OpenTofu**: almost every multi-cloud or AWS shop. The skill these notes teach.
- **CloudFormation**: AWS-only teams, and some AWS services (e.g. AWS Control Tower, Service Catalog) use it under the hood. You'll read it even if you don't write it.
- **Ansible**: configuring VMs after Terraform creates them, network devices, legacy servers.
- **Packer**: golden images (pre-approved, pre-built) for VMs and autoscaling groups (sets of VMs that grow and shrink with load).
- **Helm / GitOps (Argo CD, Flux)**: deploying apps *into* Kubernetes. Terraform usually builds the cluster; GitOps deploys to it (topic 28, and [Kubernetes topic 68](../../CKA-Certification-Course-2025/notes/68-gitops.md)).

---

## When it breaks

Mistakes teams make when picking tools:

| Mistake | What goes wrong | Better |
|---|---|---|
| Using Ansible to create cloud resources | Removing a task doesn't delete the resource; no plan preview; leftovers pile up | Terraform for cloud resources, Ansible inside servers |
| Using Terraform to configure servers over SSH (provisioners) | Fragile, not idempotent, failures leave half-built servers | Packer images, cloud-init, or Ansible (topic 52) |
| Using Terraform to deploy every app release into Kubernetes | Slow, state holds large app manifests, fights with GitOps | Terraform for the cluster and its add-ons; Helm/GitOps for apps |
| Mixing two provisioning tools on the same resource | Both think they own it; each "fixes" the other's changes forever | One owner per resource |
| Clicking in the console "just this once" | Drift (topic 37) | Change the code, even for urgent fixes, or bring it back into code right after |

---

## Quick recap

- **Provisioning** (Terraform, OpenTofu, CloudFormation, Pulumi, Crossplane) creates infrastructure through cloud APIs.
- **Configuration management** (Ansible, Chef, Puppet) sets up the inside of servers.
- **Packer / Dockerfiles** bake images, the immutable approach.
- Terraform wins on **multi-provider support, the plan preview, ecosystem and readability**.
- Tools that keep **state** can delete what you removed from the code. Ansible can't.

### Interview questions

1. **Q: Terraform vs Ansible?**
   A: Terraform provisions infrastructure through APIs and keeps state, so it can plan, update and destroy. Ansible configures the inside of servers over SSH. They complement each other: Terraform creates the VMs, Ansible configures them.
2. **Q: Terraform vs CloudFormation?**
   A: CloudFormation is AWS-only and AWS manages its state. Terraform works across hundreds of platforms with one language and workflow, but you manage the state file yourself.
3. **Q: Why might a team choose Pulumi over Terraform?**
   A: To use a general-purpose language (TypeScript, Python, Go) with loops, classes, packages and unit tests. The trade-off: harder for non-programmers to review, and a smaller hiring pool.
