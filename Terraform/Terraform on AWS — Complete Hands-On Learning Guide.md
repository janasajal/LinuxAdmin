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
- Declarative — you describe *what* you want, Terraform figures out *how*
- Cloud-agnostic — works with AWS, Azure, GCP, and 1000+ providers
- State-aware — tracks exactly what exists in your cloud account
- Plan before apply — preview changes before making them

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

### `main.tf` — basic structure:

```hcl
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
terraform init    # downloads AWS provider plugin
terraform plan    # preview what will be created
terraform apply   # create the bucket (type 'yes' to confirm)
```

### Key concepts:
- `provider "aws"` — connects Terraform to AWS
- `resource "<type>" "<name>"` — defines an AWS resource
- `<type>` = AWS resource type (e.g., `aws_s3_bucket`)
- `<name>` = your local Terraform name for this resource

---

## 6. Understanding State

### What is state?

The `terraform.tfstate` file is Terraform's **memory** — it records every resource it has created, with all AWS-assigned properties (ARNs, IDs, IPs, etc.).

```bash
cat terraform.tfstate
```

### Why state matters:

| Without state | With state |
|---|---|
| Terraform doesn't know what exists | Knows exactly what it created |
| Would create duplicate resources | Compares state vs code on every plan |
| Cannot detect drift | Detects manual changes in AWS console |

### State file structure:

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
      "instances": [{ "attributes": { ... } }]
    }
  ]
}
```

> ⚠️ **Never manually edit `terraform.tfstate`**  
> ⚠️ **Never commit `terraform.tfstate` to git** — add it to `.gitignore`

---

## 7. Variables

Variables make your Terraform code **reusable and configurable**.

### `variables.tf`:

```hcl
variable "aws_region" {
  description = "AWS region to deploy resources"
  type        = string
  default     = "ap-south-1"
}

variable "environment" {
  description = "Environment name"
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

### Using variables in code:

```hcl
provider "aws" {
  region = var.aws_region   # reference with var.<name>
}
```

---

## 8. Outputs

Outputs print useful information after `terraform apply` — resource IDs, IPs, ARNs. Essential for passing values between modules.

### `outputs.tf`:

```hcl
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
terraform output                    # all outputs
terraform output bucket_arn         # specific output
```

---

## 9. tfvars & Environments

`terraform.tfvars` sets actual values for your variables — overriding defaults.

### `terraform.tfvars`:

```hcl
aws_region  = "ap-south-1"
environment = "dev"
account_id  = "179158350051"
key_name    = "my-key-pair"
```

### Variable priority (low → high):

```
1. default in variables.tf        ← lowest priority
2. terraform.tfvars               ← overrides default
3. -var-file="prod.tfvars"        ← overrides tfvars
4. -var="environment=prod"        ← highest priority
```

### Multiple environments:

```bash
# Dev (uses terraform.tfvars automatically)
terraform apply

# Production (specify a different vars file)
terraform apply -var-file="prod.tfvars"
```

> ⚠️ Never commit `terraform.tfvars` to git if it contains sensitive values — add to `.gitignore`

---

## 10. Locals

Locals let you define computed values **once** and reuse them everywhere — avoiding repetition.

### `locals.tf`:

```hcl
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

### Using locals:

```hcl
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
# Result: all common_tags + Name tag combined
```

> 🏭 In production, `common_tags` is applied to **every single resource** for cost tracking, compliance, and auditing.

---

## 11. VPC & Networking

A complete production-ready VPC setup with public and private subnets.

### `vpc.tf` (using a module — see Section 14):

```hcl
module "vpc" {
  source = "./modules/vpc"

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
resource "aws_subnet" "public" {
  vpc_id = aws_vpc.main.id   # Terraform creates VPC first automatically!
}
```

Terraform builds a **dependency graph** and creates resources in the correct order, running independent resources **in parallel** for speed.

---

## 12. Security Groups

```hcl
resource "aws_security_group" "ec2_sg" {
  name        = "${local.name_prefix}-ec2-sg"
  description = "Security group for EC2 instance"
  vpc_id      = module.vpc.vpc_id

  # Allow SSH
  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]   # restrict to VPN CIDR in production!
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

> ⚠️ **Production rule:** Never use `0.0.0.0/0` for SSH. Restrict to your company VPN CIDR or use a bastion host.

---

## 13. Data Sources & EC2

### Data Sources

Data sources **read existing AWS information** without creating anything. Always prefixed with `data.` when referenced.

```hcl
# Fetch the latest RHEL 9 AMI automatically
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
```

**Why use data sources for AMI?**  
AMI IDs are region-specific and change with every OS update. Data sources always fetch the **latest** automatically — no hardcoding!

### EC2 Instance:

```hcl
resource "aws_instance" "web" {
  ami                    = data.aws_ami.rhel9.id    # from data source
  instance_type          = "t3.micro"
  subnet_id              = module.vpc.public_subnet_id
  vpc_security_group_ids = [aws_security_group.ec2_sg.id]
  key_name               = var.key_name

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

---

## 14. Modules

Modules package infrastructure into **reusable blocks** — like functions in programming.

### Why modules?

```
Without modules:           With modules:
dev/main.tf   ← 200 lines  dev/main.tf   ← 20 lines (calls module)
prod/main.tf  ← 200 lines  prod/main.tf  ← 20 lines (calls module)
              ← duplicate!               ← single source of truth!
```

### Module structure — every module needs 3 files:

```
modules/
└── vpc/
    ├── main.tf        ← actual resources
    ├── variables.tf   ← what the module accepts as input
    └── outputs.tf     ← what the module exposes as output
```

### `modules/vpc/main.tf`:

```hcl
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(var.common_tags, {
    Name = "${var.name_prefix}-vpc"
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
# ... more resources
```

### `modules/vpc/variables.tf`:

```hcl
variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "aws_region" {
  description = "AWS region"
  type        = string
}

variable "name_prefix" {
  description = "Prefix for resource names"
  type        = string
}

variable "common_tags" {
  description = "Common tags for all resources"
  type        = map(string)
  default     = {}
}
```

### `modules/vpc/outputs.tf`:

```hcl
output "vpc_id" {
  value = aws_vpc.this.id
}

output "public_subnet_id" {
  value = aws_subnet.public.id
}

output "private_subnet_id" {
  value = aws_subnet.private.id
}
```

### Calling the module:

```hcl
module "vpc" {
  source = "./modules/vpc"   # path to module

  # Pass inputs
  aws_region          = var.aws_region
  name_prefix         = local.name_prefix
  vpc_cidr            = "10.0.0.0/16"
  common_tags         = local.common_tags
}

# Consume module outputs
resource "aws_instance" "web" {
  subnet_id = module.vpc.public_subnet_id   # module.<name>.<output>
}
```

> ⚠️ Always run `terraform init` after adding a new module!

---

## 15. Remote State with S3 + DynamoDB

### Why remote state?

| Local state (dangerous) | Remote state (production) |
|---|---|
| Lost if EC2 is terminated | Stored safely in S3 |
| Team members overwrite each other | DynamoDB locking prevents conflicts |
| No audit trail | S3 versioning tracks every change |
| Not encrypted | Encrypted at rest |

### Step 1 — Create DynamoDB lock table:

```hcl
resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = merge(local.common_tags, {
    Name = "terraform-state-lock"
  })
}
```

### Step 2 — Configure S3 backend:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "dev/terraform.tfstate"    # path inside bucket
    region         = "ap-south-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

### Step 3 — Migrate state to S3:

```bash
terraform init -migrate-state
# Type 'yes' when prompted
```

### How locking works:

```
Engineer 1 → writes lock entry to DynamoDB → applies safely ✅
Engineer 2 → sees lock exists → waits or errors out ✅
              (prevents simultaneous applies and state corruption)
```

### Verify state is in S3:

```bash
aws s3 ls s3://my-terraform-state-bucket/dev/
aws s3 cp s3://my-terraform-state-bucket/dev/terraform.tfstate - | head -20
```

> 🏭 **Production tip:** Keep your backend infrastructure (S3 bucket + DynamoDB table) in a **separate Terraform project** (e.g., `terraform-bootstrap`). Never mix it with app infrastructure — you could accidentally destroy your state storage!

---

## 16. terraform state mv — Safe Refactoring

When you refactor code (rename resources, move into modules), Terraform sees old and new addresses as different resources and tries to **destroy and recreate** them.

**`terraform state mv` renames resources in state without touching AWS.**

### Example — moving resources into a module:

```bash
# Before refactor: aws_vpc.main
# After refactor:  module.vpc.aws_vpc.this

terraform state mv aws_vpc.main module.vpc.aws_vpc.this
terraform state mv aws_subnet.public module.vpc.aws_subnet.public
terraform state mv aws_subnet.private module.vpc.aws_subnet.private
terraform state mv aws_internet_gateway.main module.vpc.aws_internet_gateway.this
terraform state mv aws_route_table.public module.vpc.aws_route_table.public
terraform state mv aws_route_table_association.public module.vpc.aws_route_table_association.public
```

After running these, `terraform plan` should show:
```
No changes. Your infrastructure matches the configuration.
```

> 🏭 **This is one of the most important production Terraform skills.**  
> Senior engineers use `terraform state mv` constantly when reorganizing code.  
> Never blindly apply a plan that destroys production — always check if a `state mv` is the right fix!

---

## 17. terraform destroy & Cleanup

```bash
terraform destroy
# Type 'yes' to confirm — destroys ALL managed resources
```

### Common destroy errors and fixes:

**Error: S3 bucket not empty**
```bash
# Empty the bucket first
aws s3 rm s3://my-bucket-name --recursive

# Then delete it
aws s3 rb s3://my-bucket-name
```

**Error: State lock stuck**
```bash
terraform force-unlock <lock-id>
```

**Error: Backend S3 bucket deleted before migrating state back**
```bash
# Remove .terraform cache and reinitialize with local backend
rm -rf .terraform
rm -f terraform.tfstate

# Remove backend block from backend.tf, then:
terraform init
```

### Production destroy order lesson:

Keep backend resources (S3 + DynamoDB) in a **separate project** so they are never accidentally destroyed with app infrastructure.

---

## 18. Production Best Practices

### Always tag every resource:
```hcl
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    Project     = "project-name"
    Owner       = "team-name"
  }
}
```

### Never hardcode sensitive values:
```hcl
# ❌ Wrong
access_key = "AKIAIOSFODNN7EXAMPLE"

# ✅ Right — use environment variables or AWS IAM roles
# Terraform automatically reads from ~/.aws/credentials
```

### Always encrypt disks:
```hcl
root_block_device {
  encrypted = true
}
```

### Use `-out` flag in CI/CD:
```bash
terraform plan -out=tfplan      # save plan
terraform apply tfplan          # apply exactly that plan
```

### Restrict SSH in security groups:
```hcl
# ❌ Wrong — open to internet
cidr_blocks = ["0.0.0.0/0"]

# ✅ Right — restrict to VPN or bastion
cidr_blocks = ["10.0.0.0/8"]
```

### `.gitignore` for Terraform projects:
```
.terraform/
terraform.tfstate
terraform.tfstate.backup
*.tfvars          # if contains secrets
.terraform.lock.hcl   # optional — some teams commit this
```

---

## 19. Project File Structure

### Flat structure (small projects):

```
terraform-project/
├── main.tf              ← resources
├── variables.tf         ← input variables
├── outputs.tf           ← output values
├── locals.tf            ← local computed values
├── vpc.tf               ← VPC resources
├── security_group.tf    ← security group resources
├── ec2.tf               ← EC2 resources
├── backend.tf           ← backend + DynamoDB
├── terraform.tfvars     ← variable values (don't commit secrets!)
└── .terraform/          ← provider plugins (never commit)
```

### Module structure (production):

```
terraform-project/
├── main.tf
├── variables.tf
├── outputs.tf
├── locals.tf
├── backend.tf
├── terraform.tfvars
└── modules/
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── ec2/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── s3/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

---

## 20. Key Commands Reference

```bash
# Initialize
terraform init                    # first time setup
terraform init -migrate-state     # migrate to new backend
terraform init -reconfigure       # force reinitialize

# Plan & Apply
terraform plan                    # preview changes
terraform plan -out=tfplan        # save plan to file
terraform apply                   # apply changes
terraform apply tfplan            # apply saved plan
terraform apply -var-file=prod.tfvars  # use specific vars file
terraform apply -auto-approve     # skip confirmation (use in CI/CD only)

# State management
terraform show                    # show current state
terraform state list              # list all resources
terraform state show <resource>   # show specific resource
terraform state mv <old> <new>    # rename resource in state
terraform state rm <resource>     # remove resource from state (doesn't delete from AWS)
terraform force-unlock <lock-id>  # release stuck lock

# Outputs
terraform output                  # show all outputs
terraform output <name>           # show specific output

# Cleanup
terraform destroy                 # destroy all resources
terraform destroy -target=<resource>  # destroy specific resource

# Code quality
terraform fmt                     # format code
terraform validate                # validate syntax
terraform graph                   # generate dependency graph
```

---

## Complete Infrastructure Built in This Guide

```
AWS Mumbai (ap-south-1)
├── S3 Bucket            → app storage + state storage
├── DynamoDB Table       → terraform state locking
├── VPC (module)         → vpc-xxxxxxxxx (10.0.0.0/16)
│   ├── Public Subnet    → subnet-xxxxxxxxx (10.0.1.0/24) ap-south-1a
│   ├── Private Subnet   → subnet-xxxxxxxxx (10.0.2.0/24) ap-south-1b
│   ├── Internet Gateway → igw-xxxxxxxxx
│   └── Route Table      → rtb-xxxxxxxxx (public → IGW)
├── Security Group       → sg-xxxxxxxxx (SSH:22, HTTP:80)
└── EC2 Instance         → t3.micro, RHEL 9, 20GB gp3 encrypted
```

**13 production-ready AWS resources — all managed by Terraform!**

---

*Document created based on hands-on learning session.*  
*Author: Sajal Jana*
