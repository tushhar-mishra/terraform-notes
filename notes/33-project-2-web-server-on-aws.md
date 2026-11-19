# 33 · Project 2: A Web Server on AWS

> **Level:** ★★ intermediate
>
> **You will learn:** to put Part 4 together: your topic 32 VPC + a security group with **no SSH port** + an **IAM role and instance profile** for **SSM Session Manager** (shell access without SSH keys) + an EC2 instance configured at boot with **`user_data`** rendered by `templatefile` (cloud-init installs nginx and writes a page) + `user_data_replace_on_change` + a **`check` block** that verifies the site answers. Then you'll change it, break it, debug it from the console output and the cloud-init log, and destroy it.
>
> **You need:** topics 18 (`templatefile`), 23 (cloud-init), 24 (replacements), 25 (`check`), 29–32.
>
> **Part 4 · AWS** · Previous: [32 · AWS networking with Terraform](32-aws-networking.md) · Next: [34 · State deep dive](34-state-deep-dive.md) · [Back to INDEX](../INDEX.md)
>
> 💰 **Cost:** one Free-Tier-eligible `t3.micro` + one public IPv4 address (≈ USD 0.005/h). Everything else is free. **Destroy at the end.**

---

## ⚡ Core: the 10-minute version

**The idea.** A real, reachable web server, built entirely from code:

```
internet ──▶ IGW ──▶ public subnet ──▶ EC2 (nginx, port 80 only)
                                         │ IAM role: AmazonSSMManagedInstanceCore
you ── aws ssm start-session ────────────┘ (no SSH, no keys, no port 22)
```

The key new pieces:

```hcl
resource "aws_instance" "web" {
  ami                         = data.aws_ami.al2023.id
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.public["ap-south-1a"].id
  vpc_security_group_ids      = [aws_security_group.web.id]
  iam_instance_profile        = aws_iam_instance_profile.web.name
  user_data                   = templatefile("${path.module}/user-data.sh.tftpl", { ... })
  user_data_replace_on_change = true      # new boot script → new instance
}
```

**Picture it.** Ordering a pre-furnished flat: the building (VPC) exists; you rent one flat (instance) with a door that only opens for guests on the front (port 80); the caretaker has a master key card (SSM role) so nobody needs a physical key (SSH); and the move-in checklist (`user_data`) is done automatically on day one.

**The one thing to know.** `user_data` runs **once**, at **first boot**. Changing it does nothing to a running instance unless the instance is **replaced**, which `user_data_replace_on_change = true` makes explicit.

**Try it now:** build it, `curl` the output URL, open a shell with `aws ssm start-session`, then destroy.

**You've got it if you can:** explain why there's no port 22 and no key pair, and what happens when you edit the boot script.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Project layout

```
~/tf-lab/33-web/
├── versions.tf        # aws ~> 6.0, required_version, provider with default_tags
├── variables.tf
├── network.tf         # copied from topic 32, unchanged
├── security.tf        # web SG: HTTP in, all out. No SSH.
├── iam.tf             # role + policy attachment + instance profile for SSM
├── compute.tf         # AMI lookup + instance
├── user-data.sh.tftpl # boot script template
├── checks.tf          # check block: is the site up?
└── outputs.tf
```

### New concepts

| Concept | What it is |
|---|---|
| **`user_data`** | Text passed to the instance at launch. **cloud-init** (topic 23) reads it at first boot; a script starting with `#!` is run as root. Limit 16 KB. |
| **`user_data_replace_on_change`** | AWS provider option: when `user_data` changes, **replace** the instance (instead of an in-place update that would have no effect until a reboot, and then still wouldn't re-run the script). |
| **IAM role for EC2** | A role whose **trust policy** lets the EC2 service assume it. The instance gets temporary credentials through IMDS (topic 31). |
| **Instance profile** | The container that attaches a role to an instance. EC2 takes an instance profile, not a role directly. |
| **SSM Session Manager** | AWS Systems Manager gives you a shell in the instance through the AWS API: no open port, no SSH key, every session logged. The SSM agent is preinstalled on Amazon Linux 2023; it needs the `AmazonSSMManagedInstanceCore` policy and outbound HTTPS. |

---

## Why it exists

This is the classic "deploy a web server" task, done the **production way**:

- no SSH port open to the internet (the most-scanned port on earth),
- no long-lived SSH keys to rotate and lose,
- the server fully configured by code at boot (immutable: change the script → new server),
- permissions given to the instance by a role, not by keys copied onto it,
- a health check that tells you whether it actually works.

---

## Analogy

**Renting a serviced flat.**

- The **building** (VPC, subnets, gate) is already there (topic 32).
- **Security group** = the flat's door: open for guests on the front door (port 80); the back door (22) is bricked up.
- **IAM role + SSM** = the caretaker's key card: the building system lets authorised staff in, and logs every entry. No physical keys to copy.
- **`user_data`** = the move-in checklist the staff completes **once**, the day you move in. Changing the checklist later doesn't redo the move-in, unless you move into a **new** flat (`user_data_replace_on_change`).
- **`check` block** = ringing the doorbell after move-in to make sure someone answers.

---

## How it works inside

From `apply` to a working page:

```
1. Terraform: IAM role (trust: ec2.amazonaws.com) → attach SSM policy → instance profile
2. Terraform: ec2:RunInstances(ami, subnet, sg, instance profile, user_data=base64(script))
   → waits for "running" → done (Terraform does NOT wait for your script!)
3. Instance boots → cloud-init reads user_data from IMDS → runs the script as root:
      dnf install nginx → write index.html (using IMDSv2 to read its own ID/AZ) → start nginx
   (≈ 30–90 s after "running")
4. SSM agent starts → uses the role's credentials → registers with Systems Manager
5. check block runs at the end of apply → curl the public IP → probably NOT ready yet → warning
   next plan → check passes
```

Important: **"Apply complete" means the VM is running, not that your app works.** The `check` block (or a smoke test in CI) closes that gap.

---

## Code / commands

### `variables.tf`

```hcl
variable "name" {
  type    = string
  default = "tf-course"
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "az_count" {
  type    = number
  default = 2
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "page_message" {
  type    = string
  default = "Hello from Terraform on AWS"
}
```

### `network.tf`

Copy `network.tf` from topic 32 unchanged (VPC, subnets, IGW, routes, S3 endpoint, `data "aws_region"`).

### `security.tf`

```hcl
resource "aws_security_group" "web" {
  name        = "${var.name}-web"
  description = "Web server: HTTP in, everything out. No SSH (we use SSM)."
  vpc_id      = aws_vpc.main.id
  tags        = { Name = "${var.name}-web" }
}

resource "aws_vpc_security_group_ingress_rule" "http" {
  security_group_id = aws_security_group.web.id
  ip_protocol       = "tcp"
  from_port         = 80
  to_port           = 80
  cidr_ipv4         = "0.0.0.0/0"
}

resource "aws_vpc_security_group_egress_rule" "all" {
  security_group_id = aws_security_group.web.id
  ip_protocol       = "-1"
  cidr_ipv4         = "0.0.0.0/0"     # dnf repos + SSM endpoints over HTTPS
}
```

### `iam.tf`

```hcl
data "aws_iam_policy_document" "ec2_trust" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]     # only the EC2 service may assume this role
    }
  }
}

resource "aws_iam_role" "web" {
  name               = "${var.name}-web"
  assume_role_policy = data.aws_iam_policy_document.ec2_trust.json
}

resource "aws_iam_role_policy_attachment" "ssm" {
  role       = aws_iam_role.web.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"   # AWS-managed policy
}

resource "aws_iam_instance_profile" "web" {
  name = "${var.name}-web"
  role = aws_iam_role.web.name
}
```

- `data "aws_iam_policy_document"` builds the trust-policy JSON in HCL (topic 19); `.json` gives the string.
- The **trust policy** says *who* may assume the role (EC2). The **attached policy** says *what* the role may do (SSM).

### `user-data.sh.tftpl`

```bash
#!/bin/bash
set -euo pipefail
exec > >(tee -a /var/log/user-data.log) 2>&1     # keep our own log too

dnf -y install nginx

# IMDSv2: get a session token, then read instance details
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
md() { curl -s -H "X-aws-ec2-metadata-token: $TOKEN" "http://169.254.169.254/latest/meta-data/$1"; }

cat > /usr/share/nginx/html/index.html <<HTML
<h1>${message}</h1>
<p>Instance: $(md instance-id) · AZ: $(md placement/availability-zone) · Env: ${environment}</p>
HTML

systemctl enable --now nginx
```

- `${message}` and `${environment}` are **Terraform** template variables, filled in by `templatefile`.
- `$(md instance-id)` and `$TOKEN` are **shell**: a `$` followed by `(` or a letter is left alone by Terraform. Only `${...}` is interpolated. (If the script needed a literal shell `${VAR}`, you'd write `$${VAR}`.)
- `set -euo pipefail` and the log line come from [Linux topic 045](../../linux-mastery-rhel/notes/045-robust-scripts.md).

### `compute.tf`

```hcl
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.al2023.id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.public[local.azs[0]].id
  vpc_security_group_ids = [aws_security_group.web.id]
  iam_instance_profile   = aws_iam_instance_profile.web.name

  user_data = templatefile("${path.module}/user-data.sh.tftpl", {
    message     = var.page_message
    environment = "lab"
  })
  user_data_replace_on_change = true

  metadata_options {
    http_tokens = "required"
  }

  root_block_device {
    volume_type = "gp3"
    volume_size = 8
    encrypted   = true
  }

  tags = { Name = "${var.name}-web" }

  lifecycle {
    ignore_changes = [ami]          # don't replace the server every time AWS publishes a new AMI
  }
}
```

### `checks.tf` and `outputs.tf`

```hcl
check "web_responds" {
  data "http" "home" {
    url = "http://${aws_instance.web.public_ip}/"
    retry {
      attempts = 3
    }
  }
  assert {
    condition     = data.http.home.status_code == 200
    error_message = "Web server at ${aws_instance.web.public_ip} returned ${data.http.home.status_code}."
  }
}
```

(Add `http = { source = "hashicorp/http", version = "~> 3.5" }` to `required_providers`.)

```hcl
output "url" {
  value = "http://${aws_instance.web.public_ip}/"
}

output "instance_id" {
  value = aws_instance.web.id
}

output "ssm_command" {
  value = "aws ssm start-session --target ${aws_instance.web.id}"
}
```

### Apply

```bash
export AWS_PROFILE=tf-lab
terraform init
terraform apply
```

```
Plan: 21 to add, 0 to change, 0 to destroy.
...
aws_iam_role.web: Creation complete after 1s [id=tf-course-web]
aws_iam_instance_profile.web: Creation complete after 7s [id=tf-course-web]
aws_instance.web: Creating...
aws_instance.web: Still creating... [00m10s elapsed]
aws_instance.web: Creation complete after 14s [id=i-0123456789abcdef0]
╷
│ Warning: Check block assertion failed
│
│   on checks.tf line 9, in check "web_responds":
│    9:     condition     = data.http.home.status_code == 200
│ ...
│ Error making request: GET http://13.234.56.78/ giving up after 4 attempt(s): connection refused
╵

Apply complete! Resources: 21 added, 0 changed, 0 destroyed.

Outputs:

instance_id = "i-0123456789abcdef0"
ssm_command = "aws ssm start-session --target i-0123456789abcdef0"
url = "http://13.234.56.78/"
```

- The **warning is expected** on the first apply: the instance is running but cloud-init hasn't finished installing nginx.
- Wait a minute, then:

```bash
curl -s "$(terraform output -raw url)"
terraform plan        # the check runs again → no warning now
```

```
<h1>Hello from Terraform on AWS</h1>
<p>Instance: i-0123456789abcdef0 · AZ: ap-south-1a · Env: lab</p>
```

### Get a shell without SSH

Install the **Session Manager plugin** for the AWS CLI once (AWS docs: "Install the Session Manager plugin", there's an Ubuntu `.deb`), then:

```bash
$(terraform output -raw ssm_command)
```

```
Starting session with SessionId: you-0a1b2c3d4e5f6
sh-5.2$ sudo tail -5 /var/log/cloud-init-output.log
...
Cloud-init v. 22.2.2 finished at ... Datasource DataSourceEc2.  Up 48.31 seconds
sh-5.2$ systemctl is-active nginx
active
sh-5.2$ exit
```

`/var/log/cloud-init-output.log` is **the** place to debug `user_data` scripts.

### Change the boot script → replacement

Change `page_message`:

```bash
terraform plan -var='page_message=Version 2'
```

```
  # aws_instance.web must be replaced
-/+ resource "aws_instance" "web" {
      ~ id                          = "i-0123456789abcdef0" -> (known after apply)
      ~ public_ip                   = "13.234.56.78" -> (known after apply)
      ~ user_data                   = "0f3e1a..." -> "9b2c7d..." # forces replacement
        # (…)
    }

Plan: 1 to add, 0 to change, 1 to destroy.
```

- `user_data` is shown as a **hash** in plans.
- `# forces replacement` because of `user_data_replace_on_change = true`. New instance, new public IP. (Topic 24's `create_before_destroy` + a load balancer is how production avoids the gap; you'll see that in the capstone.)

Without `user_data_replace_on_change`, the plan would show `~ update in-place` for `user_data`: AWS would store the new script, but **it never runs** on an existing instance. That's a classic trap.

### Destroy

```bash
terraform destroy
```

Takes about a minute (instance termination). Then run your topic 29 leftover scanner.

---

## Hands-on lab

1. Build the project; wait; `curl` the URL; run `terraform plan` to see the check pass.
2. Open an SSM session; read `/var/log/cloud-init-output.log` and `/var/log/user-data.log`; check `curl -s localhost`.
3. Show there's no SSH: `nc -zv -w 3 <public-ip> 22` from WSL → times out.
4. Change `page_message` and apply. Note the new IP.
5. **Break it 1:** remove the egress rule, `-replace` the instance (`terraform apply -replace=aws_instance.web`, topic 41). The page never appears; SSM also stops working (the agent can't reach AWS). Look at **EC2 → Instance → Actions → Monitor and troubleshoot → Get system log**, which shows the boot output without any network access. Restore the rule.
6. **Break it 2:** introduce a typo in the script (`dnf -y install ngin`). Apply. The check warns; the system log or the SSM session shows the error. Fix it.
7. **Break it 3:** remove `user_data_replace_on_change`, change the message, apply. The page doesn't change. Explain why.
8. `terraform destroy` and run the leftover scanner.

---

## Where you'll use it

- **Bootstrapping VMs**: install agents, join clusters, fetch config: `user_data` + cloud-init is how almost every VM gets configured at boot (or Packer bakes it into the AMI, topic 52).
- **SSM instead of SSH**: the standard for production access in AWS; also supports port forwarding and running commands on fleets.
- **Instance roles**: every EC2 instance that talks to AWS (S3, Secrets Manager, ECR = Elastic Container Registry) gets a role, never copied keys.
- **The capstone** (topics 59–61) turns this into modules, adds a load balancer and an Auto Scaling group, and runs it through CI.

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `curl` hangs/times out | SG doesn't allow 80, no public IP, or the subnet isn't public. | Check SG ingress, `public_ip` output, the route table (topic 32). |
| `connection refused` | The instance is reachable but nothing listens on 80 yet: cloud-init still running or failed. | Wait 1–2 min; read `/var/log/cloud-init-output.log` (SSM) or the **system log**. |
| Script changes have no effect | `user_data` only runs at first boot. | `user_data_replace_on_change = true`, or `-replace`. |
| `TargetNotConnected` from `ssm start-session` | The SSM agent isn't registered: no role/profile, no egress, or still booting. | Check the instance profile, the egress rule; wait; **Fleet Manager** in the console shows managed instances. |
| `SessionManagerPlugin is not found` | The local CLI plugin isn't installed. | Install the Session Manager plugin. |
| `InvalidParameterValue: Value (tf-course-web) for parameter iamInstanceProfile.name is invalid` | Instance profile just created; IAM is **eventually consistent** (takes seconds to propagate). | Re-run apply (the provider usually retries). A real-world reason for patience, not `depends_on`. |
| `EntityAlreadyExists: Role with name tf-course-web already exists` | Leftover from an earlier run with lost state. | Delete it or import it (topic 39). |
| Template error: `vars map does not contain key "TOKEN"` | You wrote `${TOKEN}` in the template (Terraform tried to interpolate it). | Use `$TOKEN` or `$${TOKEN}` for shell variables. |

---

## Quick recap

- A production-style web server: VPC (topic 32) + SG with only port 80 + IAM role/instance profile for **SSM** (no SSH) + `user_data` from `templatefile`.
- `user_data` runs **once at first boot**; `user_data_replace_on_change` makes changes replace the instance.
- "Apply complete" ≠ "app works": use a `check` block or a smoke test.
- Debug boot scripts with `/var/log/cloud-init-output.log` (via SSM) or the EC2 **system log**.

### Interview questions

1. **Q: How do you give an EC2 instance permission to read from S3?**
   A: Create an IAM role whose trust policy allows `ec2.amazonaws.com`, attach a least-privilege policy for that bucket, wrap it in an instance profile and set `iam_instance_profile` on the instance. The instance gets temporary credentials from IMDS; no keys on the box.
2. **Q: You changed `user_data` and applied, but nothing changed on the server. Why?**
   A: `user_data` is executed by cloud-init only at first boot. Updating it in place doesn't re-run it. Replace the instance (`user_data_replace_on_change = true` or `-replace`), or use a config management tool for in-place changes.
3. **Q: How do you access instances without opening SSH?**
   A: AWS Systems Manager Session Manager: the instance role has `AmazonSSMManagedInstanceCore`, the agent connects outbound to SSM, and you run `aws ssm start-session`. No inbound port, no keys, and sessions are logged.
