# 32 · AWS Networking with Terraform

> **Level:** ★★ intermediate
>
> **You will learn:** how to build a proper AWS network from code: a **VPC**, **public and private subnets** spread across AZs (with `for_each` and `cidrsubnet`), an **internet gateway**, **route tables** and their **associations**, why private subnets usually need a **NAT gateway** (and why these labs avoid its cost), a free **S3 gateway endpoint**, and **security groups** written the modern way with separate rule resources, including the "Terraform removes the default egress rule" gotcha. Everything in this topic is **free**.
>
> **You need:** topics 18 (`cidrsubnet`), 20 (cycles), 22 (`for_each`), 23 (`for` expressions), 30–31. Networking background: [Linux topics 058](../../linux-mastery-rhel/notes/058-how-networks-work.md), [059](../../linux-mastery-rhel/notes/059-ip-addressing-and-subnetting.md), [061](../../linux-mastery-rhel/notes/061-routing.md), [071](../../linux-mastery-rhel/notes/071-firewalld.md).
>
> **Part 4 · AWS** · Previous: [31 · First AWS resources: S3 and EC2](31-first-aws-resources.md) · Next: [33 · Project 2: a web server on AWS](33-project-2-web-server-on-aws.md) · [Back to INDEX](../INDEX.md)
>
> 💰 **Cost:** VPCs, subnets, internet gateways, route tables, security groups and S3 gateway endpoints are free. The NAT gateway is shown but **not** created.

---

## ⚡ Core: the 10-minute version

**The idea.** A **VPC** is your private network in AWS. You cut it into **subnets** (one per AZ per tier). A subnet is **public** if its **route table** sends `0.0.0.0/0` to an **internet gateway**; otherwise it's **private**. **Security groups** are firewalls attached to instances.

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
}

resource "aws_subnet" "public" {
  for_each                = { for i, az in local.azs : az => cidrsubnet("10.0.0.0/16", 8, i) }
  vpc_id                  = aws_vpc.main.id
  availability_zone       = each.key
  cidr_block              = each.value          # 10.0.0.0/24, 10.0.1.0/24
  map_public_ip_on_launch = true
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
}
```

**Picture it.** A gated housing estate: the VPC is the estate, subnets are streets, the internet gateway is the main gate, the route table is the street signs ("to the outside world: main gate"), and security groups are each house's front-door lock.

**The one thing to know.** "Public" isn't a setting on the subnet. It's the **route** to an internet gateway. And a security group created by Terraform has **no outbound rule** until you add one.

**Try it now:** build the topic's VPC, then open **VPC → Your VPCs → Resource map** in the console to see it drawn.

**You've got it if you can:** explain what makes a subnet public, and why the security group rules are separate resources.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### The pieces

| Resource | What it is | Linux equivalent (from your notes) |
|---|---|---|
| `aws_vpc` | An isolated network with a CIDR range, e.g. `10.0.0.0/16` (65,536 addresses). | A private LAN with its own address plan ([Linux 059](../../linux-mastery-rhel/notes/059-ip-addressing-and-subnetting.md)) |
| `aws_subnet` | A slice of the VPC's range, **in one AZ**. | A subnet / VLAN |
| `aws_internet_gateway` (IGW) | The VPC's door to the internet (two-way, for resources with public IPs). | Your router's WAN side |
| `aws_route_table` + `aws_route` | Routing rules: destination CIDR → target. | `ip route` ([Linux 061](../../linux-mastery-rhel/notes/061-routing.md)) |
| `aws_route_table_association` | Connects a subnet to a route table. | Which routing table an interface uses |
| `aws_nat_gateway` + `aws_eip` | Lets **private** subnets reach the internet (outbound only). 💰 | Masquerading/NAT on a router |
| `aws_vpc_endpoint` | A private path to an AWS service without the internet. The **gateway** type for S3 and DynamoDB is free. | A private link to a service |
| `aws_security_group` | A **stateful** firewall attached to network interfaces (instances, load balancers, databases). | `firewalld` zones ([Linux 071](../../linux-mastery-rhel/notes/071-firewalld.md)) |
| `aws_vpc_security_group_ingress_rule` / `..._egress_rule` | One rule each. | One `firewall-cmd --add-rich-rule` each |
| `aws_network_acl` | A **stateless** subnet-level filter. Rarely changed from the default. | iptables on the router |

Notes:

- AWS reserves **5 addresses** in every subnet (network, router, DNS, future, broadcast). A `/24` gives 251 usable IPs.
- **Stateful** (security groups): if inbound is allowed, the reply is automatically allowed out. You don't write return rules.
- A security group can allow traffic **from another security group** (`referenced_security_group_id`): "the database accepts 5432 from anything in the app group", without IP addresses.

### Public vs private subnets

| | Public subnet | Private subnet |
|---|---|---|
| Route `0.0.0.0/0` → | **Internet gateway** | **NAT gateway** (or nothing) |
| Instances get public IPs? | Usually (`map_public_ip_on_launch = true`) | No |
| Reachable from the internet? | If the security group allows it | **No** |
| Can reach the internet? | Yes | Only through NAT (outbound only) |
| Typical tenants | Load balancers, bastion hosts, NAT gateways | App servers, databases, EKS nodes |

### The NAT gateway cost problem

Production: one NAT gateway per AZ in the public subnets, and private subnets route `0.0.0.0/0` to it. Cost: ≈ USD 0.045+/hour **each** plus data processing, i.e. roughly USD 33+/month per gateway even when idle. For these labs:

- private subnets get **no** internet route (they're still useful for databases),
- an **S3 gateway endpoint** (free) gives private subnets access to S3,
- web servers in topic 33 sit in public subnets behind a security group.

The NAT gateway code is shown below, commented out, so you know how it's done.

### Security groups the modern way

Two styles exist:

```hcl
# ⚠️ older style: inline rules inside the group
resource "aws_security_group" "web" {
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# ✅ current style: the group, plus one resource per rule
resource "aws_security_group" "web" {
  name   = "web"
  vpc_id = aws_vpc.main.id
}

resource "aws_vpc_security_group_ingress_rule" "web_http" {
  security_group_id = aws_security_group.web.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "tcp"
  from_port         = 80
  to_port           = 80
}

resource "aws_vpc_security_group_egress_rule" "web_all" {
  security_group_id = aws_security_group.web.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"            # all protocols; no ports needed
}
```

(The two `"web"` groups are alternatives: use one style, not both.)

Why the separate rules are better: each rule has its own ID, tags and clean diffs; adding one rule doesn't rewrite the whole set; and cross-references between groups don't create **cycles** (topic 20). **Never mix** inline rules and rule resources on the same group: they fight each other.

**The egress gotcha:** when you create a security group in the console, AWS adds an "allow all outbound" rule. When **Terraform** creates one, the provider **removes** that default rule. No egress rule → the instance can't reach the internet (package installs hang). Add an explicit egress rule.

---

## Why it exists

The default VPC (topic 31) is convenient but flat: every subnet is public, the CIDR is fixed (`172.31.0.0/16`), and it may overlap with your office or other VPCs, which breaks peering and VPNs later. Real environments need:

- a **planned address range** that doesn't overlap,
- **tiers** (public for load balancers, private for apps and databases),
- **multiple AZs** for availability,
- **least-privilege firewalls**,
- all of it **identical** in dev, staging and prod. That last one is exactly what Terraform is for.

---

## Analogy

**A gated housing estate.**

- **VPC** = the estate, with its own internal address plan.
- **Subnets** = streets, each in one neighbourhood (AZ).
- **Internet gateway** = the main gate to the city.
- **Route table** = the signposts at the end of each street: "city → main gate". Streets with that sign are **public** streets.
- **NAT gateway** = a courier desk at the gate: residents of private streets can send parcels out and get replies, but outsiders can't walk in.
- **Security group** = each house's door policy: "let in guests from street A on the front door (port 80)". Replies to your own visits are always let back in (stateful).
- **S3 gateway endpoint** = a private tunnel from the estate straight to the warehouse (S3), never touching city roads.

---

## How it works inside

How a packet leaves an instance in a public subnet:

```
instance 10.0.0.25 (public subnet, has public IP 13.x.x.x mapped by the IGW)
   │  security group: is this OUTBOUND allowed? (egress rule) → yes
   ▼
subnet's route table:  10.0.0.0/16 → local      0.0.0.0/0 → igw-0abc
   │  destination 1.1.1.1 matches 0.0.0.0/0
   ▼
internet gateway: translates 10.0.0.25 ↔ 13.x.x.x (1:1 NAT for the public IP)
   ▼ internet
Reply comes back → IGW → instance; the SG lets it in because the connection is tracked (stateful).
```

From a private subnet with no `0.0.0.0/0` route, the packet has nowhere to go: it's dropped. With a NAT gateway route, it goes to the NAT gateway (in a public subnet), which uses **its** public IP.

Terraform's graph for this topic:

```
aws_vpc.main ──┬─▶ aws_subnet.public[az] ──┐
               ├─▶ aws_subnet.private[az] ─┼─▶ aws_route_table_association.*
               ├─▶ aws_internet_gateway ───┼─▶ aws_route_table.public
               ├─▶ aws_route_table.private ┘        └─▶ aws_vpc_endpoint.s3 (attached to route tables)
               └─▶ aws_security_group.* ─▶ aws_vpc_security_group_*_rule.*
```

---

## Code / commands

`~/tf-lab/32-network/` with `versions.tf` and `providers.tf` as in topic 31 (`aws ~> 6.0`, `default_tags`).

`variables.tf`:

```hcl
variable "name" {
  type    = string
  default = "tf-course"
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "vpc_cidr must be a valid CIDR block."
  }
}

variable "az_count" {
  type    = number
  default = 2
}

variable "admin_cidr" {
  type        = string
  description = "Your public IP in CIDR form (x.x.x.x/32) for SSH. Find it: curl -s checkip.amazonaws.com"
  default     = "203.0.113.10/32"
}
```

`network.tf`:

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  azs = slice(data.aws_availability_zones.available.names, 0, var.az_count)

  # 10.0.0.0/24, 10.0.1.0/24 ... for public; 10.0.10.0/24, 10.0.11.0/24 ... for private
  public_subnets  = { for i, az in local.azs : az => cidrsubnet(var.vpc_cidr, 8, i) }
  private_subnets = { for i, az in local.azs : az => cidrsubnet(var.vpc_cidr, 8, i + 10) }
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true          # instances get DNS names; needed by many services
  tags                 = { Name = "${var.name}-vpc" }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.name}-igw" }
}

# ── Subnets ─────────────────────────────────────────────────────────
resource "aws_subnet" "public" {
  for_each                = local.public_subnets
  vpc_id                  = aws_vpc.main.id
  availability_zone       = each.key
  cidr_block              = each.value
  map_public_ip_on_launch = true
  tags                    = { Name = "${var.name}-public-${each.key}", tier = "public" }
}

resource "aws_subnet" "private" {
  for_each          = local.private_subnets
  vpc_id            = aws_vpc.main.id
  availability_zone = each.key
  cidr_block        = each.value
  tags              = { Name = "${var.name}-private-${each.key}", tier = "private" }
}

# ── Routing ─────────────────────────────────────────────────────────
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.name}-public-rt" }
}

resource "aws_route" "public_internet" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.main.id
}

resource "aws_route_table_association" "public" {
  for_each       = aws_subnet.public
  subnet_id      = each.value.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.name}-private-rt" }
  # no 0.0.0.0/0 route: no internet from private subnets in the lab
}

resource "aws_route_table_association" "private" {
  for_each       = aws_subnet.private
  subnet_id      = each.value.id
  route_table_id = aws_route_table.private.id
}

# Free private path to S3 for both tiers
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${data.aws_region.current.region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.public.id, aws_route_table.private.id]
  tags              = { Name = "${var.name}-s3-endpoint" }
}

data "aws_region" "current" {}

# ── 💰 Production only: NAT gateway (≈ USD 33+/month each). Not created in the lab.
# resource "aws_eip" "nat" { domain = "vpc" }
# resource "aws_nat_gateway" "main" {
#   allocation_id = aws_eip.nat.id
#   subnet_id     = aws_subnet.public[local.azs[0]].id
# }
# resource "aws_route" "private_internet" {
#   route_table_id         = aws_route_table.private.id
#   destination_cidr_block = "0.0.0.0/0"
#   nat_gateway_id         = aws_nat_gateway.main.id
# }
```

- `for_each = aws_subnet.public`: iterating over another `for_each` resource (topic 22): one association per subnet, same keys (AZ names).
- `aws_route` as a separate resource (instead of an inline `route {}` block) for the same reason as security group rules.

`security.tf`:

```hcl
resource "aws_security_group" "web" {
  name        = "${var.name}-web"
  description = "Web servers: HTTP from anywhere, SSH from admin"
  vpc_id      = aws_vpc.main.id
  tags        = { Name = "${var.name}-web" }
}

resource "aws_vpc_security_group_ingress_rule" "web_http" {
  security_group_id = aws_security_group.web.id
  description       = "HTTP from anywhere"
  ip_protocol       = "tcp"
  from_port         = 80
  to_port           = 80
  cidr_ipv4         = "0.0.0.0/0"
}

resource "aws_vpc_security_group_ingress_rule" "web_ssh" {
  security_group_id = aws_security_group.web.id
  description       = "SSH from the admin IP only"
  ip_protocol       = "tcp"
  from_port         = 22
  to_port           = 22
  cidr_ipv4         = var.admin_cidr
}

resource "aws_vpc_security_group_egress_rule" "web_all" {
  security_group_id = aws_security_group.web.id
  description       = "All outbound (Terraform removes AWS's default egress rule)"
  ip_protocol       = "-1"                  # all protocols
  cidr_ipv4         = "0.0.0.0/0"
}

resource "aws_security_group" "db" {
  name        = "${var.name}-db"
  description = "Database: PostgreSQL from web servers only"
  vpc_id      = aws_vpc.main.id
  tags        = { Name = "${var.name}-db" }
}

resource "aws_vpc_security_group_ingress_rule" "db_from_web" {
  security_group_id            = aws_security_group.db.id
  description                  = "PostgreSQL from the web tier"
  ip_protocol                  = "tcp"
  from_port                    = 5432
  to_port                      = 5432
  referenced_security_group_id = aws_security_group.web.id   # group-to-group, no IPs
}
```

`outputs.tf`:

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = { for az, s in aws_subnet.public : az => s.id }
}

output "private_subnet_ids" {
  value = { for az, s in aws_subnet.private : az => s.id }
}

output "web_sg_id" {
  value = aws_security_group.web.id
}

output "db_sg_id" {
  value = aws_security_group.db.id
}
```

### Plan summary and apply

```bash
echo "admin_cidr = \"$(curl -s checkip.amazonaws.com)/32\"" > terraform.tfvars
terraform init && terraform apply
```

```
Plan: 20 to add, 0 to change, 0 to destroy.
...
aws_vpc.main: Creating...
aws_vpc.main: Creation complete after 2s [id=vpc-0a1b2c3d4e5f6a7b8]
aws_internet_gateway.main: Creating...
aws_subnet.public["ap-south-1a"]: Creating...
aws_subnet.public["ap-south-1b"]: Creating...
aws_subnet.private["ap-south-1a"]: Creating...
aws_route_table.public: Creating...
aws_security_group.web: Creating...
...
Apply complete! Resources: 20 added, 0 changed, 0 destroyed.

Outputs:

private_subnet_ids = {
  "ap-south-1a" = "subnet-0c1d..."
  "ap-south-1b" = "subnet-0e2f..."
}
public_subnet_ids = {
  "ap-south-1a" = "subnet-0a3b..."
  "ap-south-1b" = "subnet-0b4c..."
}
...
```

- Everything waits for `aws_vpc.main`, then fans out in parallel.

Check the routing with the CLI:

```bash
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$(terraform output -raw vpc_id)" \
  --query 'RouteTables[].{name:Tags[?Key==`Name`]|[0].Value, routes:Routes[].[DestinationCidrBlock||DestinationPrefixListId, GatewayId]}'
```

```json
[
  { "name": "tf-course-public-rt",  "routes": [["10.0.0.0/16","local"], ["pl-78a54011","vpce-0d1e..."], ["0.0.0.0/0","igw-0f2a..."]] },
  { "name": "tf-course-private-rt", "routes": [["10.0.0.0/16","local"], ["pl-78a54011","vpce-0d1e..."]] },
  { "name": null,                   "routes": [["10.0.0.0/16","local"]] }
]
```

- `local`: traffic inside the VPC (always present).
- `pl-...` → `vpce-...`: the S3 endpoint's **prefix list** (AWS's list of S3 IP ranges).
- `0.0.0.0/0 → igw`: only in the public table. That's what makes those subnets public.
- The unnamed table is the VPC's **main route table**, created automatically, used by any subnet without an association.

### Changing things safely

Add a third AZ: `terraform plan -var="az_count=3"` → only the new subnets, associations and nothing else (`for_each` keyed by AZ). With `count` keyed by index, inserting would shuffle CIDRs (topic 21).

Change `vpc_cidr` → **everything** is replaced (a VPC's primary CIDR can't change). In real life that's a migration project, so choose the range carefully up front, without overlapping your other networks.

---

## Hands-on lab

1. Build the network. Open **VPC → Your VPCs → (your VPC) → Resource map** to see subnets, route tables and the IGW drawn.
2. Read the route tables with the CLI command above.
3. `terraform plan -var="az_count=3"`: confirm only additions. Apply it if you like (still free).
4. **The egress gotcha:** comment out `web_all`, apply, and look at the group's **Outbound rules** tab: empty. (Topic 33 shows the symptom: `dnf install` hangs on the instance.) Restore it.
5. **Mixing styles:** add an inline `ingress {}` block to `aws_security_group.web` and plan twice. The two styles fight; remove it.
6. Add a rule letting the **db** group accept SSH from the **web** group, using `referenced_security_group_id`.
7. Keep the network for topic 33, or `terraform destroy` (it's free either way, but keeping state tidy is good practice).

---

## Where you'll use it

- **Every AWS environment** starts with a VPC like this, usually through a module (topic 46 uses the community `terraform-aws-modules/vpc`).
- **EKS** needs subnets in at least two AZs with specific tags; **RDS** needs a DB subnet group from private subnets; **ALBs** need public subnets in two AZs.
- **Security reviews** read these security group rules: least privilege, no `0.0.0.0/0` on SSH, group-to-group rules for internal traffic.
- **Network design interviews**: CIDR planning, public vs private, NAT costs, endpoints.

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| Instance can't reach the internet (`dnf` hangs) | No egress rule (Terraform removed the default), or no route to an IGW/NAT, or no public IP. | Add an egress rule; check the subnet's route table; `map_public_ip_on_launch`. |
| `InvalidSubnet.Conflict` / `The CIDR '10.0.1.0/24' conflicts with another subnet` | Overlapping subnet CIDRs. | Fix the `cidrsubnet` numbering. |
| `InvalidSubnet.Range` | Subnet CIDR outside the VPC CIDR. | Derive subnets from `var.vpc_cidr`. |
| `DependencyViolation` when destroying the VPC/IGW/subnet | Something outside Terraform still uses it (an ENI, a manually launched instance, a load balancer). | Find it: `aws ec2 describe-network-interfaces --filters Name=vpc-id,Values=...`; delete it, re-run destroy. |
| Security group rules flip-flop every plan | Inline rules **and** rule resources on the same group. | Use one style. |
| `InvalidPermission.Duplicate` | The same rule defined twice (two resources, or one created by hand). | Remove the duplicate, or import it (topic 39). |
| Cycle between two security groups | Inline rules referencing each other. | Separate rule resources (as here). |
| Unexpected bill from networking | A NAT gateway, Elastic IPs, or cross-AZ data. | Check before you apply; destroy NAT gateways after labs. |

---

## Quick recap

- VPC → subnets (one AZ each) → route tables → associations. **Public** = route `0.0.0.0/0` to an **internet gateway**.
- Private subnets reach the internet via a **NAT gateway** (💰); the S3 **gateway endpoint** is free.
- Use `for_each` keyed by AZ and `cidrsubnet` for subnet math.
- Security groups are stateful; write rules as **separate resources**, allow **group-to-group**, never mix styles, and always add an **egress** rule.

### Interview questions

1. **Q: What makes a subnet public in AWS?**
   A: Its route table has a route for `0.0.0.0/0` to an internet gateway (and instances need public IPs). Nothing else; there's no "public" flag on the subnet itself.
2. **Q: How do instances in a private subnet download updates?**
   A: Through a NAT gateway in a public subnet (route `0.0.0.0/0` → NAT), or through VPC endpoints for AWS services (e.g. the S3 gateway endpoint), or from an internal mirror.
3. **Q: Why use `aws_vpc_security_group_ingress_rule` resources instead of inline rules?**
   A: Each rule is managed separately with clean diffs and its own ID, group-to-group references don't create dependency cycles, and they avoid the conflicts of inline rule sets. Don't mix the two styles on one group.
