
<h1 align="center">Terraform - Practical Infrastructure as Code Guide</h1>

<p align="center">Essential Terraform commands, HCL configuration, AWS EC2 practice, state management, and troubleshooting.</p>

---

This reference shows how to **define, preview, create, change, and remove infrastructure using code**. Examples use Terraform with AWS. Everything you need for the EC2 lab is shown below; no extra Terraform modules or production architecture are required.

> **Before you start:** `terraform plan` previews changes; `terraform apply` and `terraform destroy` make real changes to your AWS account. EC2 and related resources **may incur charges**, even in accounts with some free-tier benefits. Use a sandbox or authorized AWS account, confirm the region and costs, and clean up afterward.

## Table of Contents

1. [Terraform Basics](#1-terraform-basics)
2. [Installation and AWS Setup](#2-installation-and-aws-setup)
3. [HCL Configuration](#3-hcl-configuration)
4. [Essential Terraform Commands](#4-essential-terraform-commands)
5. [Variables and Outputs](#5-variables-and-outputs)
6. [Terraform State](#6-terraform-state)
7. [Practical AWS EC2 Project](#7-practical-aws-ec2-project)
8. [Common Errors and Troubleshooting](#8-common-errors-and-troubleshooting)
9. [Best Practices](#9-best-practices)

---

## 1. Terraform Basics

Terraform is an **Infrastructure as Code (IaC)** tool. You describe the infrastructure you want in `.tf` files. Terraform compares this configuration with the infrastructure it manages and proposes the necessary actions.

| Term | Meaning | In our AWS example |
|---|---|---|
| Infrastructure as Code | Define infrastructure using configuration files | EC2 instance described in `main.tf` |
| Provider | Plugin that connects Terraform to a platform | `hashicorp/aws` |
| Resource | Infrastructure object Terraform manages | `aws_instance.demo` |
| Data source | Looks up existing information without creating it | `data.aws_ami.al2023` |
| State | Terraform's record of managed resources | Local `terraform.tfstate` |
| Plan | Preview of proposed changes | `terraform plan` |
| Apply | Execute a plan | `terraform apply` |
| Destroy | Remove managed infrastructure | `terraform destroy` |

**Terraform vs AWS Console:** Both can create an EC2 instance. The console involves clicking through settings; Terraform saves the desired settings as reviewable, repeatable code. Terraform is useful when you need consistent changes, a record in Git, or automation. For a single quick manual operation, the console may be simpler.

### Understand the plan symbols

| Plan symbol | Meaning | Example situation |
|---|---|---|
| `+` | Create | EC2 instance does not exist yet |
| `~` | Update in place | Change only the instance's Name tag |
| `-/+` | Replace | A change requires destroying and recreating a resource |
| `-` | Destroy | Remove a resource no longer configured or requested for destruction |

Always read the full plan. The symbols indicate intended actions, not successful completion.

## 2. Installation and AWS Setup

**Requirements:** Terraform CLI, an AWS account with permission to use EC2, and AWS CLI for the identity check. Examples assume a Linux terminal, Git Bash, or PowerShell unless noted otherwise.

### Install or verify Terraform

| Task | Complete command | Explanation |
|---|---|---|
| Install Terraform on Windows (PowerShell) | `winget install -e --id Hashicorp.Terraform` | Installs the matching Terraform package through Windows Package Manager. |
| Verify installation | `terraform version` | Prints the installed Terraform version. |
| See command help | `terraform -help` | Lists Terraform commands. |
| Check AWS CLI | `aws --version` | Confirms AWS CLI is installed. |

For Linux, follow the [official Terraform installation instructions](https://developer.hashicorp.com/terraform/install) for your distribution. For Windows without `winget`, use the same official download page and add the Terraform executable to `PATH`.

### Set up AWS authentication

**Preferred when IAM Identity Center is available:** configure an SSO profile and sign in. The first command is interactive and requires your organization's Identity Center access details.

```bash
aws configure sso --profile terraform-lab
aws sso login --profile terraform-lab
aws sts get-caller-identity --profile terraform-lab
```

If your AWS account uses a different approved credentials method, configure an authorized AWS CLI profile instead. Do **not** place access keys in `main.tf`, `variables.tf`, or GitHub.

Tell Terraform which named profile to use:

**Linux or Git Bash**

```bash
export AWS_PROFILE=terraform-lab
aws sts get-caller-identity
```

**Windows PowerShell**

```powershell
$env:AWS_PROFILE = "terraform-lab"
aws sts get-caller-identity
```

The `AWS_PROFILE` setting applies to the current terminal session. The AWS provider uses the standard AWS credential chain. If the identity check fails, fix authentication before continuing.

## 3. HCL Configuration

Terraform files use **HCL (HashiCorp Configuration Language)**. The basic building blocks are:

| Block or item | Example from our project | Purpose |
|---|---|---|
| `terraform` | `required_providers` | Specify which provider Terraform should install. |
| `provider` | `provider "aws"` | Configure the AWS region. |
| `data` | `data "aws_ami" "al2023"` | Find an existing Amazon Linux image. |
| `resource` | `resource "aws_instance" "demo"` | Define the EC2 instance to create. |
| `variable` | `variable "instance_type"` | Make settings configurable. |
| `output` | `output "instance_id"` | Print useful values after creation. |
| Argument | `instance_type = var.instance_type` | Assign a value or reference another block. |
| Comment | `# EC2 settings` | Explain configuration without affecting Terraform. |

**HCL rules:** Use `{ ... }` for blocks, `=` for assignments, quoted strings such as `"us-east-1"`, and `#` for a single-line comment. Terraform automatically loads `.tf` files in the same working directory. The **complete, connected HCL files** are provided in Section 7.

## 4. Essential Terraform Commands

Run these from the directory containing your `.tf` files.

| Task | Complete command | Explanation |
|---|---|---|
| Check Terraform version | `terraform version` | Confirm CLI availability. |
| Initialize project | `terraform init` | Install providers and prepare the working directory. |
| Update selected provider versions | `terraform init -upgrade` | Reconsider provider versions within the configured constraints; review lockfile changes. |
| Format HCL | `terraform fmt` | Format `.tf` files in the current directory. |
| Format subdirectories too | `terraform fmt -recursive` | Format Terraform files in nested directories. |
| Check formatting without modifying | `terraform fmt -check` | Exit unsuccessfully if local `.tf` files need formatting. |
| Validate configuration | `terraform validate` | Check syntax and provider-supported configuration after `init`. |
| Preview changes | `terraform plan` | Read the expected changes; does not create EC2 instances. |
| Save a plan | `terraform plan -out=tfplan` | Save the proposed actions to a local `tfplan` file. |
| Inspect a saved plan | `terraform show tfplan` | Display the contents of the saved plan. |
| Apply a saved plan | `terraform apply tfplan` | Execute the already-reviewed plan; makes real AWS changes. |
| Apply current configuration | `terraform apply` | Recompute a plan and ask for approval before changing infrastructure. |
| View outputs | `terraform output` | Show all configured root-module outputs. |
| Get one output value | `terraform output -raw instance_id` | Print only the EC2 instance ID. |
| Inspect managed resources | `terraform state list` | List resource addresses recorded in the state. |
| Inspect one resource | `terraform state show aws_instance.demo` | Show the instance attributes stored in state. |
| View current state | `terraform show` | Display a readable rendering of current state. |
| Preview destruction | `terraform plan -destroy` | Show resources Terraform would remove. |
| Destroy managed resources | `terraform destroy` | Review and confirm deletion of managed resources. |

> **Important:** A saved plan (`tfplan`) can contain sensitive information. Do not commit or share it. If configuration, state, or infrastructure changes after the plan is saved, generate a new plan before applying.

## 5. Variables and Outputs

Variables avoid hardcoding values in every resource. Outputs let you read useful information after Terraform has created a resource.

**File: `variables.tf`**

```hcl
variable "aws_region" {
  description = "AWS region for the lab"
  type        = string
  default     = "us-east-1"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}
```

**File: `outputs.tf`**

```hcl
output "instance_id" {
  description = "ID of the created EC2 instance"
  value       = aws_instance.demo.id
}

output "private_ip" {
  description = "Private IPv4 address of the instance"
  value       = aws_instance.demo.private_ip
}
```

The `aws_instance.demo` references above match the resource in Section 7. These outputs are available after a successful `apply`.

### Override the defaults (optional)

Create **`lab.tfvars`** in the Terraform project directory:

```hcl
aws_region    = "us-east-1"
instance_type = "t3.micro"
```

| Task | Complete command | Explanation |
|---|---|---|
| Preview with a variable file | `terraform plan -var-file=lab.tfvars` | Read both values from `lab.tfvars`. |
| Apply using the same variable file | `terraform apply -var-file=lab.tfvars` | Apply with those values after reviewing the displayed plan. |
| Override one variable for a preview | `terraform plan -var="instance_type=t3.micro"` | Set a variable for this one command. |
| Read instance ID | `terraform output instance_id` | Display its named output. |

`terraform.tfvars` and `*.auto.tfvars` are automatically loaded; a file called `lab.tfvars` requires `-var-file=lab.tfvars`. Use the **same variable values** when planning and applying. Avoid putting passwords, keys, or tokens in variable files.

## 6. Terraform State

Terraform uses **state** to track which cloud resources belong to the configuration. For this simple lab, state is stored locally in `terraform.tfstate`. A `.terraform.lock.hcl` file locks selected provider versions; it is **not** the infrastructure state.

| File / directory | Purpose | Commit to Git? |
|---|---|---|
| `main.tf`, `variables.tf`, `outputs.tf` | Configuration | Yes |
| `.terraform.lock.hcl` | Provider dependency lockfile | Yes |
| `.terraform/` | Downloaded provider files / working data | No |
| `terraform.tfstate`, state backups | Resource state, possibly sensitive | No |
| `tfplan` | Saved execution plan, possibly sensitive | No |
| `*.tfvars` | Project variable values, potentially sensitive | Exclude by default; review before sharing |

**Recommended `.gitignore` for the actual Terraform project:**

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
*.tfvars.json
tfplan
tfplan.*
crash.log
crash.*.log
```

Do **not** ignore `.terraform.lock.hcl`. Do **not** delete `terraform.tfstate` while resources still exist: you may lose Terraform's record of resources that need cleanup. In a shared team environment, use a properly secured remote state backend with locking and controlled access; that setup is intentionally outside this beginner lab.

| Task | Complete command | Explanation |
|---|---|---|
| List tracked objects | `terraform state list` | See resource addresses managed by the current state. |
| Inspect an EC2 object | `terraform state show aws_instance.demo` | Display recorded attributes for this exact resource. |
| Inspect formatted state | `terraform show` | Display the state in human-readable form. |

If you encounter a state lock error, check whether another Terraform process is running. Never use `-lock=false` or `force-unlock` just to bypass a legitimate concurrent operation.

## 7. Practical AWS EC2 Project

**Goal:** Create **one Amazon Linux 2023 EC2 instance**, inspect it, modify its Name tag, then remove it using Terraform.

**Prerequisites:** Valid AWS credentials with EC2 permissions, an AWS region with an existing **default VPC and default subnet**, and permission to create and terminate an instance. The example deliberately assigns **no public IP** and configures **no public inbound access**. It does not include SSH access, a key pair, or a custom VPC. If your account has no default VPC, see Troubleshooting.

### Step 1 - Create a local lab directory

In Git Bash, Linux, or PowerShell:

```bash
mkdir terraform-ec2-lab
cd terraform-ec2-lab
```

Create these files in that directory: `main.tf`, `variables.tf`, and `outputs.tf` (contents below). You can also create `.gitignore` using Section 6.

### Step 2 - Define the AWS provider and EC2 instance

**File: `main.tf` (complete)**

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# Find a suitable Amazon Linux 2023 AMI in the chosen region.
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_instance" "demo" {
  ami                         = data.aws_ami.al2023.id
  instance_type               = var.instance_type
  associate_public_ip_address = false

  metadata_options {
    http_tokens = "required"
  }

  tags = {
    Name = "terraform-linux-lab"
  }
}
```

**File: `variables.tf`**

```hcl
variable "aws_region" {
  description = "AWS region for the lab"
  type        = string
  default     = "us-east-1"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}
```

**File: `outputs.tf`**

```hcl
output "instance_id" {
  description = "ID of the created EC2 instance"
  value       = aws_instance.demo.id
}

output "private_ip" {
  description = "Private IPv4 address of the instance"
  value       = aws_instance.demo.private_ip
}
```

| Setting | Example value | Explanation |
|---|---|---|
| Provider source | `"hashicorp/aws"` | Installs the official AWS provider. |
| Provider version | `"~> 6.0"` | Uses an AWS provider release in the `6.x` series. |
| Region | `var.aws_region` | Reads the AWS region defined in `variables.tf`. |
| AMI owner | `["amazon"]` | Restricts image lookup to Amazon-owned images. |
| Latest image | `most_recent = true` | Selects the newest matching AMI; a future image release may change the plan. |
| Instance AMI | `data.aws_ami.al2023.id` | Uses a region-appropriate AMI instead of a hardcoded AMI ID. |
| Instance size | `var.instance_type` | Reads the configured instance type. |
| Public IPv4 | `false` | Avoids automatically assigning a public IPv4 address. |
| Instance metadata | `http_tokens = "required"` | Requires IMDSv2 for metadata access. |
| EC2 Name tag | `"terraform-linux-lab"` | Sets the display name in the AWS EC2 console. |

### Step 3 - Initialize, format, and validate

```bash
terraform init
terraform fmt
terraform validate
```

Expected results: initialization downloads the provider; formatting makes HCL consistent; validation reports **Success! The configuration is valid.** None of these commands creates an EC2 instance.

### Step 4 - Preview before creating

```bash
terraform plan -out=tfplan
terraform show tfplan
```

Look for a resource like `aws_instance.demo` with **`+ create`** and a plan summary normally indicating **1 to add**. The data source may also be read during planning. If the plan proposes an unexpected deletion or replacement, do not apply it.

### Step 5 - Create the instance (may incur charges)

```bash
terraform apply tfplan
```

This applies the saved plan without a separate confirmation prompt. Run it **only after reviewing** `terraform show tfplan`. Terraform then calls AWS and stores the resulting resource ID in state.

### Step 6 - Verify the result

```bash
terraform output
terraform output -raw instance_id
terraform state list
terraform state show aws_instance.demo
```

In **AWS Console > EC2 > Instances**, choose `us-east-1` (unless you changed `aws_region`) and find the instance named `terraform-linux-lab`. The instance should have a private IP and no assigned public IPv4 address. Use the ID printed by Terraform to identify the same instance.

### Step 7 - Make a controlled change

In `main.tf`, change **only** the Name tag:

```hcl
  tags = {
    Name = "terraform-linux-lab-v2"
  }
```

Then run:

```bash
terraform fmt
terraform plan
```

You should normally see **`~ update in-place`** for the changed tag. If the plan looks correct, apply it:

```bash
terraform apply
```

Confirm the displayed plan by typing `yes`. Refresh the EC2 console to see the updated Name tag. This shows how Terraform manages **changes to existing infrastructure**, not just creation.

### Step 8 - Clean up (important)

```bash
terraform plan -destroy
terraform destroy
```

Read the destruction plan carefully. When prompted by `terraform destroy`, type `yes` to terminate the managed EC2 instance. Verify cleanup:

```bash
terraform state list
```

The managed resource should no longer appear. Also check the EC2 console for the instance's **terminated** state. Termination and AWS billing updates may take time; check for separately retained billable resources. Do not delete your state file as a substitute for `terraform destroy`.

## 8. Common Errors and Troubleshooting

| Error or symptom | Likely reason | Exact check or fix |
|---|---|---|
| `terraform: command not found` | Terraform is not installed or not on `PATH` | Install it and rerun `terraform version`; restart the terminal if needed. |
| `No valid credential sources found` | AWS authentication missing or expired | Run `aws sts get-caller-identity --profile terraform-lab`; for SSO, rerun `aws sso login --profile terraform-lab`. |
| `UnauthorizedOperation` / `AccessDenied` | Your AWS identity lacks required permissions | Check the identity with `aws sts get-caller-identity`; request the necessary EC2 permissions. |
| `Failed to query available provider packages` | Registry access, networking, or version constraints | Check connectivity to the Terraform Registry, the version constraint, and rerun `terraform init`. |
| `Missing required provider` / provider not installed | `terraform init` was not run after adding the provider | Run `terraform init`, then `terraform validate`. |
| `No matching AMI found` | AMI filter returns no image in your region | Check `aws_region` and the filters under `data "aws_ami" "al2023"`. |
| Error finding or using a default VPC | Default VPC/subnet is missing | Use a valid subnet and security-group configuration in an authorized VPC; this example assumes a default VPC. |
| `InvalidParameterValue` for instance type | Instance type unsupported in the chosen region/AZ or AMI architecture mismatched | Check available EC2 instance types and keep `x86_64` AMI with an x86-compatible instance type. |
| `Error acquiring the state lock` | Another Terraform operation holds the state lock | Wait for that operation to finish; do not disable locking blindly. |
| `terraform output` is empty | `output` blocks missing or resources not successfully created | Confirm `outputs.tf`, then run `terraform validate` and check `terraform state list`. |
| Plan unexpectedly says `-/+` | A change requires replacement | Inspect the detailed plan and avoid applying until you understand why replacement is needed. |
| `terraform apply tfplan` refuses an old plan | State changed since the plan was saved | Run `terraform plan -out=tfplan` again, review it, then apply the new plan. |
| AWS Console shows no instance | You opened the wrong region/account, or apply failed | Check `aws sts get-caller-identity`, the selected region, `terraform output`, and apply logs. |
| `terraform destroy` does not remove a manually created instance | It is not in this Terraform state | Terraform destroys resources it manages, not arbitrary resources in the account. |

## 9. Best Practices

| Practice | What to do | Why |
|---|---|---|
| Review before changing | Run `terraform plan`; inspect every create/update/replace/destroy action | Prevent unintended infrastructure changes. |
| Keep credentials out of code | Use AWS profiles, SSO, or short-lived credentials | Avoid exposing access keys in Git. |
| Protect state | Restrict access and back up state securely | State may include resource metadata and secrets. |
| Commit configuration | Commit `.tf` files and `.terraform.lock.hcl` | Makes infrastructure changes reviewable and repeatable. |
| Exclude local/generated files | Use the `.gitignore` from Section 6 | Keeps state, plan files, and provider downloads out of Git. |
| Use variables for changeable values | Put region and instance type in `variables.tf` | Makes configurations easier to reuse. |
| Avoid hardcoded regional AMI IDs | Use a restricted `aws_ami` lookup for the lab | Finds a valid AMI in the selected region. |
| Pin images for production | Choose and review a known AMI instead of automatically adopting future releases | Avoid surprising replacement plans after an upstream image update. |
| Apply least privilege | Grant the AWS identity only the required actions | Reduces the impact of mistakes and credential exposure. |
| Clean up labs | Run `terraform destroy` after checking the plan | Avoids forgotten billable resources. |

### Quick practice checklist

- [ ] Install Terraform and verify AWS identity.
- [ ] Explain `provider`, `resource`, `data`, `variable`, and `output` blocks.
- [ ] Create the three `.tf` files exactly as shown.
- [ ] Run `terraform init`, `fmt`, `validate`, and `plan`.
- [ ] Explain the `+`, `~`, `-/+`, and `-` plan symbols.
- [ ] If authorized, create and inspect one AWS EC2 instance.
- [ ] Change only its Name tag and review the new plan.
- [ ] Destroy the instance and confirm the state is clean.

---

### Official references

- [Terraform CLI commands](https://developer.hashicorp.com/terraform/cli/commands)
- [Terraform installation](https://developer.hashicorp.com/terraform/install)
- [Terraform AWS getting started](https://developer.hashicorp.com/terraform/tutorials/aws-get-started)
- [AWS provider documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [AWS CLI IAM Identity Center setup](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html)

*This is a learning reference, not an automatically deployed infrastructure project. Replace example settings with values authorized for your environment before applying.*
