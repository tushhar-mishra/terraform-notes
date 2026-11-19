# 31 · First AWS Resources: S3 and EC2

> **Level:** ★ foundation
>
> **You will learn:** to create real AWS resources on the Free Tier: an **S3 bucket** done properly (globally unique name, versioning, encryption, public access block, an uploaded object, and why S3 settings are **separate resources** in the AWS provider) and an **EC2 instance** in the **default VPC**, with the latest **Amazon Linux 2023** image found by a **data source**, IMDSv2 and an encrypted disk. You'll read real AWS plan output, see an **in-place** change (instance size) vs a **replacement** (image), and destroy cleanly.
>
> **You need:** topics 19 (data sources), 24 (replacements), 29–30 (AWS account, provider, profile).
>
> **Part 4 · AWS** · Previous: [30 · The AWS provider and authentication](30-aws-provider-and-auth.md) · Next: [32 · AWS networking with Terraform](32-aws-networking.md) · [Back to INDEX](../INDEX.md)
>
> 💰 **Cost:** S3 with a few KB: effectively free. One `t3.micro`: Free-Tier eligible (check your plan, topic 29), plus a public IPv4 address (≈ USD 0.005/h). **Destroy at the end.**

---

## ⚡ Core: the 10-minute version

**The idea.** Same Terraform, real cloud. A bucket and a VM:

```hcl
resource "random_id" "suffix" {
  byte_length = 4
}

resource "aws_s3_bucket" "lab" {
  bucket        = "tf-course-${random_id.suffix.hex}"   # bucket names are global
  force_destroy = true                                   # lab only: allow destroy with objects inside
}

data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }
}

resource "aws_instance" "lab" {
  ami           = data.aws_ami.al2023.id
  instance_type = "t3.micro"
  tags          = { Name = "tf-course-lab" }
}
```

**Picture it.** Until now you practised in a garage with toy cars. Now you're driving a real car on a quiet road: same controls, but the fuel costs money and other drivers (AWS rules) exist.

**The one thing to know.** Read AWS plans carefully: `instance_type` changes **in place** (a stop/start); `ami` changes **replace** the instance (new disk, new IP, data gone).

**Try it now:** apply, check the console, change `t3.micro` → `t3.small` and plan, then change the AMI filter and plan. Compare. **Destroy.**

**You've got it if you can:** explain why the bucket needs a random suffix and why S3 versioning is a separate resource.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### S3 (Simple Storage Service)

- Stores **objects** (files) in **buckets**. A bucket lives in one region, but its **name is global**: unique across **all** AWS accounts in the world. That's why labs add a random suffix.
- Since 2023, new buckets are **private by default** (Block Public Access on, ACLs off) and **encrypted by default** (SSE-S3, Server-Side Encryption with S3-managed keys). Terraform still declares these explicitly, so the code documents and enforces them.

**One bucket, several resources.** Since AWS provider v4, each S3 bucket setting is its **own resource** that points at the bucket:

| Resource | Setting |
|---|---|
| `aws_s3_bucket` | The bucket itself (name, `force_destroy`, tags) |
| `aws_s3_bucket_versioning` | Keep old versions of objects |
| `aws_s3_bucket_server_side_encryption_configuration` | Encryption (SSE-S3 or SSE-KMS) |
| `aws_s3_bucket_public_access_block` | The four "block public access" switches |
| `aws_s3_bucket_lifecycle_configuration` | Expire or move old objects |
| `aws_s3_bucket_policy` | Who can access it |
| `aws_s3_object` | An object (file) inside the bucket |

⚠️ Old code (v3 and earlier) put `versioning { }`, `server_side_encryption_configuration { }` etc. **inside** `aws_s3_bucket`. Those inline blocks are deprecated or removed. Use the separate resources.

### EC2 (Elastic Compute Cloud)

- An **instance** = a VM. Key arguments: `ami` (the disk image), `instance_type` (size: `t3.micro` = 2 vCPUs, 1 GiB RAM), `subnet_id` (where in the network), `vpc_security_group_ids` (firewall), `key_name` (SSH key), `user_data` (boot script, topic 33), `iam_instance_profile` (role for the VM).
- **AMI** (Amazon Machine Image) IDs are **different in every region** and change with every image release. Never hard-code them; look them up with `data "aws_ami"` (or an SSM public parameter).
- **Default VPC**: every region has a ready-made network with a public subnet in each AZ. If you don't give a `subnet_id`, the instance goes there and gets a public IP. Fine for this first lab; topic 32 builds a proper VPC.
- **IMDSv2**: the **Instance Metadata Service** is an internal HTTP endpoint (`169.254.169.254`) where an instance gets its own info and role credentials. Version 2 requires a session token, which blocks a class of SSRF (Server-Side Request Forgery) attacks. Always require it: `metadata_options { http_tokens = "required" }`.

---

## Why it exists

These are the two most common AWS resources, and they show the two big patterns:

- **S3**: one logical thing configured through **several resources**, with global naming and data you can lose.
- **EC2**: a resource where some changes are **in place** and others **replace** (with real consequences: downtime, new IP, lost local data), and whose inputs (AMI IDs) must be **looked up**, not hard-coded.

---

## Analogy

- **S3 bucket** = a storage unit at a national self-storage chain. The unit's name must be unique in the whole country (global names). Extra services (a lock, insurance, an alarm, a clean-out schedule) are **separate contracts** attached to the unit (separate resources).
- **EC2 instance** = renting a car. Changing to a bigger model of the **same car** can be done at the counter (stop/start, in place). Changing the **engine** (the AMI) means a different car (replacement): you get new keys (new IP), and anything left in the old car is gone.

---

## How it works inside

What `terraform apply` does for `aws_instance` (simplified):

```
provider → ec2:RunInstances(ImageId, InstanceType, TagSpecifications(tags_all), MetadataOptions...)
         ← InstanceId i-0abc..., state "pending"
provider waits: polls ec2:DescribeInstances until state = "running"     ("Still creating... [10s]")
         ← private/public IP, AZ, root volume ID ...  → state
plan later with instance_type changed:
  provider PlanResourceChange → instance_type is updatable (not ForceNew) → "~ update in-place"
  apply → ec2:StopInstances → wait stopped → ec2:ModifyInstanceAttribute → ec2:StartInstances
plan with ami changed:
  ami is ForceNew → "-/+" → TerminateInstances (old) → RunInstances (new)
```

`(known after apply)` appears for everything AWS assigns: `id`, `arn`, `private_ip`, `public_ip`, `availability_zone` (if you didn't pick a subnet), and so on.

---

## Code / commands

`~/tf-lab/31-first-aws/`:

`versions.tf`:

```hcl
terraform {
  required_version = ">= 1.14"
  required_providers {
    aws    = { source = "hashicorp/aws", version = "~> 6.0" }
    random = { source = "hashicorp/random", version = "~> 3.7" }
  }
}

provider "aws" {
  region = "ap-south-1"
  default_tags {
    tags = { project = "tf-course", lab = "31", managed_by = "terraform" }
  }
}
```

### The S3 part (`s3.tf`)

```hcl
resource "random_id" "suffix" {
  byte_length = 4                            # 8 hex characters
}

resource "aws_s3_bucket" "lab" {
  bucket        = "tf-course-lab-${random_id.suffix.hex}"
  force_destroy = true                       # ⚠️ lab only: lets destroy delete a non-empty bucket
}

resource "aws_s3_bucket_versioning" "lab" {
  bucket = aws_s3_bucket.lab.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "lab" {
  bucket = aws_s3_bucket.lab.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"               # SSE-S3; use "aws:kms" + a KMS key for stricter control
    }
  }
}

resource "aws_s3_bucket_public_access_block" "lab" {
  bucket                  = aws_s3_bucket.lab.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_object" "hello" {
  bucket       = aws_s3_bucket.lab.id
  key          = "hello.txt"
  content      = "Uploaded by Terraform\n"
  content_type = "text/plain"
}
```

### The EC2 part (`ec2.tf`)

```hcl
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]                    # only images published by AWS itself

  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]     # Amazon Linux 2023, standard (not minimal), x86
  }

  filter {
    name   = "state"
    values = ["available"]
  }
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

resource "aws_instance" "lab" {
  ami           = data.aws_ami.al2023.id
  instance_type = var.instance_type

  metadata_options {
    http_tokens = "required"                  # IMDSv2 only
  }

  root_block_device {
    volume_type = "gp3"
    volume_size = 8                            # GiB; check the AMI's minimum
    encrypted   = true
  }

  tags = {
    Name = "tf-course-lab"                     # "Name" is the label the console shows
  }
}
```

`outputs.tf`:

```hcl
output "bucket" {
  value = aws_s3_bucket.lab.bucket
}

output "instance" {
  value = {
    id         = aws_instance.lab.id
    ami        = aws_instance.lab.ami
    ami_name   = data.aws_ami.al2023.name
    az         = aws_instance.lab.availability_zone
    public_ip  = aws_instance.lab.public_ip
    private_ip = aws_instance.lab.private_ip
  }
}
```

### Plan (trimmed)

```
$ export AWS_PROFILE=tf-lab && terraform init && terraform plan
data.aws_ami.al2023: Reading...
data.aws_ami.al2023: Read complete after 1s [id=ami-0f1a2b3c4d5e6f708]

  # aws_instance.lab will be created
  + resource "aws_instance" "lab" {
      + ami                                  = "ami-0f1a2b3c4d5e6f708"
      + arn                                  = (known after apply)
      + associate_public_ip_address          = (known after apply)
      + availability_zone                    = (known after apply)
      + id                                   = (known after apply)
      + instance_state                       = (known after apply)
      + instance_type                        = "t3.micro"
      + private_ip                           = (known after apply)
      + public_ip                            = (known after apply)
      + subnet_id                            = (known after apply)
      + tags                                 = {
          + "Name" = "tf-course-lab"
        }
      + tags_all                             = {
          + "Name"       = "tf-course-lab"
          + "lab"        = "31"
          + "managed_by" = "terraform"
          + "project"    = "tf-course"
        }
      + vpc_security_group_ids               = (known after apply)
      ...
      + metadata_options {
          + http_endpoint = "enabled"
          + http_tokens   = "required"
          ...
        }
      + root_block_device {
          + encrypted   = true
          + volume_id   = (known after apply)
          + volume_size = 8
          + volume_type = "gp3"
          ...
        }
    }

  # aws_s3_bucket.lab will be created
  + resource "aws_s3_bucket" "lab" {
      + bucket        = (known after apply)
      + force_destroy = true
      ...
  # aws_s3_bucket_public_access_block.lab will be created
  # aws_s3_bucket_server_side_encryption_configuration.lab will be created
  # aws_s3_bucket_versioning.lab will be created
  # aws_s3_object.hello will be created
  # random_id.suffix will be created

Plan: 7 to add, 0 to change, 0 to destroy.
```

| Line | Meaning |
|---|---|
| `ami = "ami-0f1a..."` | Already known: the data source was read at plan time. |
| `subnet_id = (known after apply)` | No subnet given → AWS picks a default-VPC subnet. |
| `vpc_security_group_ids = (known after apply)` | No group given → the default VPC's **default security group** is used (allows traffic only from itself). |
| `bucket = (known after apply)` | The name includes the random suffix, which doesn't exist yet. |
| The plan for `aws_instance` is ~80 lines | Real AWS resources expose many attributes. Read the ones with values you set, and the summary. |

### Apply

```
aws_instance.lab: Creating...
random_id.suffix: Creating...
random_id.suffix: Creation complete after 0s [id=k3Jx9w]
aws_s3_bucket.lab: Creating...
aws_s3_bucket.lab: Creation complete after 2s [id=tf-course-lab-9372b1c3]
aws_s3_bucket_versioning.lab: Creating...
aws_s3_bucket_public_access_block.lab: Creating...
aws_s3_bucket_server_side_encryption_configuration.lab: Creating...
aws_s3_object.hello: Creating...
...
aws_instance.lab: Still creating... [00m10s elapsed]
aws_instance.lab: Creation complete after 13s [id=i-0a1b2c3d4e5f67890]

Apply complete! Resources: 7 added, 0 changed, 0 destroyed.

Outputs:

bucket = "tf-course-lab-9372b1c3"
instance = {
  "ami" = "ami-0f1a2b3c4d5e6f708"
  "ami_name" = "al2023-ami-2023.9.20261110.0-kernel-6.12-x86_64"
  "az" = "ap-south-1b"
  "id" = "i-0a1b2c3d4e5f67890"
  "private_ip" = "172.31.12.34"
  "public_ip" = "13.233.45.67"
}
```

- The instance and the S3 resources were created **in parallel**; the four S3 settings waited only for the bucket.
- `"Still creating... [00m10s elapsed]"`: the provider polls until the instance is `running`.

Check with the CLI:

```bash
aws s3 ls "s3://$(terraform output -raw bucket)/"
aws ec2 describe-instances --instance-ids "$(terraform output -json instance | jq -r .id)" \
  --query 'Reservations[0].Instances[0].{state:State.Name,type:InstanceType,imds:MetadataOptions.HttpTokens}'
```

```
2026-11-19 12:01:33         22 hello.txt
{
    "state": "running",
    "type": "t3.micro",
    "imds": "required"
}
```

### In place vs replace

Run `terraform plan -var="instance_type=t3.small"`:

```
  # aws_instance.lab will be updated in-place
  ~ resource "aws_instance" "lab" {
        id            = "i-0a1b2c3d4e5f67890"
      ~ instance_type = "t3.micro" -> "t3.small"
        tags          = { "Name" = "tf-course-lab" }
        # (30 unchanged attributes hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

`~` in place, but the **instance stops and starts** during apply (a minute of downtime; the public IP usually changes on stop/start, the private IP doesn't). "In place" doesn't always mean "no impact".

Now change the AMI filter to the ARM image `al2023-ami-2023.*-arm64` and keep the x86 instance type `t3.micro`:

```
  # aws_instance.lab must be replaced
-/+ resource "aws_instance" "lab" {
      ~ ami        = "ami-0f1a2b3c4d5e6f708" -> "ami-0e9d8c7b6a5f4e3d2" # forces replacement
      ~ id         = "i-0a1b2c3d4e5f67890" -> (known after apply)
      ~ private_ip = "172.31.12.34" -> (known after apply)
      ~ public_ip  = "13.233.45.67" -> (known after apply)
      ...
Plan: 1 to add, 0 to change, 1 to destroy.
```

`# forces replacement` on `ami`. This plan would also **fail at apply**: `t3` is an x86 family, and an ARM image needs `t4g`. A `precondition` (topic 25) can catch that at plan time:

```hcl
# inside resource "aws_instance" "lab"
lifecycle {
  precondition {
    condition     = data.aws_ami.al2023.architecture == "x86_64" || startswith(var.instance_type, "t4g")
    error_message = "ARM (arm64) AMIs need a Graviton (AWS ARM CPU) instance type like t4g.*"
  }
}
```

Revert the filter afterwards.

### The "latest AMI" trap

`most_recent = true` means the AMI ID **changes** whenever AWS publishes a new image (often weekly), and the next plan wants to **replace** your instance. For long-lived servers, either pin the AMI ID in a variable and update it on purpose, or add `lifecycle { ignore_changes = [ami] }` (topic 24) and patch the OS in place. For immutable setups (Auto Scaling groups with launch templates), rolling to new AMIs is the goal.

### Destroy

```bash
terraform destroy
```

```
Plan: 0 to add, 0 to change, 7 to destroy.
...
aws_s3_object.hello: Destroying... [id=hello.txt]
aws_instance.lab: Destroying... [id=i-0a1b2c3d4e5f67890]
...
aws_s3_bucket.lab: Destroying... [id=tf-course-lab-9372b1c3]
aws_s3_bucket.lab: Destruction complete after 1s
aws_instance.lab: Still destroying... [00m30s elapsed]
aws_instance.lab: Destruction complete after 42s

Destroy complete! Resources: 7 destroyed.
```

`force_destroy = true` let Terraform empty the versioned bucket (including old versions). Without it, destroy fails with `BucketNotEmpty`, a good default for real data.

---

## Hands-on lab

1. Build and apply the project. Find both resources in the console. Look at the instance's **Tags** tab: your `default_tags` are there.
2. Read `terraform state show aws_instance.lab`: compare the attributes with the console.
3. `terraform apply -var="instance_type=t3.small"` and watch the console: the instance goes **stopping → stopped → pending → running**. Apply without the `-var` to change it back.
4. Change the AMI filter to `arm64` and plan only (don't apply). Add the precondition and see it fail at plan time.
5. Upload a second object with `aws_s3_object` using `source = "${path.module}/README.md"` and `etag = filemd5(...)`. Change the file and plan: the `etag` change makes Terraform re-upload it.
6. Remove `force_destroy`, apply, then try `terraform destroy`: read `BucketNotEmpty`. Put it back and destroy.
7. Run your leftover scanner from topic 29.

---

## Where you'll use it

- **S3**: Terraform state buckets (topic 35), log buckets, artefact buckets, static websites: always with versioning, encryption, public access block.
- **EC2**: build agents, bastion hosts, legacy apps. Most production compute is EC2 under Auto Scaling, ECS or EKS, but the instance arguments are the same.
- **AMI lookups** with data sources in every EC2 module.
- **Reading AWS plans**: the core daily skill: find what you set, the `forces replacement` lines, and the summary.

---

## When it breaks

| Error | Meaning | Fix |
|---|---|---|
| `BucketAlreadyExists` | Someone, anywhere, already uses that name. | Random suffix or `bucket_prefix`. |
| `BucketAlreadyOwnedByYou` | You already have it (maybe from a previous run with lost state). | Import it (topic 39) or delete it. |
| `BucketNotEmpty` on destroy | Bucket has objects/versions. | Empty it, or `force_destroy = true` (labs only). |
| `Your query returned no results` (aws_ami) | Filter matches nothing in this region. | Check the name pattern with `aws ec2 describe-images --owners amazon --filters 'Name=name,Values=al2023-ami-2023*' --query 'Images[].Name'`. |
| `InvalidParameterValue: The architecture 'arm64' of the specified instance type does not match the architecture 'x86_64' of the specified AMI` (or the reverse) | AMI and instance type architectures differ. | Matching pair; add a precondition. |
| `Unsupported: Your requested instance type (t3.micro) is not supported in your requested Availability Zone` | Rare: that type isn't in the chosen AZ. | Pick another subnet/AZ. |
| `VcpuLimitExceeded` / `InstanceLimitExceeded` | Account quota (new accounts have low limits). | Smaller instances, or request a quota increase. |
| Plan replaces the instance every week | `most_recent` AMI changed. | Pin the AMI, or `ignore_changes = [ami]`. |
| `Error: deprecated ... versioning` on old code | Inline S3 blocks from provider v3. | Move them into the separate `aws_s3_bucket_*` resources. |

---

## Quick recap

- S3 names are **global** → random suffix. Each bucket setting is a **separate resource**: versioning, encryption, public access block, policy, lifecycle.
- Look up AMIs with `data "aws_ami"`; never hard-code IDs. Watch for `most_recent` causing replacements.
- EC2: `instance_type` = in place (with stop/start); `ami` = replacement. Require IMDSv2; encrypt the root disk.
- `force_destroy` is for labs. Always `terraform destroy` and check for leftovers.

### Interview questions

1. **Q: Why are S3 versioning and encryption separate resources in the AWS provider?**
   A: Since provider v4, each bucket sub-configuration maps to its own API and resource (`aws_s3_bucket_versioning`, `..._server_side_encryption_configuration`, etc.). It gives smaller, clearer diffs and independent management. The old inline blocks are deprecated.
2. **Q: How do you avoid hard-coding AMI IDs?**
   A: Use the `aws_ami` data source with an owner and name filter (or the public SSM parameter for the image), since AMI IDs differ per region and change with each release.
3. **Q: Which EC2 changes replace an instance?**
   A: Changing the AMI, subnet/AZ, or certain launch-only settings forces replacement. Changing instance type, tags or security groups is done in place, though an instance type change still stops and starts the instance.
