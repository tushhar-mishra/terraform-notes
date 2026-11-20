# 59 · Capstone Part 1: Foundation

> **Level:** ★★★ advanced
>
> **You will learn:** to build a realistic Terraform codebase from scratch, combining Parts 1–8: a **bootstrap** (state bucket + CI roles), two reusable **modules** (`network` wrapping the public VPC module; `web` = **ALB** + **Auto Scaling group** + **launch template** + security groups + SSM role), and the **dev** environment as two **separate states** (`network`, `web`) that share values through **SSM parameters**. At the end you'll have a load-balanced, self-healing web tier in dev, built only from code.
>
> **You need:** all previous topics, especially 32–33, 35, 38, 44–47, 49, 51, 53, 55.
>
> **Part 10 · Running it in production** · Previous: [58 · Terraform in Jenkins](58-jenkins.md) · Next: [60 · Capstone part 2: environments and CI/CD](60-capstone-2-environments-and-cicd.md) · [Back to INDEX](../INDEX.md)
>
> 💰 **Cost:** an Application Load Balancer (≈ USD 0.02–0.03/hour + usage) and two `t3.micro` instances with public IPv4 addresses. A few hours of work costs well under USD 1. **Destroy at the end of each session** (part 3 has the full teardown).

---

## ⚡ Core: the 10-minute version

**What you'll build:**

```
                        Internet
                           │ :80
                 ┌─────────▼──────────┐
                 │  ALB (public subnets, 2 AZs)  ── SG alb: 80 from 0.0.0.0/0
                 └─────────┬──────────┘
                           │ :80 (only from the ALB's SG)
          ┌────────────────┴────────────────┐
     EC2 (AZ a)                         EC2 (AZ b)        ← Auto Scaling group, min 2
     nginx via user_data                nginx              launch template: AL2023, IMDSv2,
     IAM role: SSM                                         encrypted gp3, SSM role
```

**Repository:**

```
acme-infra/
├── bootstrap/                 # state bucket + GitHub OIDC roles (topics 35, 57), local state
├── modules/
│   ├── network/               # wraps terraform-aws-modules/vpc + publishes SSM parameters
│   └── web/                   # ALB + target group + ASG + launch template + SGs + IAM
├── envs/
│   └── dev/
│       ├── network/           # state: dev/network
│       └── web/               # state: dev/web (reads network via SSM)
├── policy/                    # Conftest policies (topic 54)
└── .github/workflows/         # part 2
```

**The one thing to know.** Every decision here maps to a topic you've learned: thin roots (47), separate states (38, 49), secure modules (44), no SSH (33), versioned backend (35), tests and scans (53–55).

**Try it now:** follow the steps; apply `dev/network`, then `dev/web`, then `curl` the ALB URL.

**You've got it if you can:** explain why `network` and `web` are separate states, and how `web` finds the VPC.

> 📖 **That's the core.** Everything below is the full build.

---

## What it is

### Design decisions (and the topic behind each)

| Decision | Why | Topic |
|---|---|---|
| Separate `network` and `web` states | Different change rates; a web change can't touch the VPC | 38, 47, 49 |
| Share via **SSM parameters**, not remote state | Consumers don't need read access to network state | 38 |
| Wrap `terraform-aws-modules/vpc` | Tested community code + our defaults (no NAT in dev) | 46 |
| `web` takes `vpc_id`/`subnet_ids` as inputs | Dependency inversion: the module stays reusable | 47 |
| ALB + ASG + launch template | Self-healing, rolling updates, no single server | 24 (replacements), 33 |
| Instances in **public** subnets but only reachable from the ALB | Avoids NAT cost in the lab; SG-to-SG rule keeps them private in practice | 32 |
| No SSH; SSM role | No open port 22, no keys | 33 |
| `default_tags` + `allowed_account_ids` | Cost tracking; wrong-account protection | 30 |
| S3 backend with lock file, versioned bucket | Teams, CI, recovery | 35, 36, 42 |

### Two new AWS pieces

- **Application Load Balancer (ALB)**: receives HTTP(S) traffic on public subnets in ≥ 2 AZs and forwards it to a **target group** of healthy instances. It health-checks each instance (`GET /` → 200).
- **Auto Scaling group (ASG)**: keeps N instances running from a **launch template** (AMI, type, `user_data`, SG, role). If an instance fails its health check, the ASG replaces it. An **instance refresh** rolls instances to a new launch template version gradually.

---

## Why it exists

Single-topic labs teach one idea at a time. Real work is wiring many ideas together into a codebase other people can run, review and change safely. This capstone gives you a portfolio-quality project you can explain end to end in interviews.

---

## Analogy

Building a **small chain of shops**: a land plan and roads (network) laid once by the council; each shop (web tier) built from a standard franchise kit (modules), with a front door (ALB) that sends customers to any open counter (instances), and a manager (ASG) who replaces a counter clerk who goes home sick.

---

## How it works inside

```
apply envs/dev/network  → VPC, subnets, IGW, routes (via the public module)
                        → SSM: /acme/dev/network/vpc_id, public_subnet_ids, private_subnet_ids
apply envs/dev/web      → read SSM params (data sources)
                        → SGs (alb, app) + rules → IAM role/profile → launch template
                        → ALB + target group + listener → ASG (desired 2) attached to the TG
                        → ASG launches 2 instances → cloud-init installs nginx
                        → ALB health checks pass → targets "healthy" → traffic flows
```

---

## Code / commands

Create the repo: `mkdir -p ~/acme-infra && cd ~/acme-infra && git init -b main`, and add the `.gitignore` from topic 05.

### 1. Bootstrap

Copy topic 35's bootstrap (state bucket) and topic 57's OIDC roles into `bootstrap/` (one configuration, local state). Name the bucket `acme-tfstate-<account-id>`. Apply once:

```bash
cd ~/acme-infra/bootstrap && terraform init && terraform apply
```

Keep `bootstrap/terraform.tfstate` safe (or migrate it into the bucket under `bootstrap/terraform.tfstate`, topic 35).

### 2. Module: `modules/network`

`modules/network/versions.tf`:

```hcl
terraform {
  required_version = ">= 1.9"
  required_providers {
    aws = { source = "hashicorp/aws", version = ">= 6.0" }
  }
}
```

`modules/network/variables.tf`:

```hcl
variable "name"        { type = string }
variable "environment" { type = string }

variable "cidr" {
  type = string
  validation {
    condition     = can(cidrhost(var.cidr, 0))
    error_message = "cidr must be a valid CIDR block."
  }
}

variable "az_count" {
  type    = number
  default = 2
  validation {
    condition     = var.az_count >= 2 && var.az_count <= 3
    error_message = "Use 2 or 3 AZs."
  }
}

variable "enable_nat_gateway" {
  type        = bool
  description = "💰 NAT for private subnets. Off in dev."
  default     = false
}
```

`modules/network/main.tf`:

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  azs = slice(data.aws_availability_zones.available.names, 0, var.az_count)
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 6.0"

  name = "${var.name}-${var.environment}"
  cidr = var.cidr
  azs  = local.azs

  public_subnets  = [for i, _ in local.azs : cidrsubnet(var.cidr, 8, i)]
  private_subnets = [for i, _ in local.azs : cidrsubnet(var.cidr, 8, i + 10)]

  enable_nat_gateway     = var.enable_nat_gateway
  one_nat_gateway_per_az = var.enable_nat_gateway
  enable_dns_hostnames   = true
}

# Published contract for other states (topic 38)
locals {
  prefix = "/${var.name}/${var.environment}/network"
}

resource "aws_ssm_parameter" "vpc_id" {
  name  = "${local.prefix}/vpc_id"
  type  = "String"
  value = module.vpc.vpc_id
}

resource "aws_ssm_parameter" "public_subnet_ids" {
  name  = "${local.prefix}/public_subnet_ids"
  type  = "String"
  value = jsonencode(module.vpc.public_subnets)
}

resource "aws_ssm_parameter" "private_subnet_ids" {
  name  = "${local.prefix}/private_subnet_ids"
  type  = "String"
  value = jsonencode(module.vpc.private_subnets)
}
```

`modules/network/outputs.tf`:

```hcl
output "vpc_id"             { value = module.vpc.vpc_id }
output "public_subnet_ids"  { value = module.vpc.public_subnets }
output "private_subnet_ids" { value = module.vpc.private_subnets }
output "ssm_prefix"         { value = local.prefix }
```

### 3. Module: `modules/web`

`modules/web/variables.tf`:

```hcl
variable "name"        { type = string }
variable "environment" { type = string }

variable "network" {
  description = "Where to deploy (dependency inversion: passed in, not looked up)."
  type = object({
    vpc_id            = string
    public_subnet_ids = list(string)
  })
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "capacity" {
  type = object({
    min     = number
    max     = number
    desired = number
  })
  default = { min = 2, max = 4, desired = 2 }
  validation {
    condition     = var.capacity.min <= var.capacity.desired && var.capacity.desired <= var.capacity.max
    error_message = "Need min <= desired <= max."
  }
}

variable "page_message" {
  type    = string
  default = "Hello from acme-shop"
}
```

`modules/web/security.tf`:

```hcl
resource "aws_security_group" "alb" {
  name        = "${var.name}-${var.environment}-alb"
  description = "ALB: HTTP from the internet"
  vpc_id      = var.network.vpc_id
}

resource "aws_vpc_security_group_ingress_rule" "alb_http" {
  #trivy:ignore:AVD-AWS-0107 Public load balancer: HTTP from anywhere is intended.
  security_group_id = aws_security_group.alb.id
  ip_protocol       = "tcp"
  from_port         = 80
  to_port           = 80
  cidr_ipv4         = "0.0.0.0/0"
}

resource "aws_vpc_security_group_egress_rule" "alb_to_app" {
  security_group_id            = aws_security_group.alb.id
  ip_protocol                  = "tcp"
  from_port                    = 80
  to_port                      = 80
  referenced_security_group_id = aws_security_group.app.id
}

resource "aws_security_group" "app" {
  name        = "${var.name}-${var.environment}-app"
  description = "App instances: HTTP only from the ALB"
  vpc_id      = var.network.vpc_id
}

resource "aws_vpc_security_group_ingress_rule" "app_from_alb" {
  security_group_id            = aws_security_group.app.id
  ip_protocol                  = "tcp"
  from_port                    = 80
  to_port                      = 80
  referenced_security_group_id = aws_security_group.alb.id
}

resource "aws_vpc_security_group_egress_rule" "app_https" {
  #trivy:ignore:AVD-AWS-0104 Instances need HTTPS to package repos and SSM endpoints (no NAT/endpoints in dev).
  security_group_id = aws_security_group.app.id
  ip_protocol       = "tcp"
  from_port         = 443
  to_port           = 443
  cidr_ipv4         = "0.0.0.0/0"
}

resource "aws_vpc_security_group_egress_rule" "app_http" {
  #trivy:ignore:AVD-AWS-0104 Some package mirrors use HTTP redirects.
  security_group_id = aws_security_group.app.id
  ip_protocol       = "tcp"
  from_port         = 80
  to_port           = 80
  cidr_ipv4         = "0.0.0.0/0"
}
```

- Group-to-group rules: instances accept port 80 **only from the ALB's security group**, even though they sit in public subnets.
- Egress is limited to 80/443 instead of "all", with documented suppressions (topic 53).

`modules/web/iam.tf`: the same SSM role + instance profile as topic 33 (`aws_iam_role` trusting `ec2.amazonaws.com`, attach `AmazonSSMManagedInstanceCore`, `aws_iam_instance_profile`), named `"${var.name}-${var.environment}-web"`.

`modules/web/templates/user-data.sh.tftpl`: topic 33's script (nginx + IMDSv2 page), plus `${environment}` in the page.

`modules/web/compute.tf`:

```hcl
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }
}

resource "aws_launch_template" "web" {
  name_prefix   = "${var.name}-${var.environment}-web-"     # prefix: allows create_before_destroy
  # ⚠️ Floating AMI: every AL2023 release changes this ID, so the NEXT unrelated
  # apply creates a new launch template version and (with instance_refresh)
  # replaces the whole fleet without anyone asking for it. Fine for this lab;
  # in prod pin it: coalesce(var.ami_id, data.aws_ami.al2023.id) (topic 61),
  # and roll new AMIs deliberately through a PR.
  image_id      = data.aws_ami.al2023.id
  instance_type = var.instance_type

  iam_instance_profile {
    arn = aws_iam_instance_profile.web.arn
  }

  network_interfaces {
    associate_public_ip_address = true                      # public subnets, no NAT (lab)
    security_groups             = [aws_security_group.app.id]
  }

  metadata_options {
    http_tokens = "required"
  }

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_type = "gp3"
      volume_size = 8
      encrypted   = true
    }
  }

  user_data = base64encode(templatefile("${path.module}/templates/user-data.sh.tftpl", {
    message     = var.page_message
    environment = var.environment
  }))

  tag_specifications {
    resource_type = "instance"
    tags          = { Name = "${var.name}-${var.environment}-web" }
  }

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_lb" "web" {
  name               = "${var.name}-${var.environment}-web"
  load_balancer_type = "application"
  subnets            = var.network.public_subnet_ids
  security_groups    = [aws_security_group.alb.id]
  drop_invalid_header_fields = true
}

resource "aws_lb_target_group" "web" {
  name_prefix = "web-"                                   # max 6 chars; allows CBD
  port        = 80
  protocol    = "HTTP"
  vpc_id      = var.network.vpc_id

  health_check {
    path                = "/"
    healthy_threshold   = 2
    unhealthy_threshold = 2
    interval            = 15
    matcher             = "200"
  }

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.web.arn
  port              = 80
  protocol          = "HTTP"
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}

resource "aws_autoscaling_group" "web" {
  name_prefix         = "${var.name}-${var.environment}-web-"
  vpc_zone_identifier = var.network.public_subnet_ids
  target_group_arns   = [aws_lb_target_group.web.arn]
  health_check_type   = "ELB"                            # replace instances the ALB marks unhealthy
  health_check_grace_period = 120

  min_size         = var.capacity.min
  max_size         = var.capacity.max
  desired_capacity = var.capacity.desired

  launch_template {
    id      = aws_launch_template.web.id
    version = aws_launch_template.web.latest_version
  }

  instance_refresh {                                     # roll instances when the template changes
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 50
    }
  }

  tag {
    key                 = "Name"
    value               = "${var.name}-${var.environment}-web"
    propagate_at_launch = true
  }

  lifecycle {
    ignore_changes = [desired_capacity]                  # scaling policies may change it (topic 24)
  }
}
```

`modules/web/outputs.tf`:

```hcl
output "url"            { value = "http://${aws_lb.web.dns_name}/" }
output "asg_name"       { value = aws_autoscaling_group.web.name }
output "app_sg_id"      { value = aws_security_group.app.id }
output "target_group"   { value = aws_lb_target_group.web.arn }
```

`modules/web/versions.tf`: same as the network module.

### 4. Environment roots: `envs/dev/network` and `envs/dev/web`

`envs/dev/network/main.tf`:

```hcl
terraform {
  required_version = ">= 1.14"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.0" }
  }
  backend "s3" {
    bucket       = "acme-tfstate-123456789012"
    key          = "dev/network/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}

provider "aws" {
  region              = "ap-south-1"
  allowed_account_ids = ["123456789012"]
  default_tags {
    tags = { project = "acme-shop", environment = "dev", stack = "network", owner = "platform", managed_by = "terraform" }
  }
}

module "network" {
  source      = "../../../modules/network"
  name        = "acme"
  environment = "dev"
  cidr        = "10.30.0.0/16"
}

output "vpc_id" { value = module.network.vpc_id }
```

`envs/dev/web/main.tf`:

```hcl
terraform {
  required_version = ">= 1.14"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.0" }
  }
  backend "s3" {
    bucket       = "acme-tfstate-123456789012"
    key          = "dev/web/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}

provider "aws" {
  region              = "ap-south-1"
  allowed_account_ids = ["123456789012"]
  default_tags {
    tags = { project = "acme-shop", environment = "dev", stack = "web", owner = "platform", managed_by = "terraform" }
  }
}

variable "page_message" {
  type    = string
  default = "Hello from acme-shop (dev)"
}

data "aws_ssm_parameter" "vpc_id" {
  name = "/acme/dev/network/vpc_id"
}

data "aws_ssm_parameter" "public_subnet_ids" {
  name = "/acme/dev/network/public_subnet_ids"
}

module "web" {
  source      = "../../../modules/web"
  name        = "acme"
  environment = "dev"

  network = {
    vpc_id            = nonsensitive(data.aws_ssm_parameter.vpc_id.value)
    public_subnet_ids = jsondecode(nonsensitive(data.aws_ssm_parameter.public_subnet_ids.value))
  }

  capacity     = { min = 2, max = 3, desired = 2 }
  page_message = var.page_message
}

output "url" { value = module.web.url }

check "site_up" {
  data "http" "home" {
    url = module.web.url
  }
  assert {
    condition     = data.http.home.status_code == 200
    error_message = "ALB returned ${data.http.home.status_code} (instances may still be starting)."
  }
}
```

(Add `http = { source = "hashicorp/http", version = "~> 3.5" }` to `required_providers` for the check.)

### 5. Apply dev

```bash
export AWS_PROFILE=tf-lab && aws sso login
cd ~/acme-infra/envs/dev/network && terraform init && terraform apply
cd ../web && terraform init && terraform apply
```

```
Plan: 17 to add, 0 to change, 0 to destroy.            # web (count varies)
...
aws_lb.web: Still creating... [02m10s elapsed]
aws_lb.web: Creation complete after 2m31s [id=arn:aws:elasticloadbalancing:...]
...
aws_autoscaling_group.web: Creation complete after 47s [id=acme-dev-web-2026111912...]
╷
│ Warning: Check block assertion failed   ← expected on first apply: instances still booting
╵
Outputs:

url = "http://acme-dev-web-1234567890.ap-south-1.elb.amazonaws.com/"
```

Wait 2–3 minutes:

```bash
URL=$(terraform output -raw url)
for i in 1 2 3 4; do curl -s "$URL" | grep -o 'Instance: [^ ]*'; done
terraform plan          # check passes now; "No changes"
```

```
Instance: i-0a1b2c3d4e5f60001
Instance: i-0f9e8d7c6b5a40002
Instance: i-0a1b2c3d4e5f60001
Instance: i-0f9e8d7c6b5a40002
```

Two instances behind the ALB, in two AZs.

### 6. Prove it heals

```bash
aws ec2 terminate-instances --instance-ids i-0a1b2c3d4e5f60001
watch -n 10 "aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names $(terraform output -raw asg_name 2>/dev/null || echo) --query 'AutoScalingGroups[0].Instances[].[InstanceId,LifecycleState,HealthStatus]' --output text"
```

(If `asg_name` isn't a root output, add `output "asg_name" { value = module.web.asg_name }`.) The ASG launches a replacement; the site keeps answering from the other instance. Terraform state doesn't change: individual instances are owned by the ASG, not by Terraform.

### 7. Quality gates (run them now)

```bash
cd ~/acme-infra
terraform fmt -check -recursive
tflint --recursive
trivy config --severity HIGH,CRITICAL .
```

Add a mock-provider test for `modules/web` (topic 55): assert the ALB is `application`, the app SG's ingress only references the ALB SG, and `capacity` validation rejects `min > desired`.

---

## Hands-on lab

1. Build everything above; get the URL working.
2. Terminate an instance; watch the ASG heal.
3. Change `page_message` and apply: the launch template gets a new version and **instance refresh** rolls both instances, keeping at least 50% healthy. `curl` in a loop during the refresh: no downtime.
4. Run the quality gates and fix or document every HIGH/CRITICAL finding.
5. Commit everything (`git add . && git commit -m "capstone: dev foundation"`).
6. **End of session:** `terraform destroy` in `envs/dev/web`, then `envs/dev/network` (reverse order). Keep `bootstrap`.

---

## Where you'll use it

This is a miniature of a real company's AWS web tier. Interviewers love "walk me through a Terraform project you built": you can now explain every file and decision here, and why.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `ParameterNotFound` in `dev/web` | `dev/network` not applied, or names differ | Apply network first; check `/acme/dev/network/...` |
| ALB returns `502`/`503` | Targets not healthy yet, or app SG doesn't allow the ALB | Wait; check target health in the console; check SG rules |
| Targets unhealthy forever | `user_data` failed (no egress, typo) | SSM into an instance; `/var/log/cloud-init-output.log` (topic 33) |
| `DuplicateTargetGroupName` / name length errors | Fixed names with `create_before_destroy` | `name_prefix` (target group prefix ≤ 6 chars) |
| `DependencyViolation` destroying network | `web` still deployed | Destroy `web` first |
| Plan shows `desired_capacity` drift | Scaling changed it | It's in `ignore_changes` by design |

---

## Quick recap

- Bootstrap (state + CI roles) → modules (`network` wrapping the public VPC module; `web` with ALB + ASG + launch template + SGs + SSM role) → environment roots with **separate states** sharing values via **SSM**.
- Instances only accept traffic from the ALB's SG; no SSH; IMDSv2; encrypted disks.
- ASG heals instances; instance refresh rolls updates without downtime.
- Destroy in reverse order: `web`, then `network`.

### Interview questions

1. **Q: Walk me through how your web tier is built with Terraform.**
   A: A network state creates the VPC with a wrapped community module and publishes IDs to SSM. A web state reads them and calls a web module: an ALB with a target group and listener, an Auto Scaling group from a launch template (AL2023, IMDSv2, encrypted disk, SSM instance role, nginx via user_data), and security groups where instances only accept traffic from the ALB's group. Instance refresh rolls changes.
2. **Q: Why do your instances not appear in Terraform state?**
   A: The ASG owns them. Terraform manages the ASG and launch template; the ASG creates, replaces and scales instances. That's also why `desired_capacity` is in `ignore_changes`.
3. **Q: How does a change to `user_data` reach running servers without downtime?**
   A: It creates a new launch template version; the ASG's instance refresh replaces instances in rolling batches while keeping a minimum healthy percentage behind the load balancer.
