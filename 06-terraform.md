# Terraform Cheat Sheet

> Infrastructure as Code with Terraform — CLI workflow, state management, HCL syntax, variables, modules, and provisioning patterns.

---

## Table of Contents

- [Core Workflow](#1-core-workflow)
- [Plan & Apply Options](#2-plan--apply-options)
- [State Management](#3-state-management)
- [Import & Move Resources](#4-import--move-resources)
- [Workspaces](#5-workspaces)
- [Inspect & Debug](#6-inspect--debug)
- [HCL Basics — Variables, Locals, Outputs](#7-hcl-basics--variables-locals-outputs)
- [Resources & Data Sources](#8-resources--data-sources)
- [Modules](#9-modules)
- [Meta-Arguments & Functions](#10-meta-arguments--functions)
- [Provisioners](#11-provisioners)
- [Remote Backends](#12-remote-backends)

---

## 1. Core Workflow

```
terraform init                  # Initialize, download providers & modules
terraform init -upgrade         # Upgrade provider versions
terraform init -backend-config=backend.hcl
terraform fmt                   # Format code to canonical style
terraform fmt -check -recursive # Check formatting (CI)
terraform validate              # Validate configuration
terraform plan                  # Preview changes
terraform plan -out=tfplan      # Save plan for apply
terraform apply                 # Apply changes (asks confirmation)
terraform apply tfplan          # Apply saved plan
terraform apply -auto-approve   # Skip confirmation (CI only)
terraform destroy               # Destroy all managed infrastructure
terraform destroy -target=aws_instance.web
```

## 2. Plan & Apply Options

```
terraform plan -var="instance_type=t3.large"
terraform plan -var-file=prod.tfvars
terraform plan -var-file=secrets.tfvars
terraform plan -target=module.network          # Plan only specific module/resource
terraform plan -replace=aws_instance.web       # Force replacement (taint replacement)
terraform plan -refresh=false                  # Skip state refresh
terraform apply -parallelism=10                # Control concurrent operations
terraform destroy -target=aws_s3_bucket.data   # Destroy single resource
```

## 3. State Management

```
terraform state list                       # All resources in state
terraform state show aws_instance.web      # Full attributes of one resource
terraform state rm aws_instance.web        # Untrack (does NOT delete real resource)
terraform state mv aws_instance.old aws_instance.new   # Rename/restructure
terraform state pull > backup.tfstate      # Backup state
terraform state replace-provider hashicorp/aws registry.custom/aws
terraform force-unlock LOCK_ID             # Unlock stuck state
terraform refresh                          # Re-sync state with real infrastructure
```

## 4. Import & Move Resources

```
# Import existing infrastructure into state
terraform import aws_instance.web i-0abcd1234
terraform import aws_s3_bucket.data my-existing-bucket
terraform import 'aws_security_group.rule["https"]' sg-01234

# Generate config for existing resources (Terraform 1.5+)
terraform plan -generate-config-out=generated.tf
```

## 5. Workspaces

```
terraform workspace list        # List workspaces (* = current)
terraform workspace show        # Current workspace
terraform workspace new staging # Create + switch
terraform workspace select prod # Switch
terraform workspace delete staging
```

**Pattern:** use workspaces for identical environments, separate directories/backends for different environments.

## 6. Inspect & Debug

```
terraform output                    # All outputs
terraform output instance_ip        # Single output
terraform output -json              # JSON (for scripts)
terraform show                      # Human-readable state/plan
terraform show -json tfplan
terraform graph | dot -Tpng > graph.png    # Dependency graph
terraform providers                 # Show required providers
terraform version
terraform console                   # Interactive HCL expression evaluator
TF_LOG=DEBUG terraform apply        # Debug logging
TF_LOG_PATH=./debug.log terraform plan
```

## 7. HCL Basics — Variables, Locals, Outputs

```hcl
variable "instance_type" {
  type        = string
  default     = "t3.micro"
  description = "EC2 instance type"
  sensitive   = false
  validation {
    condition     = contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
    error_message = "Instance type must be t3.micro, t3.small, or t3.medium."
  }
}

variable "tags" {
  type = map(string)
  default = { Environment = "dev", ManagedBy = "terraform" }
}

variable "subnet_ids" {
  type = list(string)
}

locals {
  name_prefix = "${var.project}-${var.environment}"
  common_tags = merge(var.tags, { Project = var.project })
}

resource "aws_instance" "web" {
  tags = local.common_tags
}

output "instance_ip" {
  value = aws_instance.web.public_ip
}

output "db_password" {
  value     = var.db_password
  sensitive = true
}
```

**Variable definition precedence (low → high):** environment `TF_VAR_x` → `terraform.tfvars` → `*.auto.tfvars` (alphabetical) → `-var` / `-var-file` flags.

```bash
export TF_VAR_db_password="s3cret"      # env vars via TF_VAR_ prefix
terraform plan -var-file=prod.tfvars
```

## 8. Resources & Data Sources

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = { Name = "main-vpc" }

  lifecycle {
    create_before_destroy = true          # Zero-downtime replacements
    prevent_destroy       = true          # Safety guard
    ignore_changes        = [tags]        # Don't drift on specific attrs
  }
}

# Read existing infrastructure (no creation/management)
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]          # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
}
```

## 9. Modules

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.13.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
}

# Local module
module "app" {
  source = "./modules/app"
  env    = "prod"
}

output "vpc_id" {
  value = module.vpc.vpc_id
}
```

**Module sources:** registry (`namespace/name/provider`), Git (`git::https://...`), local paths, S3 buckets.

## 10. Meta-Arguments & Functions

```hcl
# count — multiple instances of the same resource
resource "aws_instance" "web" {
  count         = 3
  instance_type = "t3.micro"
  tags          = { Name = "web-${count.index}" }
}

# for_each — keyed instances
resource "aws_s3_bucket" "logs" {
  for_each = toset(["app", "audit", "access"])
  bucket   = "company-${each.key}-logs"
}

# Conditional
resource "aws_instance" "bastion" {
  count         = var.enable_bastion ? 1 : 0
  instance_type = "t3.micro"
}

# Common functions
locals {
  upper_env   = upper(var.environment)
  merged      = merge(var.tags, { Team = "devops" })
  flattened   = flatten([var.list_a, var.list_b])
  json_string = jsonencode({ name = "app", port = 8080 })
  parsed      = jsondecode(var.json_input)
  cidrs       = cidrsubnets("10.0.0.0/16", 8, 8, 8)   # ["10.0.0.0/24", "10.1.0.0/24", ...]
}
```

## 11. Provisioners

```hcl
resource "aws_instance" "web" {
  # ...

  provisioner "file" {
    source      = "./app.conf"
    destination = "/etc/app/app.conf"
    connection {
      type = "ssh"
      user = "ubuntu"
      host = self.public_ip
    }
  }

  provisioner "remote-exec" {
    inline = ["sudo apt-get update", "sudo apt-get install -y nginx"]
    connection {
      type = "ssh"
      user = "ubuntu"
      host = self.public_ip
    }
  }

  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> inventory.txt"
  }
}
```

> Prefer cloud-init/user_data, Packer images, or configuration tools (Ansible) over provisioners where possible.

## 12. Remote Backends

```hcl
terraform {
  required_version = ">= 1.6"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "prod/network/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"        # State locking
    encrypt        = true
  }
}
```

### CI/CD Pipeline Flow

```bash
terraform fmt -check -recursive
terraform init -backend-config=backend.hcl
terraform validate
terraform plan -out=tfplan -var-file=$ENV.tfvars
terraform apply -auto-approve tfplan      # after approval gate
```

---
