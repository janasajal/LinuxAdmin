# Terraform on AWS — Complete Hands-On Learning Guide

**Author:** Sajal Jana
**Environment:** RHEL 9.6 on AWS EC2 (ap-south-1 / Mumbai)
**Terraform Version:** v1.14.6
**AWS Provider Version:** hashicorp/aws v6.35.1

---

## Table of Contents

1. [What is Terraform?](#1-what-is-terraform)
2. [Installation](#2-installation)
3. [AWS CLI Setup](#3-aws-cli-setup)
4. [Core Terraform Commands](#4-core-terraform-commands)
5. [Your First Terraform File](#5-your-first-terraform-file)
6. [Understanding State](#6-understanding-state)
7. [Variables](#7-variables)
8. [Outputs](#8-outputs)
9. [tfvars & Environments](#9-tfvars--environments)
10. [Locals](#10-locals)
11. [VPC & Networking](#11-vpc--networking)
12. [Security Groups](#12-security-groups)
13. [Data Sources & EC2](#13-data-sources--ec2)
14. [Modules](#14-modules)
15. [Remote State with S3 + DynamoDB](#15-remote-state-with-s3--dynamodb)
16. [terraform state mv — Safe Refactoring](#16-terraform-state-mv--safe-refactoring)
17. [terraform destroy & Cleanup](#17-terraform-destroy--cleanup)
18. [Production Best Practices](#18-production-best-practices)
19. [Project File Structure](#19-project-file-structure)
20. [Key Commands Reference](#20-key-commands-reference)

---

## 1. What is Terraform?

Terraform is an **Infrastructure as Code (IaC)** tool by HashiCorp. It lets you define, provision, and manage cloud infrastructure using human-readable configuration files.

**Why Terraform?**

- **Declarative** — you describe *what* you want, Terraform figures out *how*
- **Cloud-agnostic** — works with AWS, Azure, GCP, and 1000+ providers
- **State-aware** — tracks exactly what exists in your cloud account
- **Plan before apply** — preview changes before making them

---

## 2. Installation

### On RHEL 9 (Production-grade approach using official HashiCorp RPM repo):

```bash
# Add HashiCorp repository
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo

# Install Terraform
sudo yum install -y terraform

# Verify installation
terraform -version
```

**Expected output:**

```
Terraform v1.14.6
on linux_amd64
```

---

## 3. AWS CLI Setup

Terraform uses the AWS CLI credentials to authenticate with AWS.

### Install AWS CLI v2:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify
aws --version
```

### Create IAM User for Terraform (AWS Console):

1. Go to **IAM → Users → Create User**
2. Username: `terraform-learner`
3. Attach policy: `AdministratorAccess` *(use least-privilege in production)*
4. Go to **Security Credentials → Create Access Key → CLI**
5. Download the `.csv` file — you won't see the secret again!

> ⚠️ **Never use `AdministratorAccess` in production.** Create a custom IAM policy with only the permissions Terraform needs.

### Configure AWS CLI:

```bash
aws configure
```

```
AWS Access Key ID:      ← your Access Key
AWS Secret Access Key:  ← your Secret Key
Default region name:    ap-south-1
Default output format:  json
```

Credentials are stored at:

```
~/.aws/credentials     ← Access Key + Secret Key
~/.aws/config          ← Region + output format
```

### Verify credentials:

```bash
aws sts get-caller-identity
```

**Expected output:**

```json
{
    "UserId": "AIDA...",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/terraform-learner"
}
```

---

## 4. Core Terraform Commands

| Command | Description |
|---|---|
| `terraform init` | Download providers, initialize backend, register modules |
| `terraform plan` | Dry run — show what will be created/changed/destroyed |
| `terraform apply` | Execute the plan and make real changes in AWS |
| `terraform destroy` | Destroy all managed resources |
| `terraform output` | Print output values |
| `terraform show` | Show current state |
| `terraform state list` | List all resources in state |
| `terraform state mv` | Move/rename resource in state |
| `terraform force-unlock` | Release a stuck state lock |
| `terraform fmt` | Format code to canonical style |
| `terraform validate` | Validate configuration syntax |

### Plan symbols explained:

| Symbol | Meaning |
|---|---|
| `+` | Will be **created** |
| `-` | Will be **destroyed** |
| `~` | Will be **updated in-place** |
| `-/+` | Will be **destroyed and recreated** |

---

## 5. Your First Terraform File

### Project setup:

```bash
mkdir ~/terraform-learning
cd ~/terraform-learning
```

### File: `~/terraform-learning/main.tf`

This is the root configuration file. It declares the AWS provider and defines your first resource.

```hcl
# ~/terraform-learning/main.tf

# Tell Terraform which provider to use
provider "aws" {
  region = "ap-south-1"
}

# Create an S3 bucket
resource "aws_s3_bucket" "my_first_bucket" {
  bucket = "my-unique-bucket-name-123456"

  tags = {
    Name        = "My First Terraform Bucket"
    Environment = "Learning"
    ManagedBy   = "Terraform"
  }
}
```

### Run it:

```bash
cd ~/terraform-learning
terraform init    # downloads AWS provider plugin → creates ~/terraform-learning/.terraform/
terraform plan    # preview what will be created
terraform apply   # create the bucket (type 'yes' to confirm)
```

After `terraform init`, your directory looks like:

```
~/terraform-learning/
├── main.tf
├── .terraform/                  ← provider plugins downloaded here
│   └── providers/
│       └── registry.terraform.io/hashicorp/aws/6.35.1/linux_amd64/
│           └── terraform-provider-aws_v6.35.1_x5
└── .terraform.lock.hcl          ← provider version lock file (commit this to git)
```

### Key concepts:

- `provider "aws"` — connects Terraform to AWS
- `resource "<type>" "<name>"` — defines an AWS resource
- `<type>` = AWS resource type (e.g., `aws_s3_bucket`)
- `<name>` = your local Terraform name for this resource

---

## 6. Understanding State

### What is state?

After `terraform apply`, Terraform creates `~/terraform-learning/terraform.tfstate`. This is Terraform's **memory** — it records every resource it has created, with all AWS-assigned properties (ARNs, IDs, IPs, etc.).

```bash
cat ~/terraform-learning/terraform.tfstate
```

### Why state matters:

| Without state | With state |
|---|---|
| Terraform doesn't know what exists | Knows exactly what it created |
| Would create duplicate resources | Compares state vs code on every plan |
| Cannot detect drift | Detects manual changes in AWS console |

### State file structure (`~/terraform-learning/terraform.tfstate`):

```json
{
  "version": 4,
  "terraform_version": "1.14.6",
  "serial": 1,
  "lineage": "unique-id",
  "resources": [
    {
      "type": "aws_s3_bucket",
      "name": "my_first_bucket",
      "instances": [{ "attributes": { "..." : "..." } }]
    }
  ]
}
```

> ⚠️ **Never manually edit `~/terraform-learning/terraform.tfstate`**
> ⚠️ **Never commit `terraform.tfstate` to git** — add it to `~/terraform-learning/.gitignore`

---

## 7. Variables

Variables make your Terraform code **reusable and configurable**.

### File: `~/terraform-learning/variables.tf`

All input variable declarations go here.

```hcl
# ~/terraform-learning/variables.tf

variable "aws_region" {
  description = "AWS region to deploy resources"
  type        = string
  default     = "ap-south-1"
}

variable "environment" {
  description = "Environment name (dev / staging / prod)"
  type        = string
  default     = "dev"
}

variable "account_id" {
  description = "Your AWS account ID"
  type        = string
}

variable "key_name" {
  description = "EC2 Key Pair name for SSH access"
  type        = string
}
```

### Variable types:

| Type | Example |
|---|---|
| `string` | `"ap-south-1"` |
| `number` | `3` |
| `bool` | `true` |
| `list(string)` | `["a", "b", "c"]` |
| `map(string)` | `{ key = "value" }` |

### Using variables in `~/terraform-learning/main.tf`:

```hcl
# ~/terraform-learning/main.tf

provider "aws" {
  region = var.aws_region   # reference with var.<name>
}
```

---

## 8. Outputs

Outputs print useful information after `terraform apply` — resource IDs, IPs, ARNs. Essential for passing values between modules.

### File: `~/terraform-learning/outputs.tf`

```hcl
# ~/terraform-learning/outputs.tf

output "bucket_name" {
  description = "The name of the S3 bucket"
  value       = aws_s3_bucket.my_first_bucket.bucket
}

output "bucket_arn" {
  description = "The ARN of the S3 bucket"
  value       = aws_s3_bucket.my_first_bucket.arn
}

output "ec2_public_ip" {
  description = "EC2 Public IP"
  value       = aws_instance.web.public_ip
}
```

### Query outputs anytime:

```bash
cd ~/terraform-learning
terraform output                    # all outputs
terraform output bucket_arn         # specific output
```

---

## 9. tfvars & Environments

`terraform.tfvars` sets actual values for your variables — overriding defaults.

### File: `~/terraform-learning/terraform.tfvars`

```hcl
# ~/terraform-learning/terraform.tfvars
# This file is auto-loaded by Terraform. DO NOT commit if it contains secrets.

aws_region  = "ap-south-1"
environment = "dev"
account_id  = "179158350051"
key_name    = "my-key-pair"
```

### File: `~/terraform-learning/prod.tfvars`

Create separate tfvars files for each environment:

```hcl
# ~/terraform-learning/prod.tfvars

aws_region  = "ap-south-1"
environment = "prod"
account_id  = "179158350051"
key_name    = "prod-key-pair"
```

### Variable priority (low → high):

```
1. default in ~/terraform-learning/variables.tf        ← lowest priority
2. ~/terraform-learning/terraform.tfvars               ← overrides default
3. -var-file="~/terraform-learning/prod.tfvars"        ← overrides tfvars
4. -var="environment=prod"                             ← highest priority
```

### Multiple environments:

```bash
# Dev (uses terraform.tfvars automatically)
cd ~/terraform-learning
terraform apply

# Production (specify a different vars file)
terraform apply -var-file="prod.tfvars"
```

> ⚠️ Add `~/terraform-learning/terraform.tfvars` to `~/terraform-learning/.gitignore` if it contains sensitive values.

---

## 10. Locals

Locals let you define computed values **once** and reuse them everywhere — avoiding repetition.

### File: `~/terraform-learning/locals.tf`

```hcl
# ~/terraform-learning/locals.tf

locals {
  # Build a common prefix used across all resources
  name_prefix = "myapp-${var.account_id}"

  # Common tags applied to every resource
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    Project     = "my-project"
    Owner       = "team-name"
  }
}
```

### Using locals in `~/terraform-learning/main.tf`:

```hcl
# ~/terraform-learning/main.tf

resource "aws_s3_bucket" "example" {
  bucket = local.name_prefix    # referenced with local.<name>

  tags = merge(local.common_tags, {
    Name = "My Specific Bucket"
  })
}
```

### `merge()` function:

`merge()` combines two or more maps — perfect for tags:

```hcl
merge(local.common_tags, { Name = "specific-name" })
# Result: all common_tags + Name tag combined into one map
```

> 🏭 In production, `common_tags` is applied to **every single resource** for cost tracking, compliance, and auditing.

---

## 11. VPC & Networking

A complete production-ready VPC setup with public and private subnets.

### File: `~/terraform-learning/vpc.tf`

This calls the VPC module (defined in Section 14) to provision all networking resources.

```hcl
# ~/terraform-learning/vpc.tf

module "vpc" {
  source = "./modules/vpc"     # points to ~/terraform-learning/modules/vpc/

  aws_region          = var.aws_region
  name_prefix         = local.name_prefix
  vpc_cidr            = "10.0.0.0/16"
  public_subnet_cidr  = "10.0.1.0/24"
  private_subnet_cidr = "10.0.2.0/24"
  common_tags         = local.common_tags
}
```

### What gets created:

```
VPC (10.0.0.0/16)
 ├── Public Subnet (10.0.1.0/24)   → Route Table → Internet Gateway → Internet
 └── Private Subnet (10.0.2.0/24)  → No internet access (internal only)
```

### Resource dependency — Terraform auto-resolves order:

```hcl
# ~/terraform-learning/modules/vpc/main.tf

resource "aws_subnet" "public" {
  vpc_id = aws_vpc.main.id   # Terraform creates VPC first automatically!
}
```

Terraform builds a **dependency graph** and creates resources in the correct order, running independent resources **in parallel** for speed.

---

## 12. Security Groups

### File: `~/terraform-learning/security_group.tf`

```hcl
# ~/terraform-learning/security_group.tf

resource "aws_security_group" "ec2_sg" {
  name        = "${local.name_prefix}-ec2-sg"
  description = "Security group for EC2 instance"
  vpc_id      = module.vpc.vpc_id    # output from ~/terraform-learning/modules/vpc/outputs.tf

  # Allow SSH
  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]   # ⚠️ restrict to VPN CIDR in production!
  }

  # Allow HTTP
  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Allow all outbound
  egress {
    description = "All outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"            # -1 means all protocols
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-ec2-sg"
  })
}
```

> ⚠️ **Production rule:** Never use `0.0.0.0/0` for SSH. Restrict to your company VPN CIDR (e.g., `10.0.0.0/8`) or use a bastion host.

---

## 13. Data Sources & EC2

### Data Sources

Data sources **read existing AWS information** without creating anything. Always prefixed with `data.` when referenced.

### File: `~/terraform-learning/ec2.tf`

```hcl
# ~/terraform-learning/ec2.tf

# ── DATA SOURCE ──────────────────────────────────────────────────────────────
# Fetch the latest RHEL 9 AMI automatically — no hardcoding!
data "aws_ami" "rhel9" {
  most_recent = true
  owners      = ["309956199498"]   # Red Hat's official AWS account ID

  filter {
    name   = "name"
    values = ["RHEL-9*GA*"]
  }

  filter {
    name   = "architecture"
    values = ["x86_64"]
  }
}

# ── RESOURCE ─────────────────────────────────────────────────────────────────
resource "aws_instance" "web" {
  ami                    = data.aws_ami.rhel9.id    # resolved from data source above
  instance_type          = "t3.micro"
  subnet_id              = module.vpc.public_subnet_id        # from ~/terraform-learning/modules/vpc/outputs.tf
  vpc_security_group_ids = [aws_security_group.ec2_sg.id]    # from ~/terraform-learning/security_group.tf
  key_name               = var.key_name                       # from ~/terraform-learning/variables.tf

  root_block_device {
    volume_size = 20
    volume_type = "gp3"
    encrypted   = true    # always encrypt in production!
  }

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web-server"
  })
}
```

**Why use data sources for AMI?**
AMI IDs are region-specific and change with every OS update. Data sources always fetch the **latest** automatically — no hardcoding!

---

## 14. Modules

Modules package infrastructure into **reusable blocks** — like functions in programming.

### Why modules?

```
Without modules:                    With modules:
~/terraform-learning/dev/main.tf    ~/terraform-learning/dev/main.tf    ← 20 lines
  → 200 lines of VPC code             → calls ./modules/vpc
~/terraform-learning/prod/main.tf   ~/terraform-learning/prod/main.tf   ← 20 lines
  → 200 lines of VPC code (duplicate) → calls ./modules/vpc (single source of truth)
```

### Module structure — every module needs 3 files:

```
~/terraform-learning/modules/
└── vpc/
    ├── main.tf        ← actual AWS resources
    ├── variables.tf   ← what the module accepts as input
    └── outputs.tf     ← what the module exposes as output
```

---

### File: `~/terraform-learning/modules/vpc/main.tf`

```hcl
# ~/terraform-learning/modules/vpc/main.tf

resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(var.common_tags, {
    Name = "${var.name_prefix}-vpc"
  })
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id

  tags = merge(var.common_tags, {
    Name = "${var.name_prefix}-igw"
  })
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnet_cidr
  availability_zone       = "${var.aws_region}a"
  map_public_ip_on_launch = true

  tags = merge(var.common_tags, {
    Name = "${var.name_prefix}-public-subnet"
  })
}

resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnet_cidr
  availability_zone = "${var.aws_region}b"

  tags = merge(var.common_tags, {
    Name = "${var.name_prefix}-private-subnet"
  })
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }

  tags = merge(var.common_tags, {
    Name = "${var.name_prefix}-public-rt"
  })
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
```

---

### File: `~/terraform-learning/modules/vpc/variables.tf`

```hcl
# ~/terraform-learning/modules/vpc/variables.tf

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidr" {
  description = "CIDR block for public subnet"
  type        = string
  default     = "10.0.1.0/24"
}

variable "private_subnet_cidr" {
  description = "CIDR block for private subnet"
  type        = string
  default     = "10.0.2.0/24"
}

variable "aws_region" {
  description = "AWS region"
  type        = string
}

variable "name_prefix" {
  description = "Prefix for all resource names"
  type        = string
}

variable "common_tags" {
  description = "Common tags for all resources"
  type        = map(string)
  default     = {}
}
```

---

### File: `~/terraform-learning/modules/vpc/outputs.tf`

```hcl
# ~/terraform-learning/modules/vpc/outputs.tf
# These outputs are consumed in ~/terraform-learning/vpc.tf, ec2.tf, security_group.tf, etc.

output "vpc_id" {
  description = "ID of the created VPC"
  value       = aws_vpc.this.id
}

output "public_subnet_id" {
  description = "ID of the public subnet"
  value       = aws_subnet.public.id
}

output "private_subnet_id" {
  description = "ID of the private subnet"
  value       = aws_subnet.private.id
}

output "internet_gateway_id" {
  description = "ID of the Internet Gateway"
  value       = aws_internet_gateway.this.id
}
```

---

### Calling the module from the root (`~/terraform-learning/vpc.tf`):

```hcl
# ~/terraform-learning/vpc.tf

module "vpc" {
  source = "./modules/vpc"    # relative path to ~/terraform-learning/modules/vpc/

  # Pass inputs (defined in ~/terraform-learning/modules/vpc/variables.tf)
  aws_region          = var.aws_region
  name_prefix         = local.name_prefix
  vpc_cidr            = "10.0.0.0/16"
  public_subnet_cidr  = "10.0.1.0/24"
  private_subnet_cidr = "10.0.2.0/24"
  common_tags         = local.common_tags
}

# Consume module outputs (defined in ~/terraform-learning/modules/vpc/outputs.tf)
# Usage example — EC2 instance uses the public subnet from the module:
# subnet_id = module.vpc.public_subnet_id
```

> ⚠️ Always run `terraform init` after adding a new module — it registers the module path.

---

## 15. Remote State with S3 + DynamoDB

### Why remote state?

| Local state (dangerous) | Remote state (production) |
|---|---|
| Lost if EC2 is terminated | Stored safely in S3 |
| Team members overwrite each other | DynamoDB locking prevents conflicts |
| No audit trail | S3 versioning tracks every change |
| Not encrypted at rest | Encrypted with SSE |

### Step 1 — Create the DynamoDB lock table

Keep this in a **separate bootstrap project**, not mixed with app infrastructure.

### File: `~/terraform-bootstrap/main.tf`

```hcl
# ~/terraform-bootstrap/main.tf
# Run this ONCE to create the state locking table.
# This project must use LOCAL state (no backend block here).

provider "aws" {
  region = "ap-south-1"
}

resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name      = "terraform-state-lock"
    ManagedBy = "Terraform"
  }
}

resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state-bucket-179158350051"

  tags = {
    Name      = "terraform-state-bucket"
    ManagedBy = "Terraform"
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

```bash
# Apply once to create infrastructure for state storage
cd ~/terraform-bootstrap
terraform init
terraform apply
```

---

### Step 2 — Configure S3 backend for your app project

### File: `~/terraform-learning/backend.tf`

```hcl
# ~/terraform-learning/backend.tf
# This block tells Terraform to store state in S3 instead of locally.
# After adding this, run: terraform init -migrate-state

terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket-179158350051"
    key            = "dev/terraform.tfstate"   # path inside the S3 bucket
    region         = "ap-south-1"
    dynamodb_table = "terraform-state-lock"    # created in ~/terraform-bootstrap/main.tf
    encrypt        = true
  }
}
```

---

### Step 3 — Migrate local state to S3:

```bash
cd ~/terraform-learning
terraform init -migrate-state
# Type 'yes' when prompted
# ~/terraform-learning/terraform.tfstate is now uploaded to S3
```

### How locking works:

```
Engineer 1 → runs terraform apply → writes lock entry to DynamoDB → applies safely ✅
Engineer 2 → runs terraform apply → sees lock already exists    → blocked until lock released ✅
              (prevents simultaneous applies and state corruption)
```

### Verify state is in S3:

```bash
aws s3 ls s3://my-terraform-state-bucket-179158350051/dev/
aws s3 cp s3://my-terraform-state-bucket-179158350051/dev/terraform.tfstate - | head -20
```

> 🏭 **Production tip:** Keep `~/terraform-bootstrap/` as a completely separate project. Never run `terraform destroy` inside `~/terraform-learning/` expecting it to delete the bootstrap resources — they live in a different state file.

---

## 16. terraform state mv — Safe Refactoring

When you refactor code (rename resources, move into modules), Terraform sees old and new addresses as different resources and tries to **destroy and recreate** them.

**`terraform state mv` renames resources in state without touching AWS.**

### Example — moving flat resources into the VPC module

Before refactoring, resources were defined directly in `~/terraform-learning/main.tf`:

```
aws_vpc.main
aws_subnet.public
aws_subnet.private
aws_internet_gateway.main
aws_route_table.public
aws_route_table_association.public
```

After refactoring, they live inside `~/terraform-learning/modules/vpc/main.tf`:

```
module.vpc.aws_vpc.this
module.vpc.aws_subnet.public
module.vpc.aws_subnet.private
module.vpc.aws_internet_gateway.this
module.vpc.aws_route_table.public
module.vpc.aws_route_table_association.public
```

### Run `state mv` for each resource:

```bash
cd ~/terraform-learning

terraform state mv aws_vpc.main                          module.vpc.aws_vpc.this
terraform state mv aws_subnet.public                     module.vpc.aws_subnet.public
terraform state mv aws_subnet.private                    module.vpc.aws_subnet.private
terraform state mv aws_internet_gateway.main             module.vpc.aws_internet_gateway.this
terraform state mv aws_route_table.public                module.vpc.aws_route_table.public
terraform state mv aws_route_table_association.public    module.vpc.aws_route_table_association.public
```

After running all commands, verify:

```bash
terraform plan
# Expected output:
# No changes. Your infrastructure matches the configuration.
```

> 🏭 **This is one of the most important production Terraform skills.**
> Senior engineers use `terraform state mv` constantly when reorganizing code.
> Never blindly apply a plan that destroys production — always check if a `state mv` is the right fix!

---

## 17. terraform destroy & Cleanup

```bash
cd ~/terraform-learning
terraform destroy
# Type 'yes' to confirm — destroys ALL resources tracked in state
```

### Common destroy errors and fixes:

**Error: S3 bucket not empty**

```bash
# Empty the S3 bucket first
aws s3 rm s3://my-unique-bucket-name-123456 --recursive

# Then Terraform can destroy it
terraform destroy
```

**Error: State lock stuck** (e.g., a previous apply was interrupted)

```bash
cd ~/terraform-learning
terraform force-unlock <lock-id>
# lock-id is shown in the error message
```

**Error: Backend S3 bucket accidentally deleted before migrating state back**

```bash
# 1. Remove .terraform directory
rm -rf ~/terraform-learning/.terraform
rm -f  ~/terraform-learning/terraform.tfstate

# 2. Remove the backend block from ~/terraform-learning/backend.tf
#    (comment out or delete the terraform { backend "s3" { ... } } block)

# 3. Reinitialize with local backend
cd ~/terraform-learning
terraform init
```

### Production destroy order lesson:

Because `~/terraform-bootstrap/` is a separate project, destroying `~/terraform-learning/` infrastructure can never accidentally destroy your S3 state bucket or DynamoDB lock table. They are managed by completely separate state.

---

## 18. Production Best Practices

### Always tag every resource (`~/terraform-learning/locals.tf`):

```hcl
# ~/terraform-learning/locals.tf

locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    Project     = "project-name"
    Owner       = "team-name"
  }
}
```

Apply to every resource:

```hcl
tags = merge(local.common_tags, { Name = "resource-specific-name" })
```

### Never hardcode sensitive values:

```hcl
# ❌ Wrong — never hardcode credentials anywhere in ~/terraform-learning/
access_key = "AKIAIOSFODNN7EXAMPLE"

# ✅ Right — Terraform reads from ~/.aws/credentials automatically
# Or use IAM Instance Profile when running from an EC2 instance
```

### Always encrypt EBS volumes (`~/terraform-learning/ec2.tf`):

```hcl
# ~/terraform-learning/ec2.tf

root_block_device {
  volume_size = 20
  volume_type = "gp3"
  encrypted   = true
}
```

### Use `-out` flag in CI/CD pipelines:

```bash
# In your CI/CD pipeline (e.g., Jenkins, GitLab CI)
cd ~/terraform-learning
terraform plan -out=tfplan       # save exact plan to ~/terraform-learning/tfplan
terraform apply tfplan           # apply exactly that plan — no surprises
```

### Restrict SSH in security groups (`~/terraform-learning/security_group.tf`):

```hcl
# ~/terraform-learning/security_group.tf

# ❌ Wrong — open to the entire internet
cidr_blocks = ["0.0.0.0/0"]

# ✅ Right — restrict to your corporate VPN or internal network
cidr_blocks = ["10.0.0.0/8"]
```

### `~/terraform-learning/.gitignore`:

```gitignore
# ~/terraform-learning/.gitignore

.terraform/                # provider plugins — regenerated by terraform init
terraform.tfstate          # local state — use remote state in production
terraform.tfstate.backup   # backup of previous state
*.tfvars                   # may contain secrets — evaluate per file
tfplan                     # saved plan files

# Keep these in git:
# .terraform.lock.hcl      ← commit this! Pins provider versions for the team
```

---

## 19. Project File Structure

### Flat structure (small/learning projects):

```
~/terraform-learning/
├── main.tf              ← root resources (provider, misc)
├── variables.tf         ← all input variable declarations
├── outputs.tf           ← all output value declarations
├── locals.tf            ← computed locals (name_prefix, common_tags)
├── vpc.tf               ← module "vpc" call
├── security_group.tf    ← aws_security_group resources
├── ec2.tf               ← data "aws_ami" + aws_instance resources
├── backend.tf           ← terraform { backend "s3" { ... } }
├── terraform.tfvars     ← actual variable values (don't commit secrets!)
├── prod.tfvars          ← production overrides
├── .gitignore           ← excludes .terraform/, tfstate, etc.
├── .terraform.lock.hcl  ← provider version lock (commit this)
└── .terraform/          ← downloaded providers (never commit)
    └── providers/
        └── registry.terraform.io/hashicorp/aws/6.35.1/linux_amd64/
```

### Module structure (production):

```
~/terraform-learning/
├── main.tf
├── variables.tf
├── outputs.tf
├── locals.tf
├── vpc.tf
├── security_group.tf
├── ec2.tf
├── backend.tf
├── terraform.tfvars
├── prod.tfvars
├── .gitignore
├── .terraform.lock.hcl
└── modules/
    ├── vpc/
    │   ├── main.tf        ← aws_vpc, aws_subnet, aws_igw, aws_route_table
    │   ├── variables.tf   ← vpc_cidr, public_subnet_cidr, etc.
    │   └── outputs.tf     ← vpc_id, public_subnet_id, private_subnet_id
    ├── ec2/
    │   ├── main.tf        ← data "aws_ami", aws_instance
    │   ├── variables.tf   ← instance_type, key_name, subnet_id, etc.
    │   └── outputs.tf     ← instance_id, public_ip, private_ip
    └── s3/
        ├── main.tf        ← aws_s3_bucket, versioning, encryption
        ├── variables.tf   ← bucket_name, enable_versioning, etc.
        └── outputs.tf     ← bucket_id, bucket_arn

~/terraform-bootstrap/       ← SEPARATE project for state infrastructure
├── main.tf                  ← aws_s3_bucket (state), aws_dynamodb_table (lock)
├── outputs.tf               ← bucket_name, table_name
└── terraform.tfstate        ← local state only — never remote
```

---

## 20. Key Commands Reference

All commands should be run from within the project directory:

```bash
cd ~/terraform-learning

# ── Initialize ─────────────────────────────────────────────────────────────
terraform init                        # first-time setup; downloads providers to .terraform/
terraform init -migrate-state         # migrate local tfstate to the S3 backend in backend.tf
terraform init -reconfigure           # force reinitialize (e.g., after changing backend config)

# ── Plan & Apply ────────────────────────────────────────────────────────────
terraform plan                        # preview all changes
terraform plan -out=tfplan            # save plan to ~/terraform-learning/tfplan
terraform apply                       # apply changes (prompts for 'yes')
terraform apply tfplan                # apply the exact saved plan (no prompt)
terraform apply -var-file=prod.tfvars # use ~/terraform-learning/prod.tfvars
terraform apply -auto-approve         # skip confirmation prompt (CI/CD only)

# ── State Management ────────────────────────────────────────────────────────
terraform show                        # show all resources in current state
terraform state list                  # list all resource addresses in state
terraform state show aws_instance.web # show full details of a specific resource
terraform state mv <old> <new>        # rename a resource in state (no AWS changes)
terraform state rm <resource>         # remove from state without deleting from AWS
terraform force-unlock <lock-id>      # release a stuck DynamoDB state lock

# ── Outputs ─────────────────────────────────────────────────────────────────
terraform output                      # show all defined outputs
terraform output bucket_arn           # show a specific output value

# ── Cleanup ─────────────────────────────────────────────────────────────────
terraform destroy                     # destroy all resources in state
terraform destroy -target=aws_instance.web  # destroy only a specific resource

# ── Code Quality ────────────────────────────────────────────────────────────
terraform fmt                         # format all .tf files in current directory
terraform validate                    # validate HCL syntax (no AWS calls)
terraform graph                       # output dependency graph in DOT format
```

---

## Complete Infrastructure Built in This Guide

```
AWS Mumbai (ap-south-1)
│
├── ~/terraform-bootstrap/              ← Bootstrap project (separate state)
│   ├── S3 Bucket (state storage)       → my-terraform-state-bucket-179158350051
│   └── DynamoDB Table (state lock)     → terraform-state-lock
│
└── ~/terraform-learning/               ← App project (state in S3)
    ├── S3 Bucket (app)                 → my-unique-bucket-name-123456
    ├── VPC (module)                    → 10.0.0.0/16
    │   ├── Public Subnet               → 10.0.1.0/24  (ap-south-1a)
    │   ├── Private Subnet              → 10.0.2.0/24  (ap-south-1b)
    │   ├── Internet Gateway
    │   └── Route Table                 → public subnet → IGW
    ├── Security Group                  → SSH:22, HTTP:80
    └── EC2 Instance                    → t3.micro, RHEL 9, 20GB gp3 encrypted
```
