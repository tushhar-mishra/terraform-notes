# 29 · AWS Basics for Terraform

> **Level:** ★ foundation
>
> **You will learn:** the AWS knowledge Terraform assumes: accounts, the **root user** and why you lock it away, **IAM** (users, groups, roles, policies), **IAM Identity Center** (SSO, Single Sign-On) and short-lived credentials, **regions** and **availability zones**, the **Free Tier** (it changed in July 2025) and the services that quietly cost money, a **budget alert**, installing the **AWS CLI v2**, and setting up a **named profile** that Terraform will use in topic 30.
>
> **You need:** topic 05 (environment variables, APIs). No AWS experience assumed.
>
> **Part 4 · AWS** · Previous: [28 · Terraform and Kubernetes (kind)](28-terraform-and-kubernetes.md) · Next: [30 · The AWS provider and authentication](30-aws-provider-and-auth.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Before Terraform can touch AWS, you need: an **account**, a safe **identity** that isn't the root user, **credentials** on your machine, a chosen **region**, and a **budget alert** so mistakes don't become bills.

```bash
aws configure sso --profile tf-lab     # recommended: short-lived credentials via IAM Identity Center
# or, simpler for a personal lab:  aws configure --profile tf-lab   (access keys of an IAM user)
aws sts get-caller-identity --profile tf-lab
```

```json
{
  "UserId": "AROA...:you",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/AWSReservedSSO_AdministratorAccess_.../you"
}
```

**Picture it.** AWS is a huge building. The **root user** is the master key: lock it in a safe. **IAM** hands out ID badges with specific door permissions. A **region** is a city where the building has a branch; **availability zones** are separate buildings in that city.

**The one thing to know.** Never use the root user's access keys, never commit keys, set a **budget alert** before your first `terraform apply`, and **destroy** every lab.

**Try it now:** install the AWS CLI, configure a `tf-lab` profile, and run `aws sts get-caller-identity --profile tf-lab`.

**You've got it if you can:** explain user vs role, region vs AZ, and name three AWS things that cost money even when idle.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Accounts and the root user

- An **AWS account** is a container for resources and billing, identified by a 12-digit **account ID**.
- The email you signed up with is the **root user**: it can do **everything**, including closing the account. Protect it: strong password, **MFA** (Multi-Factor Authentication), **no access keys**, and use it only for the few tasks that require it (billing settings, closing the account).
- Companies use **AWS Organizations**: many accounts (dev, staging, prod, logging, security) under one management account. Separate accounts are the strongest isolation boundary in AWS.

### IAM (Identity and Access Management)

| Thing | What it is | Credentials | Typical use |
|---|---|---|---|
| **IAM user** | A long-lived identity in one account | Password and/or **access keys** (long-lived) | Legacy; break-glass; simple personal labs |
| **IAM group** | A set of users sharing policies | — | Attach permissions to many users |
| **IAM role** | An identity **anyone allowed** can temporarily "become" (**assume**) | **Temporary** credentials (minutes to hours) | CI pipelines, EC2 instances, cross-account access, SSO users |
| **Policy** | A JSON document: which **actions** on which **resources** are **allowed/denied** | — | Attached to users, groups, roles |

A policy example (allow reading one bucket):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::acme-reports", "arn:aws:s3:::acme-reports/*"]
    }
  ]
}
```

- **ARN** (Amazon Resource Name): the unique ID of anything in AWS: `arn:aws:<service>:<region>:<account>:<resource>`.
- **Least privilege**: grant only the actions needed. For learning labs, an administrator role in a **sandbox account** is acceptable; production pipelines get narrow roles (topic 57).

### IAM Identity Center (SSO)

The modern way for humans to get AWS access: you log in through a browser, pick an account and a **permission set** (a role), and the CLI receives **temporary** credentials. Nothing long-lived sits on your laptop. It's free and set up from the console (IAM Identity Center → Enable).

### Regions and Availability Zones

- A **region** is a geographic area with its own copy of AWS services: `ap-south-1` (Mumbai), `us-east-1` (N. Virginia), `eu-west-1` (Ireland). Most resources live in **one** region; a few are **global** (IAM, Route 53, CloudFront).
- An **Availability Zone (AZ)** is one or more separate data centres in a region: `ap-south-1a`, `ap-south-1b`, `ap-south-1c`. Spreading across AZs survives a data-centre failure.
- These notes use **`ap-south-1`**. Pick a region near you and use it consistently.

### The Free Tier (changed July 2025)

- **Accounts created on or after 15 July 2025**: you choose a **Free plan** (sign-up credits, about USD 100 plus up to USD 100 more for completing starter activities, valid for **6 months**; the account can't incur charges beyond the credits unless you upgrade) or a **Paid plan**. Credits apply to most services, and small instance types like `t3.micro` / `t4g.micro` are eligible.
- **Older accounts** keep the **legacy 12-month Free Tier** (e.g. 750 hours/month of `t2.micro`/`t3.micro`, 5 GB of S3, for 12 months after sign-up).
- **Always-free** offers (some Lambda, DynamoDB, CloudWatch usage) exist on both.
- Terms change. Check the AWS Free Tier page and your **Billing → Free Tier** dashboard before labs.

### Things that cost money even when idle (lab traps)

| Resource | Why it's a trap |
|---|---|
| **NAT Gateway** | Charged per hour (≈ USD 0.045+/h, region-dependent) **plus** per GB. A forgotten one is ≈ USD 30+/month. These labs avoid it. |
| **Public IPv4 addresses** | Every public IPv4 address costs ≈ USD 0.005/h (since 2024), attached or not. |
| **Elastic IPs**, **load balancers**, **EKS clusters** (≈ USD 0.10/h for the control plane) | Hourly charges whether used or not. |
| **EBS volumes and snapshots** | Disks left behind after instances are deleted. |
| **Data transfer out** | Downloads to the internet beyond the free allowance. |
| **Resources in another region** | You forgot they exist because the console shows one region at a time. |

The rule for every AWS lab: **`terraform destroy` at the end**, then check the billing console the next day.

---

## Why it exists

Terraform's AWS provider calls the same APIs as the console and CLI (topic 05), **as some identity**. If that identity is root, a leaked key means a lost account. If credentials are long-lived keys in a file, a stolen laptop or a committed `.env` file means a breach. If there's no budget, a typo in `count` means a surprise bill. This topic sets up the safety rails **before** you automate anything.

---

## Analogy

**A large office building with branches in many cities.**

- **Account** = one company's floor space and its electricity bill.
- **Root user** = the building owner's master key. Kept in a safe; only for emergencies.
- **IAM user** = an employee's permanent badge. **IAM role** = a visitor badge issued at reception for a few hours, to whoever is allowed to ask for it.
- **Policy** = the list of doors a badge opens.
- **Region** = which city's branch. **AZ** = separate buildings in that city, with separate power, so one fire doesn't stop the business.
- **Budget alert** = the accountant who emails you when the electricity bill passes a limit.

---

## How it works inside

How a credential becomes an API call (what the AWS provider does in topic 30):

```
1. Find credentials (in order): env vars → profile in ~/.aws/config & ~/.aws/credentials
   (SSO cache, assume-role, keys) → container/instance role ...
2. If SSO: use the cached token from `aws sso login` → get temporary role credentials
   (AccessKeyId + SecretAccessKey + SessionToken, valid for hours)
3. Sign each HTTPS request with the secret key (AWS Signature Version 4: an HMAC, a keyed cryptographic hash,
   of the request, date, region and service). The secret key itself is never sent.
4. AWS checks the signature → who you are (authentication)
   → evaluates policies → is this action on this resource allowed? (authorization)
5. Allowed → do it. Denied → 403 AccessDenied / UnauthorizedOperation.
```

The files the CLI and Terraform share:

| File | Holds |
|---|---|
| `~/.aws/config` | Profiles: region, output format, SSO settings, `role_arn`/`source_profile` for assuming roles |
| `~/.aws/credentials` | Access keys (if you use them). Treat it like a password file: `chmod 600`. |
| `~/.aws/sso/cache/` | Temporary SSO tokens |

---

## Code / commands

### 1. Secure the account (console, once)

1. Sign in as root → **Security credentials** → assign an **MFA** device. Delete any root access keys.
2. **Billing and Cost Management → Budgets → Create budget** → "Zero spend budget" or a monthly cost budget of e.g. USD 5, with an email alert at 80% and 100%.
3. **IAM Identity Center → Enable**. Create a user for yourself, a permission set `AdministratorAccess` (sandbox/lab account only), and assign it to your user in this account. Note the **AWS access portal URL** (`https://d-xxxxxxxxxx.awsapps.com/start`).

   *(Simpler alternative for a personal lab: IAM → Users → create `tf-lab` with programmatic access and `AdministratorAccess`, then create an access key. Long-lived keys are less safe; delete them when you finish the course.)*

### 2. Install the AWS CLI v2 in WSL2

```bash
cd /tmp
curl -fsSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip -q awscliv2.zip && sudo ./aws/install
aws --version
```

```
aws-cli/2.31.x Python/3.13.x Linux/6.6.x-microsoft-standard-WSL2 exe/x86_64.ubuntu.24
```

(AWS also publishes a signature file for the zip if you want to verify it, like topic 06's checksums.)

### 3a. Configure an SSO profile (recommended)

```bash
aws configure sso --profile tf-lab
```

```
SSO session name (Recommended): my-sso
SSO start URL [None]: https://d-xxxxxxxxxx.awsapps.com/start
SSO region [None]: ap-south-1
SSO registration scopes [sso:account:access]:
Attempting to automatically open the SSO authorization page in your default browser.
...
The only AWS account available to you is: 123456789012
Using the account ID 123456789012
The only role available to you is: AdministratorAccess
Using the role name "AdministratorAccess"
Default client Region [None]: ap-south-1
CLI default output format (json if not specified) [None]: json
Profile name [AdministratorAccess-123456789012]: tf-lab
```

It writes `~/.aws/config`:

```ini
[profile tf-lab]
sso_session = my-sso
sso_account_id = 123456789012
sso_role_name = AdministratorAccess
region = ap-south-1
output = json

[sso-session my-sso]
sso_start_url = https://d-xxxxxxxxxx.awsapps.com/start
sso_region = ap-south-1
sso_registration_scopes = sso:account:access
```

Each day (tokens expire): `aws sso login --profile tf-lab`.

### 3b. Or an access-key profile (simple lab)

```bash
aws configure --profile tf-lab
```

```
AWS Access Key ID [None]: AKIA................
AWS Secret Access Key [None]: ****************************************
Default region name [None]: ap-south-1
Default output format [None]: json
```

Keys go to `~/.aws/credentials`. Check: `ls -l ~/.aws/credentials` should show `-rw-------`.

### 4. Check who you are

```bash
aws sts get-caller-identity --profile tf-lab
export AWS_PROFILE=tf-lab           # now every aws/terraform command in this shell uses it
aws ec2 describe-availability-zones --query 'AvailabilityZones[].ZoneName' --output text
```

```
ap-south-1a	ap-south-1b	ap-south-1c
```

- `sts get-caller-identity` = "who am I?". It works even with zero permissions, so it's the first thing to run when debugging auth.
- `AWS_PROFILE` (topic 05) is how you tell both the CLI and Terraform which profile to use.

### 5. Find leftovers across all regions

```bash
for r in $(aws ec2 describe-regions --query 'Regions[].RegionName' --output text); do
  n=$(aws ec2 describe-instances --region "$r" --query 'length(Reservations[].Instances[])' --output text)
  [ "$n" != "0" ] && echo "$r: $n instance(s)"
done; echo "scan done"
```

Run it after labs. Billing → **Bills** and **Cost Explorer** show the rest.

---

## Hands-on lab

1. Secure the root user with MFA; delete root access keys if any.
2. Create a **budget** with an email alert (zero-spend or USD 5).
3. Set up **IAM Identity Center** with a user and an `AdministratorAccess` permission set (or an IAM user with keys if you prefer the simple path).
4. Install the AWS CLI, configure the `tf-lab` profile, run `aws sts get-caller-identity`.
5. Run `aws s3 ls` and `aws ec2 describe-vpcs --query 'Vpcs[].{id:VpcId,default:IsDefault,cidr:CidrBlock}'`. Every region has a **default VPC** (a ready-made network); you'll meet it in topic 31.
6. Run the leftover scanner. Keep it as `~/bin/aws-leftovers.sh`.

---

## Where you'll use it

- Every AWS Terraform run needs a profile/role like this, locally or in CI.
- **CI pipelines** use roles with **OIDC** instead of stored keys ([GitHub Actions topic 36](../../GitHub-Actions-Basics-To-Production/notes/36-oidc-cloud-authentication.md); Terraform version in topic 57).
- **Multi-account setups** use roles assumed across accounts (topic 30).
- **Cost control** is part of every DevOps job: budgets, tags for cost allocation, cleanup.

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `Unable to locate credentials` | No profile selected and no env vars. | `export AWS_PROFILE=tf-lab`, or pass `--profile`. |
| `The SSO session associated with this profile has expired` / `Token has expired` | SSO tokens are short-lived. | `aws sso login --profile tf-lab`. |
| `InvalidClientTokenId` / `SignatureDoesNotMatch` | Wrong, deleted or mistyped access keys; or a clock problem. | Recreate keys; check WSL time (`date`; `sudo hwclock -s` if WSL's clock drifted after sleep). |
| `AccessDenied` / `UnauthorizedOperation` | Authenticated, but the policy doesn't allow this action. | Check `get-caller-identity` (right role?), then the permissions. |
| Resources "disappeared" in the console | You're looking at a different region. | Check the region selector (top right). |
| Unexpected bill | Forgotten NAT gateway, public IPs, volumes, another region. | Budget alerts, `terraform destroy`, the leftover scanner, Cost Explorer. |

---

## Quick recap

- Lock away the **root user** (MFA, no keys). Use **IAM Identity Center** (temporary credentials) or, for a simple lab, an IAM user's keys.
- **Users** are long-lived; **roles** are assumed for temporary credentials; **policies** define allowed actions on ARNs.
- **Region** = geographic area; **AZ** = isolated data centre(s) in it.
- Free Tier changed in July 2025: check your plan. Watch idle costs: NAT gateways, public IPv4, load balancers, EKS, volumes.
- Set a **budget alert**, use a named **profile** (`AWS_PROFILE`), and **destroy** after every lab.

### Interview questions

1. **Q: IAM user vs IAM role?**
   A: A user is a long-lived identity with a password and/or access keys. A role has no long-lived credentials; trusted principals (users, services, CI via OIDC, other accounts) assume it to get temporary credentials. Roles are preferred for automation.
2. **Q: Why shouldn't Terraform run with root credentials?**
   A: Root has unrestricted access and can't be limited by IAM policies. A leak would compromise the whole account. Use roles with least privilege and temporary credentials.
3. **Q: What's the difference between a region and an availability zone?**
   A: A region is a geographic area with its own set of AWS services; an AZ is an isolated data centre (or group of them) within a region. You spread resources across AZs for high availability.
