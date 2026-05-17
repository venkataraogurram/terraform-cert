# Terraform Associate — Sample HCL Snippets

Practical, copy-pasteable HCL examples covering the core exam objectives. Use these to practice reading and writing real configurations.

---

## 1. Project Skeleton

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }

  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "tf-state-lock"
  }
}

# providers.tf
provider "aws" {
  region = var.region

  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}
```

---

## 2. Variables — All Patterns

```hcl
# variables.tf

# Primitive
variable "region" {
  type        = string
  default     = "us-east-1"
  description = "AWS region for resources"
}

variable "instance_count" {
  type    = number
  default = 1
}

variable "enable_logging" {
  type    = bool
  default = true
}

# Collection types
variable "subnet_cidrs" {
  type    = list(string)
  default = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "tags" {
  type = map(string)
  default = {
    Owner = "platform"
    Team  = "infra"
  }
}

variable "allowed_cidrs" {
  type    = set(string)
  default = ["10.0.0.0/8", "172.16.0.0/12"]
}

# Structural types
variable "instance_config" {
  type = object({
    ami           = string
    instance_type = string
    key_name      = optional(string, "default-key")   # optional with default
    tags          = optional(map(string), {})
  })
}

variable "ingress_rules" {
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = []
}

# Sensitive
variable "db_password" {
  type      = string
  sensitive = true
}

# With validation
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }

  validation {
    condition     = length(var.environment) <= 10
    error_message = "Environment name must be 10 characters or less."
  }
}

# Nullable control
variable "extra_tag" {
  type     = string
  default  = null
  nullable = true
}
```

---

## 3. Outputs

```hcl
# outputs.tf

output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "Public IP of the web instance"
  value       = aws_instance.web.public_ip
}

output "subnet_ids" {
  description = "All subnet IDs"
  value       = aws_subnet.private[*].id
}

output "db_connection_string" {
  description = "Database connection string"
  value       = "postgres://${aws_db_instance.main.username}:${var.db_password}@${aws_db_instance.main.endpoint}/${aws_db_instance.main.db_name}"
  sensitive   = true
}

# Conditional output
output "cluster_endpoint" {
  description = "EKS cluster endpoint, if created"
  value       = var.create_cluster ? aws_eks_cluster.main[0].endpoint : null
}
```

---

## 4. Locals

```hcl
locals {
  # Computed values
  name_prefix = "${var.project}-${var.environment}"

  common_tags = merge(
    var.tags,
    {
      Project     = var.project
      Environment = var.environment
      Terraform   = "true"
      Timestamp   = timestamp()
    }
  )

  # Conditional values
  is_prod = var.environment == "prod"

  instance_type = local.is_prod ? "t3.large" : "t3.micro"

  # Filtered lists
  active_users = [for u in var.users : u if u.active]

  # Transformations
  user_emails = { for u in var.users : u.name => u.email }

  # Nested data
  subnet_pairs = flatten([
    for vpc_name, vpc in var.vpcs : [
      for cidr in vpc.subnets : {
        vpc_name = vpc_name
        cidr     = cidr
        az       = element(var.azs, index(vpc.subnets, cidr))
      }
    ]
  ])
}
```

---

## 5. `count` Patterns

```hcl
# Conditional resource
resource "aws_eip" "nat" {
  count  = var.enable_nat ? 1 : 0
  domain = "vpc"
}

# N copies
resource "aws_instance" "worker" {
  count         = var.worker_count
  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Name = "worker-${count.index + 1}"   # 1-indexed naming
  }
}

# Iterate over a list
resource "aws_subnet" "private" {
  count             = length(var.subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.subnet_cidrs[count.index]
  availability_zone = element(var.azs, count.index)

  tags = {
    Name = "private-${count.index}"
  }
}

# Reference to specific instance
output "first_worker_id" {
  value = aws_instance.worker[0].id
}

# Reference to all
output "all_worker_ids" {
  value = aws_instance.worker[*].id
}
```

---

## 6. `for_each` Patterns

```hcl
# Iterate over a set of strings
resource "aws_iam_user" "users" {
  for_each = toset(["alice", "bob", "carol"])
  name     = each.key
}

# Iterate over a map
resource "aws_iam_user" "users_with_paths" {
  for_each = {
    alice = "/admins/"
    bob   = "/devs/"
    carol = "/devs/"
  }

  name = each.key
  path = each.value
}

# Iterate over a list of objects (convert to map first)
variable "instances" {
  type = list(object({
    name = string
    type = string
    az   = string
  }))
}

resource "aws_instance" "named" {
  for_each = { for i in var.instances : i.name => i }

  ami               = var.ami_id
  instance_type     = each.value.type
  availability_zone = each.value.az

  tags = {
    Name = each.key
  }
}

# Reference specific instance
output "alice_id" {
  value = aws_iam_user.users["alice"].id
}

# Reference all
output "user_arns" {
  value = { for k, u in aws_iam_user.users : k => u.arn }
}
```

---

## 7. `dynamic` Blocks

```hcl
# Security group with dynamic ingress
resource "aws_security_group" "web" {
  name        = "${local.name_prefix}-web"
  description = "Web SG"
  vpc_id      = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.ingress_rules
    iterator = rule
    content {
      description = lookup(rule.value, "description", null)
      from_port   = rule.value.from_port
      to_port     = rule.value.to_port
      protocol    = rule.value.protocol
      cidr_blocks = rule.value.cidr_blocks
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = local.common_tags
}

# Conditional dynamic block (zero or one)
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  dynamic "root_block_device" {
    for_each = var.custom_root_volume ? [1] : []
    content {
      volume_size = 50
      volume_type = "gp3"
      encrypted   = true
    }
  }
}

# Nested dynamic blocks
resource "aws_lb_listener_rule" "complex" {
  listener_arn = aws_lb_listener.web.arn
  priority     = 100

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }

  dynamic "condition" {
    for_each = var.routing_rules
    iterator = rule
    content {
      dynamic "host_header" {
        for_each = rule.value.hosts != null ? [1] : []
        content {
          values = rule.value.hosts
        }
      }

      dynamic "path_pattern" {
        for_each = rule.value.paths != null ? [1] : []
        content {
          values = rule.value.paths
        }
      }
    }
  }
}
```

---

## 8. `lifecycle` Block — All Features

```hcl
# Zero-downtime replacement with name_prefix
resource "aws_security_group" "rotating" {
  name_prefix = "${local.name_prefix}-"
  vpc_id      = aws_vpc.main.id

  lifecycle {
    create_before_destroy = true
  }
}

# Production database with destroy protection
resource "aws_db_instance" "prod" {
  identifier        = "prod-db"
  engine            = "postgres"
  instance_class    = "db.t3.medium"
  allocated_storage = 100
  username          = "admin"
  password          = var.db_password

  lifecycle {
    prevent_destroy = true

    ignore_changes = [
      password,        # rotated externally
      engine_version,  # auto-updated
      final_snapshot_identifier,
    ]
  }
}

# Force replacement when SG changes
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  vpc_security_group_ids = [aws_security_group.web.id]

  lifecycle {
    create_before_destroy = true
    replace_triggered_by  = [aws_security_group.web]
  }
}

# Preconditions and postconditions
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  lifecycle {
    postcondition {
      condition     = self.architecture == "x86_64"
      error_message = "AMI must be x86_64."
    }
  }
}

resource "aws_instance" "guarded" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  lifecycle {
    precondition {
      condition     = data.aws_ami.amazon_linux.id != ""
      error_message = "AMI lookup must return a result."
    }

    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP."
    }
  }
}
```

---

## 9. Data Sources

```hcl
# Look up an AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]   # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# AWS account info
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}
data "aws_availability_zones" "available" {
  state = "available"
}

# Existing VPC
data "aws_vpc" "default" {
  default = true
}

# IAM policy document
data "aws_iam_policy_document" "s3_access" {
  statement {
    effect    = "Allow"
    actions   = ["s3:GetObject", "s3:PutObject"]
    resources = ["arn:aws:s3:::my-bucket/*"]

    principals {
      type        = "AWS"
      identifiers = [data.aws_caller_identity.current.account_id]
    }

    condition {
      test     = "StringEquals"
      variable = "aws:RequestedRegion"
      values   = [data.aws_region.current.name]
    }
  }
}

# Read another stack's state
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-tf-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.network.outputs.private_subnet_ids[0]
  # ...
}
```

---

## 10. Modules

```hcl
# Local module
module "vpc" {
  source = "./modules/vpc"

  cidr_block = "10.0.0.0/16"
  azs        = ["us-east-1a", "us-east-1b"]
  tags       = local.common_tags
}

# Registry module
module "vpc_public" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "main-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true

  tags = local.common_tags
}

# Git source with version pinning
module "compute" {
  source = "git::https://github.com/myorg/tf-modules.git//compute?ref=v1.2.3"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
}

# Module with for_each
module "buckets" {
  source   = "./modules/s3-bucket"
  for_each = var.bucket_configs

  name       = each.key
  versioning = each.value.versioning
  lifecycle  = each.value.lifecycle_rules
}

# Pass alternate provider
module "west_replica" {
  source = "./modules/replica"

  providers = {
    aws = aws.west
  }

  primary_db_arn = aws_db_instance.main.arn
}
```

### Inside a module

```hcl
# modules/vpc/variables.tf
variable "cidr_block" {
  type = string
}

variable "azs" {
  type = list(string)
}

variable "tags" {
  type    = map(string)
  default = {}
}

# modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block = var.cidr_block

  tags = merge(var.tags, {
    Name = "module-vpc"
  })
}

resource "aws_subnet" "private" {
  count             = length(var.azs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = cidrsubnet(var.cidr_block, 8, count.index)
  availability_zone = var.azs[count.index]
}

# modules/vpc/outputs.tf
output "vpc_id" {
  value = aws_vpc.this.id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}

# modules/vpc/versions.tf
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}
```

---

## 11. Provisioners

```hcl
# local-exec
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  provisioner "local-exec" {
    command = "echo '${self.id} ${self.public_ip}' >> instances.txt"
  }

  # Destroy-time
  provisioner "local-exec" {
    when    = destroy
    command = "echo 'Destroying ${self.id}'"
  }
}

# remote-exec
resource "aws_instance" "configured" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  key_name      = var.key_name

  connection {
    type        = "ssh"
    user        = "ec2-user"
    host        = self.public_ip
    private_key = file("~/.ssh/${var.key_name}.pem")
  }

  provisioner "file" {
    source      = "${path.module}/scripts/setup.sh"
    destination = "/tmp/setup.sh"
  }

  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/setup.sh",
      "sudo /tmp/setup.sh",
    ]
  }
}

# terraform_data (replaces null_resource in 1.4+)
resource "terraform_data" "deploy" {
  triggers_replace = [var.app_version]

  provisioner "local-exec" {
    command = "./deploy.sh ${var.app_version}"
  }
}

# null_resource (still works, but terraform_data is preferred)
resource "null_resource" "trigger" {
  triggers = {
    config_hash = filemd5("${path.module}/config.json")
  }

  provisioner "local-exec" {
    command = "./regenerate.sh"
  }
}
```

---

## 12. Conditional Patterns

```hcl
# Conditional resource creation
resource "aws_eip" "lb" {
  count    = var.create_eip ? 1 : 0
  instance = aws_instance.web.id
}

# Conditional value
locals {
  instance_type = var.environment == "prod" ? "m5.large" : "t3.micro"

  backup_retention = (
    var.environment == "prod" ? 30 :
    var.environment == "staging" ? 7 :
    1
  )
}

# Conditional module
module "monitoring" {
  source = "./modules/monitoring"
  count  = var.enable_monitoring ? 1 : 0

  resource_arns = [aws_instance.web.arn]
}

# Conditional output (handles count)
output "monitoring_endpoint" {
  value = var.enable_monitoring ? module.monitoring[0].endpoint : null
}

# Try/can for safe lookups
locals {
  region = try(var.config.region, "us-east-1")

  has_db = can(var.config.database.host)
}
```

---

## 13. Functions in Practice

```hcl
locals {
  # String functions
  upper_env    = upper(var.environment)
  short_region = substr(var.region, 0, 2)
  formatted    = format("%s-%s-%03d", var.project, var.environment, var.index)

  # Collection functions
  unique_tags  = distinct(concat(var.tag_a, var.tag_b))
  total_cidrs  = length(var.subnet_cidrs)
  first_three  = slice(var.azs, 0, 3)

  # Map functions
  all_tags = merge(
    var.default_tags,
    var.environment_tags,
    { Name = local.upper_env }
  )

  # CIDR math
  vpc_cidr      = "10.0.0.0/16"
  subnet_cidrs  = [for i in range(4) : cidrsubnet(local.vpc_cidr, 8, i)]
  first_host    = cidrhost(local.vpc_cidr, 1)
  netmask       = cidrnetmask(local.vpc_cidr)

  # Encoding
  user_data_b64 = base64encode(templatefile("${path.module}/init.sh", {
    role = var.role
  }))

  # JSON for IAM
  bucket_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "AllowList"
        Effect    = "Allow"
        Principal = "*"
        Action    = "s3:ListBucket"
        Resource  = "arn:aws:s3:::${var.bucket_name}"
      },
    ]
  })

  # Filesystem
  config        = jsondecode(file("${path.module}/config.json"))
  template_data = templatefile("${path.module}/template.tftpl", { vars = local })
  all_scripts   = fileset("${path.module}/scripts", "*.sh")

  # Date
  timestamp_iso = formatdate("YYYY-MM-DD'T'hh:mm:ss", timestamp())

  # Type conversion
  port_string = tostring(var.port)
  cidrs_set   = toset(var.subnet_cidrs)
  pairs_map   = tomap({ key = "value" })
}
```

---

## 14. `templatefile` Example

```
# templates/init.sh.tftpl
#!/bin/bash
set -e

echo "Configuring ${role} on ${hostname}"

%{ if enable_monitoring }
yum install -y amazon-cloudwatch-agent
systemctl start amazon-cloudwatch-agent
%{ endif }

%{ for user in users ~}
useradd ${user}
%{ endfor ~}

cat > /etc/myapp/config.json <<EOF
${jsonencode(app_config)}
EOF
```

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  user_data = templatefile("${path.module}/templates/init.sh.tftpl", {
    role              = "web"
    hostname          = "web-01"
    enable_monitoring = true
    users             = ["alice", "bob"]
    app_config = {
      port    = 8080
      workers = 4
    }
  })
}
```

---

## 15. Real-World Mini-Stack

A small but complete VPC + EC2 deployment:

```hcl
# main.tf

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${local.name_prefix}-vpc"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${local.name_prefix}-igw"
  }
}

resource "aws_subnet" "public" {
  for_each = {
    for idx, az in var.azs : az => {
      cidr  = cidrsubnet(aws_vpc.main.cidr_block, 8, idx)
      index = idx
    }
  }

  vpc_id                  = aws_vpc.main.id
  cidr_block              = each.value.cidr
  availability_zone       = each.key
  map_public_ip_on_launch = true

  tags = {
    Name = "${local.name_prefix}-public-${each.key}"
    Tier = "public"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${local.name_prefix}-public"
  }
}

resource "aws_route_table_association" "public" {
  for_each = aws_subnet.public

  subnet_id      = each.value.id
  route_table_id = aws_route_table.public.id
}

resource "aws_security_group" "web" {
  name_prefix = "${local.name_prefix}-web-"
  description = "Web SG"
  vpc_id      = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.ingress_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  lifecycle {
    create_before_destroy = true
  }
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_instance" "web" {
  count = var.web_count

  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = local.instance_type
  subnet_id              = values(aws_subnet.public)[count.index % length(aws_subnet.public)].id
  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = templatefile("${path.module}/templates/init.sh.tftpl", {
    role  = "web"
    index = count.index
  })

  tags = {
    Name = "${local.name_prefix}-web-${count.index + 1}"
  }

  lifecycle {
    create_before_destroy = true
    ignore_changes        = [ami]   # don't replace on AMI updates
  }
}

# outputs.tf

output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = { for k, s in aws_subnet.public : k => s.id }
}

output "web_instances" {
  value = {
    for i, instance in aws_instance.web : i => {
      id        = instance.id
      public_ip = instance.public_ip
    }
  }
}
```

---

## 16. Imports (1.5+)

```hcl
# Bring an existing VPC into state declaratively
import {
  to = aws_vpc.main
  id = "vpc-0abc1234"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "imported-vpc"
  }
}
```

```bash
# Auto-generate the resource block
terraform plan -generate-config-out=imported.tf

# Apply to record the import in state
terraform apply
```

For older Terraform:
```bash
# Add the resource block manually first, then:
terraform import aws_vpc.main vpc-0abc1234
```

---

## 17. Backend Migration

```hcl
# Old config (local backend)
terraform {
  backend "local" {
    path = "terraform.tfstate"
  }
}

# Change to S3
terraform {
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "tf-state-lock"
  }
}
```

```bash
# Migrate the existing state to the new backend
terraform init -migrate-state

# Or start fresh (don't copy old state)
terraform init -reconfigure
```

---

## 18. Workspace Patterns

```hcl
# Use workspace name in resource names
resource "aws_s3_bucket" "data" {
  bucket = "my-app-${terraform.workspace}-data"
}

# Workspace-specific behavior
locals {
  is_prod = terraform.workspace == "prod"

  instance_type = local.is_prod ? "m5.large" : "t3.micro"

  backup_count = {
    default = 1
    dev     = 1
    staging = 7
    prod    = 30
  }

  retention = lookup(local.backup_count, terraform.workspace, 1)
}
```

```bash
terraform workspace new dev
terraform workspace new prod
terraform workspace select prod
terraform apply
```

---

## 19. Common Recipes

### S3 bucket with versioning, encryption, logging
```hcl
resource "aws_s3_bucket" "data" {
  bucket = "${local.name_prefix}-data"
}

resource "aws_s3_bucket_versioning" "data" {
  bucket = aws_s3_bucket.data.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "data" {
  bucket = aws_s3_bucket.data.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### IAM role with assume policy
```hcl
data "aws_iam_policy_document" "assume_role" {
  statement {
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "ec2" {
  name               = "${local.name_prefix}-ec2-role"
  assume_role_policy = data.aws_iam_policy_document.assume_role.json
}

resource "aws_iam_role_policy_attachment" "ssm" {
  role       = aws_iam_role.ec2.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_instance_profile" "ec2" {
  name = aws_iam_role.ec2.name
  role = aws_iam_role.ec2.name
}
```

### Random suffixes for unique names
```hcl
resource "random_id" "suffix" {
  byte_length = 4
}

resource "aws_s3_bucket" "unique" {
  bucket = "${local.name_prefix}-${random_id.suffix.hex}"
}
```

---

## 20. Practice Challenges

Try these to test yourself:

### Challenge 1
Write a configuration that creates one EC2 instance per element of `var.servers` (a `map(object)`), using `for_each` and stable identity by name.

### Challenge 2
Convert this `count`-based block to `for_each`:
```hcl
resource "aws_iam_user" "this" {
  count = length(var.users)
  name  = var.users[count.index]
}
```

### Challenge 3
Generate a list of 6 subnets across 3 AZs (2 per AZ — one public, one private) using `flatten` and nested `for`.

### Challenge 4
Write a `dynamic` block that conditionally adds an `ingress` rule for SSH (port 22) only if `var.allow_ssh` is true.

### Challenge 5
Create a module that accepts a `map(string)` of bucket names → regions, and creates one bucket in each region using provider aliases.

### Challenge 6
Write the locals + resources to take this input and create the right number of subnets:
```hcl
variable "vpcs" {
  default = {
    prod = ["10.0.1.0/24", "10.0.2.0/24"]
    dev  = ["10.1.1.0/24"]
  }
}
```

### Challenge 7
Write a `lifecycle` block that:
- Creates new resources before destroying old ones
- Replaces this resource if the security group changes
- Ignores changes to `tags`

### Challenge 8
Write the steps (commands) to:
1. Migrate state from local to S3 backend
2. Recover state if `terraform.tfstate` is deleted
3. Detect drift in CI with `-detailed-exitcode`

---

## Solutions to Practice Challenges

### Challenge 1
```hcl
variable "servers" {
  type = map(object({
    instance_type = string
    az            = string
  }))
}

resource "aws_instance" "this" {
  for_each = var.servers

  ami               = data.aws_ami.amazon_linux.id
  instance_type     = each.value.instance_type
  availability_zone = each.value.az

  tags = {
    Name = each.key
  }
}
```

### Challenge 2
```hcl
resource "aws_iam_user" "this" {
  for_each = toset(var.users)
  name     = each.key
}
```

### Challenge 3
```hcl
locals {
  azs = ["us-east-1a", "us-east-1b", "us-east-1c"]

  subnets = flatten([
    for idx, az in local.azs : [
      {
        cidr = cidrsubnet("10.0.0.0/16", 8, idx * 2)
        az   = az
        tier = "public"
      },
      {
        cidr = cidrsubnet("10.0.0.0/16", 8, idx * 2 + 1)
        az   = az
        tier = "private"
      },
    ]
  ])
}

resource "aws_subnet" "this" {
  for_each = { for s in local.subnets : "${s.az}-${s.tier}" => s }

  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az

  tags = {
    Name = each.key
    Tier = each.value.tier
  }
}
```

### Challenge 4
```hcl
resource "aws_security_group" "web" {
  # ...

  dynamic "ingress" {
    for_each = var.allow_ssh ? [1] : []
    content {
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/8"]
    }
  }
}
```

### Challenge 5
```hcl
# variables.tf
variable "buckets" {
  type = map(string)   # name => region
}

# main.tf — needs one provider per region
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

provider "aws" {
  alias  = "us_west_2"
  region = "us-west-2"
}

# Build a map of buckets by region
locals {
  by_region = {
    for name, region in var.buckets : name => {
      name   = name
      region = region
    }
  }

  east_buckets = { for k, v in local.by_region : k => v if v.region == "us-east-1" }
  west_buckets = { for k, v in local.by_region : k => v if v.region == "us-west-2" }
}

resource "aws_s3_bucket" "east" {
  provider = aws.us_east_1
  for_each = local.east_buckets
  bucket   = each.key
}

resource "aws_s3_bucket" "west" {
  provider = aws.us_west_2
  for_each = local.west_buckets
  bucket   = each.key
}
```

### Challenge 6
```hcl
locals {
  subnets = flatten([
    for vpc, cidrs in var.vpcs : [
      for cidr in cidrs : {
        vpc  = vpc
        cidr = cidr
      }
    ]
  ])
}

resource "aws_subnet" "this" {
  for_each = { for s in local.subnets : "${s.vpc}-${s.cidr}" => s }

  vpc_id     = aws_vpc.this[each.value.vpc].id
  cidr_block = each.value.cidr
}
```

### Challenge 7
```hcl
lifecycle {
  create_before_destroy = true
  replace_triggered_by  = [aws_security_group.web]
  ignore_changes        = [tags]
}
```

### Challenge 8
```bash
# 1. Migrate state from local to S3
# (after updating backend config in code)
terraform init -migrate-state

# 2. Recover state from backup
cp terraform.tfstate.backup terraform.tfstate
terraform plan   # verify no diff

# Or from remote backend (no command needed if using one):
terraform init   # pulls from backend

# 3. Drift detection in CI
terraform plan -refresh-only -detailed-exitcode
echo "Exit: $?"
# 0 = clean, 1 = error, 2 = drift
```

---

Use these snippets as a reference and as practice — read them carefully, type them out, and modify them. Recognizing real HCL is half the exam.
