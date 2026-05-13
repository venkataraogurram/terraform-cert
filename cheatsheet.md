# Terraform Associate Exam — Detailed Cheatsheet

---

## 1. Infrastructure as Code (IaC) Fundamentals

### What is IaC?
- Manage infrastructure through code instead of manual processes
- Enables version control, repeatability, collaboration, and automation

### IaC approaches
| Approach | Description | Example |
|---|---|---|
| **Declarative** | Define desired end state | Terraform, CloudFormation |
| **Imperative** | Define exact steps to reach state | Ansible (mostly), scripts |

### Terraform advantages
- Cloud-agnostic (works with 1000+ providers)
- Declarative HCL syntax
- Execution plan before applying changes
- Resource graph for parallel operations
- State management
- Module system for reuse

### Terraform vs other tools
| Tool | Type | Language |
|---|---|---|
| Terraform | Declarative IaC | HCL / JSON |
| Ansible | Configuration mgmt (imperative) | YAML |
| CloudFormation | Declarative IaC (AWS only) | YAML / JSON |
| Pulumi | IaC | Python, TS, Go, etc. |
| Chef/Puppet | Configuration mgmt | Ruby DSL |

---

## 2. Terraform Architecture

```
┌──────────────────────────────────────────────┐
│               Terraform Core                 │
│  - Reads config files (.tf)                  │
│  - Manages state                             │
│  - Builds resource graph                     │
│  - Communicates with providers               │
└────────────────┬─────────────────────────────┘
                 │ RPC (gRPC)
     ┌───────────┼──────────────┐
     ▼           ▼              ▼
 AWS Provider  GCP Provider  Azure Provider
 (plugin)      (plugin)      (plugin)
```

### Terraform binaries
- Single binary written in Go
- Providers are separate binaries, downloaded during `terraform init`
- Plugins stored in `.terraform/providers/`

---

## 3. Terraform Workflow (Detailed)

```
Write → Init → Validate → Plan → Apply → Destroy
```

### Stage details

| Stage | Command | What happens |
|---|---|---|
| Write | — | Author `.tf` files |
| Init | `terraform init` | Downloads providers, modules, sets up backend |
| Validate | `terraform validate` | Syntax + logic check (no API calls) |
| Plan | `terraform plan` | Diff current state vs desired state |
| Apply | `terraform apply` | Execute the plan |
| Destroy | `terraform destroy` | Destroy all managed resources |

### Key flags

```bash
terraform init -upgrade              # upgrade providers to latest allowed
terraform init -backend=false        # skip backend initialization
terraform init -reconfigure          # reconfigure backend (ignore existing state)
terraform init -migrate-state        # migrate state to new backend

terraform plan -out=tfplan           # save plan to file
terraform plan -var="key=value"      # pass variable inline
terraform plan -var-file="prod.tfvars"
terraform plan -target=aws_instance.web   # plan only specific resource
terraform plan -refresh=false        # skip state refresh
terraform plan -destroy              # show destroy plan

terraform apply tfplan               # apply saved plan
terraform apply -auto-approve        # skip interactive approval
terraform apply -replace=aws_instance.web  # force replace (replaces taint)
terraform apply -parallelism=10      # default is 10 concurrent operations

terraform destroy -target=aws_instance.web
terraform destroy -auto-approve
```

---

## 4. HCL (HashiCorp Configuration Language)

### Basic syntax rules
- Files end in `.tf` or `.tf.json`
- Blocks, arguments, and expressions are the three main constructs
- Comments: `#`, `//` (single line), `/* ... */` (multi-line)
- Strings use double quotes only
- Multiline strings use heredoc syntax

### Heredoc
```hcl
value = <<-EOT
  Hello
  World
EOT
```

### String interpolation
```hcl
"Hello, ${var.name}!"
"${var.env}-${var.region}-bucket"
```

### String directives (template syntax)
```hcl
"%{for item in var.list}${item}%{endfor}"
"%{if var.env == "prod"}production%{else}non-prod%{endif}"
```

---

## 5. Resource Configuration (Deep Dive)

### Full resource syntax
```hcl
resource "PROVIDER_TYPE" "LOCAL_NAME" {
  # arguments
  
  timeouts {
    create = "30m"
    update = "20m"
    delete = "10m"
  }

  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
    ignore_changes        = [tags, ami]
    replace_triggered_by  = [aws_security_group.web]
    precondition {
      condition     = var.instance_type != "t2.micro"
      error_message = "Must not use t2.micro in prod."
    }
    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP."
    }
  }

  provisioner "local-exec" {
    command = "echo ${self.id}"
  }
}
```

### Resource behavior on changes
| Change type | Behavior |
|---|---|
| In-place update | Resource updated without recreation |
| Destroy and recreate | Resource deleted then created (disrupts service) |
| `create_before_destroy` | New resource created first, then old one deleted |
| `prevent_destroy` | Plan fails if resource would be destroyed |

### Referencing resource attributes
```hcl
resource "aws_instance" "web" { ... }

# Reference in same config
aws_instance.web.id
aws_instance.web.public_ip

# After count
aws_instance.web[0].id
aws_instance.web[*].id    # all instances (splat)

# After for_each
aws_instance.web["us-east-1"].id
```

---

## 6. Variables (Deep Dive)

### Variable declaration options
```hcl
variable "image_id" {
  type        = string
  description = "The ID of the AMI to use"
  default     = "ami-abc123"
  sensitive   = true
  nullable    = false    # disallow null values

  validation {
    condition     = can(regex("^ami-", var.image_id))
    error_message = "Must be a valid AMI ID starting with ami-."
  }
}
```

### Variable type constraints
```hcl
# Primitives
type = string
type = number
type = bool

# Collections
type = list(string)
type = set(number)
type = map(string)

# Structural
type = object({
  name    = string
  age     = number
  enabled = bool
})

type = tuple([string, number, bool])

# Any
type = any
```

### Variable precedence (lowest to highest)
1. `default` in variable block
2. `terraform.tfvars` (auto-loaded)
3. `terraform.tfvars.json` (auto-loaded)
4. `*.auto.tfvars` (alphabetical order)
5. `*.auto.tfvars.json` (alphabetical order)
6. `-var-file` flag (command line, left to right)
7. `-var` flag (command line, left to right)
8. `TF_VAR_<name>` environment variables

### Accessing variables
```hcl
var.image_id
var.settings.name       # object attribute
var.list[0]             # list index
var.map["key"]          # map key
```

---

## 7. Outputs (Deep Dive)

```hcl
output "instance_ip" {
  value       = aws_instance.web.public_ip
  description = "Public IP of web instance"
  sensitive   = true    # hide from CLI output

  precondition {
    condition     = aws_instance.web.public_ip != ""
    error_message = "Instance must have a public IP."
  }

  depends_on = [aws_security_group.web]
}
```

- Outputs are shown after `terraform apply`
- `terraform output` — list all outputs
- `terraform output instance_ip` — specific output
- `terraform output -json` — JSON format
- `terraform output -raw instance_ip` — raw string (no quotes)
- Used by child modules to expose values to parent
- Used by `terraform_remote_state` data source

---

## 8. Locals

```hcl
locals {
  common_tags = {
    Environment = var.env
    Project     = var.project
    ManagedBy   = "Terraform"
  }

  instance_name = "${var.env}-${var.project}-web"
  
  # Can reference other locals
  full_name = "${local.instance_name}-v2"
}

# Usage
resource "aws_instance" "web" {
  tags = local.common_tags
}
```

- Avoid repetition; computed values from expressions
- Cannot be overridden by user (unlike variables)

---

## 9. Data Sources

```hcl
data "aws_ami" "latest_amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Reference
resource "aws_instance" "web" {
  ami = data.aws_ami.latest_amazon_linux.id
}
```

- Read-only — fetches information from provider
- Refreshed on every plan/apply
- Use `depends_on` if the data source depends on a managed resource

### terraform_remote_state
```hcl
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "my-tf-state"
    key    = "vpc/terraform.tfstate"
    region = "us-east-1"
  }
}

# Access outputs from that state
data.terraform_remote_state.vpc.outputs.vpc_id
```

---

## 10. Meta-Arguments (Deep Dive)

### count
```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-abc123"
  instance_type = "t3.micro"
  tags = {
    Name = "web-${count.index}"
  }
}

# Reference
aws_instance.web[0].id
aws_instance.web[*].id    # list of all IDs
```

### for_each
```hcl
# With a set
resource "aws_iam_user" "user" {
  for_each = toset(["alice", "bob", "carol"])
  name     = each.key
}

# With a map
resource "aws_instance" "web" {
  for_each      = {
    prod = "t3.large"
    dev  = "t3.micro"
  }
  instance_type = each.value
  tags          = { Name = each.key }
}

# Reference
aws_instance.web["prod"].id
```

### count vs for_each
| | count | for_each |
|---|---|---|
| Index type | Integer | String key |
| On removal | All later indices shift (can cause destroy/recreate) | Only removed item is destroyed |
| Use when | Identical resources | Resources with distinct identities |
| Data type | Number | Map or Set |

### depends_on
```hcl
resource "aws_instance" "web" {
  depends_on = [
    aws_iam_role_policy.example,
    module.vpc
  ]
}
```
- Use only when dependency is not expressible through references
- Creates an explicit edge in the resource graph

### provider (for multiple provider configs)
```hcl
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}

resource "aws_instance" "web_eu" {
  provider = aws.eu_west
  ...
}
```

---

## 11. Dynamic Blocks

```hcl
variable "ingress_rules" {
  type = list(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
  }))
}

resource "aws_security_group" "web" {
  name = "web-sg"

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

- `dynamic` block name matches the nested block type
- `for_each` can be a list, map, or set
- Access value with `<label>.value`, key with `<label>.key`
- Use `iterator` to rename the iterator variable:

```hcl
dynamic "ingress" {
  for_each = var.ingress_rules
  iterator = rule
  content {
    from_port = rule.value.port
  }
}
```

---

## 12. Expressions & Functions (Deep Dive)

### Type conversion
```hcl
tostring(42)         # "42"
tonumber("42")       # 42
tobool("true")       # true
tolist(toset([...]))
toset([...])
tomap({...})
```

### String functions
```hcl
length("hello")              # 5
upper("hello")               # "HELLO"
lower("HELLO")               # "hello"
title("hello world")         # "Hello World"
trimspace("  hello  ")       # "hello"
trim("!!hello!!", "!")        # "hello"
trimprefix("hello-world", "hello-")  # "world"
trimsuffix("hello-world", "-world")  # "hello"
replace("hello world", " ", "-")     # "hello-world"
split(",", "a,b,c")          # ["a","b","c"]
join(",", ["a","b","c"])     # "a,b,c"
contains(["a","b"], "a")     # true
startswith("hello", "he")    # true
endswith("hello", "lo")      # true
substr("hello", 0, 3)        # "hel"
format("Hello, %s! You are %d.", "Alice", 30)
formatlist("Hello, %s!", ["Alice","Bob"])
indent(4, "hello\nworld")
```

### Collection functions
```hcl
length(["a","b","c"])         # 3
element(["a","b","c"], 1)     # "b"
slice(["a","b","c","d"], 1, 3)  # ["b","c"]
concat(["a"], ["b","c"])      # ["a","b","c"]
flatten([["a","b"],["c"]])    # ["a","b","c"]
distinct(["a","b","a"])       # ["a","b"]
compact(["a","","b"])         # ["a","b"]
reverse(["a","b","c"])        # ["c","b","a"]
sort(["c","a","b"])           # ["a","b","c"]
index(["a","b","c"], "b")     # 1
contains(["a","b"], "a")      # true

# Map functions
keys({a=1,b=2})               # ["a","b"]
values({a=1,b=2})             # [1,2]
lookup({a=1,b=2}, "a", 0)     # 1
merge({a=1},{b=2})            # {a=1,b=2}
zipmap(["a","b"],[1,2])       # {a=1,b=2}
tomap({a="1",b="2"})

# Set operations
setintersection(["a","b"],["b","c"])   # ["b"]
setunion(["a","b"],["b","c"])          # ["a","b","c"]
setsubtract(["a","b"],["b"])           # ["a"]
```

### Numeric functions
```hcl
abs(-5)              # 5
ceil(1.2)            # 2
floor(1.9)           # 1
max(1,2,3)           # 3
min(1,2,3)           # 1
pow(2,3)             # 8
signum(-5)           # -1
log(8,2)             # 3
```

### Encoding functions
```hcl
base64encode("hello")
base64decode("aGVsbG8=")
jsonencode({key = "value"})
jsondecode("{\"key\":\"value\"}")
yamlencode({key = "value"})
yamldecode("key: value")
urlencode("hello world")
```

### Filesystem functions
```hcl
file("path/to/file")
filebase64("path/to/file")
fileexists("path/to/file")    # bool
fileset("dir", "*.txt")       # set of matching files
templatefile("tmpl.tftpl", { name = "Alice" })
```

### Hash/crypto functions
```hcl
md5("hello")
sha1("hello")
sha256("hello")
bcrypt("password")
uuid()
uuidv5("dns", "example.com")
```

### IP network functions
```hcl
cidrhost("10.0.0.0/16", 5)           # "10.0.0.5"
cidrnetmask("10.0.0.0/16")           # "255.255.0.0"
cidrsubnet("10.0.0.0/16", 8, 1)      # "10.0.1.0/24"
cidrsubnets("10.0.0.0/16", 4, 4, 8)  # list of subnets
cidrcontains("10.0.0.0/8","10.1.2.3") # true
```

### Date/time functions
```hcl
timestamp()                           # current UTC time (RFC 3339)
timeadd("2024-01-01T00:00:00Z", "24h")
timecmp("2024-01-01T00:00:00Z", "2024-01-02T00:00:00Z")  # -1
formatdate("DD MMM YYYY", timestamp())
```

### Type-checking functions
```hcl
can(tonumber("abc"))     # false — no error, returns bool
try(tonumber("abc"), 0)  # 0 — returns fallback on error
type(42)                 # "number"
```

### Conditional expression
```hcl
condition ? true_value : false_value

var.env == "prod" ? 3 : 1
var.name != "" ? var.name : "default"
```

### for expression
```hcl
# List comprehension
[for s in var.list : upper(s)]
[for i, s in var.list : "${i}: ${s}"]

# Map comprehension
{for k, v in var.map : k => upper(v)}
{for s in var.list : s => length(s)}

# With filter (if clause)
[for s in var.list : upper(s) if s != "skip"]

# Nested for
flatten([for r in var.regions : [for z in r.zones : z]])
```

### splat expression
```hcl
# Legacy splat (lists only)
aws_instance.web.*.public_ip

# Full splat (works with any type)
aws_instance.web[*].public_ip
```

---

## 13. Providers (Deep Dive)

### Provider configuration
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"   # registry.terraform.io/hashicorp/aws
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = ">= 3.0"
    }
  }
  required_version = ">= 1.5.0"
}

provider "aws" {
  region  = "us-east-1"
  profile = "dev"
  
  default_tags {
    tags = {
      ManagedBy = "Terraform"
    }
  }
}
```

### Provider sources
- `hashicorp/aws` = `registry.terraform.io/hashicorp/aws`
- Registry tiers:
  - **Official** — maintained by HashiCorp (`hashicorp/`)
  - **Partner** — maintained by technology partners (verified badge)
  - **Community** — maintained by community

### Multiple provider instances (alias)
```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

# Default provider (no alias)
resource "aws_vpc" "east" { ... }

# Named provider
resource "aws_vpc" "west" {
  provider = aws.west
  ...
}

# In modules
module "vpc" {
  source = "./modules/vpc"
  providers = {
    aws = aws.west
  }
}
```

### Provider version constraints
```
= 5.0.0   → exactly 5.0.0
!= 5.0.0  → anything except 5.0.0
> 5.0.0   → greater than 5.0.0
>= 5.0.0  → 5.0.0 or greater
< 5.0.0   → less than 5.0.0
<= 5.0.0  → 5.0.0 or less
~> 5.0    → >= 5.0, < 6.0  (patch-level flexibility)
~> 5.0.0  → >= 5.0.0, < 5.1.0  (patch only)
```

---

## 14. State (Deep Dive)

### What state contains
- Maps Terraform config to real-world resources
- Tracks metadata (dependencies, resource IDs)
- Caches resource attributes for performance
- Stored in `terraform.tfstate` (JSON format)

### State commands
```bash
terraform state list                         # list all resources
terraform state list aws_instance.*          # filter by type
terraform state show aws_instance.web        # show resource attributes
terraform state mv aws_instance.web aws_instance.app   # rename
terraform state mv module.old module.new     # rename module
terraform state rm aws_instance.web          # remove from state (resource stays)
terraform state pull                         # fetch and print current state
terraform state push terraform.tfstate       # upload state to backend
terraform state replace-provider old new     # change provider source
terraform force-unlock LOCK_ID               # manually release lock
```

### Sensitive state data
- State always contains sensitive values in plaintext
- Protect state file with:
  - Encryption at rest (S3 + SSE, Terraform Cloud)
  - Access controls (IAM, Terraform Cloud teams)
  - Avoid storing in version control

### Refreshing state
```bash
terraform refresh      # updates state to match real-world (deprecated in favor of -refresh-only)
terraform apply -refresh-only   # preferred: update state without making changes
terraform plan -refresh=false   # skip refresh (use cached state)
```

---

## 15. Backends (Deep Dive)

### Default backend (local)
```hcl
terraform {
  backend "local" {
    path = "relative/path/to/terraform.tfstate"
  }
}
```

### S3 backend (most common)
```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state-bucket"
    key            = "prod/us-east-1/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true             # encrypt state at rest
    dynamodb_table = "tf-state-lock"  # state locking
    kms_key_id     = "arn:aws:kms:..." # custom KMS key

    # Role-based access
    role_arn = "arn:aws:iam::123456789:role/TerraformRole"
  }
}
```

### DynamoDB table for locking
- Must have partition key named `LockID` (String type)
- Prevents concurrent state modification

### Terraform Cloud / HCP Terraform backend
```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "my-workspace"
    }
  }
}
```

### Partial backend configuration
```hcl
# In config (leave sensitive values out)
terraform {
  backend "s3" {
    bucket = "my-state-bucket"
    region = "us-east-1"
  }
}
```
```bash
# Pass remaining values at init time
terraform init -backend-config="key=prod/terraform.tfstate" \
               -backend-config="dynamodb_table=tf-lock"
```

### Backend migration
```bash
terraform init -migrate-state    # move state to new backend
terraform init -reconfigure      # ignore existing state, start fresh
```

### Supported backends
`local`, `s3`, `azurerm`, `gcs`, `consul`, `http`, `kubernetes`, `pg` (PostgreSQL), `oss`, `remote` (Terraform Cloud)

---

## 16. Modules (Deep Dive)

### Module structure
```
modules/
  vpc/
    main.tf
    variables.tf
    outputs.tf
    README.md
    versions.tf
```

### Module call
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"             # required for registry modules

  # Pass input variables
  name = "my-vpc"
  cidr = "10.0.0.0/16"

  # Override providers
  providers = {
    aws = aws.us_east
  }

  # Explicit dependency
  depends_on = [aws_internet_gateway.main]
}
```

### Module sources
```hcl
source = "./modules/vpc"                          # local path
source = "../shared/modules/vpc"                  # relative path
source = "terraform-aws-modules/vpc/aws"          # Terraform Registry
source = "hashicorp/consul/aws"                   # Registry (explicit)
source = "github.com/org/repo//modules/vpc"       # GitHub
source = "github.com/org/repo//modules/vpc?ref=v1.0"  # pinned ref
source = "git::https://example.com/repo.git"      # generic git
source = "git::ssh://git@github.com/org/repo.git//modules/vpc"
source = "s3::https://s3.amazonaws.com/bucket/module.zip"
source = "https://example.com/module.zip"
```

### Accessing module outputs
```hcl
module.vpc.vpc_id
module.vpc.public_subnets
```

### Module versioning (Registry)
- Pin versions in production: `version = "5.0.0"`
- Use constraints for flexibility: `version = "~> 5.0"`
- `terraform init -upgrade` updates to latest matching version

### Root vs child modules
- **Root module** = current working directory (entry point)
- **Child module** = called via `module` block
- **Published module** = shared on Terraform Registry
- Modules can call other modules (nested modules)

### Module best practices
- Always declare `versions.tf` with `required_providers`
- Always declare `variables.tf` and `outputs.tf`
- Use `description` on all variables and outputs
- Avoid hardcoding provider config inside modules (pass via `providers`)

---

## 17. Workspaces

```bash
terraform workspace new prod
terraform workspace new dev
terraform workspace select prod
terraform workspace list
terraform workspace show       # current workspace name
terraform workspace delete dev
```

### Using workspace in config
```hcl
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"
  tags = {
    Environment = terraform.workspace
  }
}
```

### State storage per workspace
- Local backend: `terraform.tfstate.d/<workspace>/terraform.tfstate`
- Remote backends: handled by backend (separate state per workspace)

### Workspace limitations
- All workspaces share the same config
- Not designed for strong isolation (e.g., separate AWS accounts)
- Use separate root modules + backends for true environment isolation

---

## 18. Provisioners (Deep Dive)

> Provisioners are a last resort. Prefer cloud-init, user-data, or configuration management tools.

### local-exec
```hcl
resource "aws_instance" "web" {
  # ...

  provisioner "local-exec" {
    command    = "ansible-playbook -i ${self.public_ip}, playbook.yml"
    working_dir = "/path/to/ansible"
    environment = {
      ANSIBLE_HOST_KEY_CHECKING = "False"
    }
    interpreter = ["/bin/bash", "-c"]
    when        = create   # or destroy
    on_failure  = fail     # or continue
  }
}
```

### remote-exec
```hcl
resource "aws_instance" "web" {
  # ...

  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
    timeout     = "5m"
  }

  provisioner "remote-exec" {
    inline = [
      "sudo yum update -y",
      "sudo yum install -y nginx",
    ]
    # OR
    script  = "scripts/setup.sh"
    # OR
    scripts = ["scripts/setup.sh", "scripts/configure.sh"]
  }
}
```

### file
```hcl
provisioner "file" {
  source      = "configs/app.conf"
  destination = "/etc/app/app.conf"
  # OR
  content     = "inline content here"
  destination = "/etc/app/app.conf"
}
```

### null_resource
```hcl
resource "null_resource" "setup" {
  triggers = {
    instance_id = aws_instance.web.id   # re-run when instance changes
  }

  provisioner "local-exec" {
    command = "echo ${aws_instance.web.public_ip} >> inventory.txt"
  }
}
```

### Destroy-time provisioners
```hcl
provisioner "local-exec" {
  when    = destroy
  command = "echo 'Destroying ${self.id}'"
}
```

- Destroy-time provisioners cannot reference `var`, `data`, or other resources
- `self` is the only reference allowed

---

## 19. Terraform Registry & Providers

### Registry URL
- Providers: `registry.terraform.io`
- Modules: `registry.terraform.io`
- Public URL: `https://registry.terraform.io`

### Provider tiers
| Tier | Maintained by | Indicator |
|---|---|---|
| Official | HashiCorp | `hashicorp/` namespace |
| Partner | Technology partners | Verified badge |
| Community | Community members | No badge |

### Finding providers
- `terraform providers` — show providers used in config
- `terraform providers lock` — add/update provider hashes in lock file
- `terraform providers mirror <dir>` — copy providers to local directory

### Lock file (`.terraform.lock.hcl`)
- Locks provider versions and hashes
- **Should be committed** to version control
- Updated by `terraform init` and `terraform providers lock`
- Ensures consistent provider versions across team

---

## 20. Terraform Cloud & Enterprise

### Key features comparison
| Feature | OSS | Cloud Free | Cloud Plus | Enterprise |
|---|---|---|---|---|
| Remote state | ✗ | ✓ | ✓ | ✓ |
| Remote runs | ✗ | ✓ | ✓ | ✓ |
| VCS integration | ✗ | ✓ | ✓ | ✓ |
| Private registry | ✗ | ✓ | ✓ | ✓ |
| Sentinel policies | ✗ | ✗ | ✓ | ✓ |
| SSO | ✗ | ✗ | ✓ | ✓ |
| Audit logging | ✗ | ✗ | ✗ | ✓ |
| Self-hosted agents | ✗ | ✗ | ✓ | ✓ |
| Team permissions | ✗ | Limited | ✓ | ✓ |

### Run modes
| Mode | Runs on | State stored |
|---|---|---|
| Remote (default) | TFC | TFC |
| Local | Local machine | TFC |
| Agent | Self-hosted runner | TFC |

### Sentinel (Policy as Code)
```python
# Example Sentinel policy
import "tfplan/v2" as tfplan

main = rule {
  all tfplan.resource_changes as _, changes {
    changes.type is not "aws_instance" or
    changes.change.after.instance_type in ["t3.micro", "t3.small"]
  }
}
```

- Enforcement levels: `advisory` (warn), `soft-mandatory` (override allowed), `hard-mandatory` (no override)

### Workspace variables
- Terraform variables (override tfvars)
- Environment variables (set in run environment)
- Can be marked sensitive (write-only, never shown)
- Variable sets: reusable groups of variables across workspaces

### Run triggers
- VCS-driven: auto-trigger on commits
- API-driven: trigger via API or CLI
- UI: manual trigger

---

## 21. Key Files Reference

| File | Committed? | Purpose |
|---|---|---|
| `main.tf` | Yes | Main resources |
| `variables.tf` | Yes | Variable declarations |
| `outputs.tf` | Yes | Output declarations |
| `providers.tf` | Yes | Provider configuration |
| `versions.tf` | Yes | Required versions |
| `terraform.tfvars` | Caution | Variable values (may have secrets) |
| `*.auto.tfvars` | Caution | Auto-loaded variable values |
| `.terraform.lock.hcl` | **Yes** | Provider version lock |
| `terraform.tfstate` | **No** | State (may have secrets) |
| `terraform.tfstate.backup` | No | State backup |
| `.terraform/` | No | Plugins/modules cache |
| `*.tfplan` | No | Saved plan files |

---

## 22. Debugging & Troubleshooting

### Logging
```bash
export TF_LOG=TRACE    # TRACE, DEBUG, INFO, WARN, ERROR
export TF_LOG_PATH=/tmp/terraform.log

# Provider-specific logging
export TF_LOG_PROVIDER=DEBUG

# Disable logging
unset TF_LOG
```

### Common issues
| Issue | Solution |
|---|---|
| State lock stuck | `terraform force-unlock <LOCK_ID>` |
| Provider version conflict | Update `.terraform.lock.hcl` with `terraform init -upgrade` |
| Resource drift | `terraform apply -refresh-only` |
| Config/state mismatch | `terraform import` to bring resource under management |
| Circular dependency | Use `depends_on` carefully or restructure config |

### terraform console
```bash
terraform console    # interactive expression evaluator
> var.region
> length(var.list)
> cidrsubnet("10.0.0.0/16", 8, 1)
```

---

## 23. Resource Graph & Parallelism

- Terraform builds a **directed acyclic graph (DAG)** of resources
- Independent resources are created/destroyed **in parallel** (default: 10 concurrent)
- Dependencies are respected (sequential where needed)
- `terraform graph | dot -Tsvg > graph.svg` — visualize the graph
- `-parallelism=N` to control concurrency

---

## 24. Import & Drift

### terraform import
```bash
# Import existing resource into state
terraform import aws_instance.web i-1234567890abcdef0
terraform import aws_s3_bucket.bucket my-bucket-name
terraform import 'aws_instance.web[0]' i-1234567890   # count
terraform import 'aws_instance.web["key"]' i-xyz       # for_each
terraform import module.vpc.aws_vpc.main vpc-abc123    # module
```

- Writes to state only; you must write the matching config manually
- Run `terraform plan` after import to verify no diff

### import block (Terraform 1.5+)
```hcl
import {
  to = aws_instance.web
  id = "i-1234567890abcdef0"
}
```

### generate config (Terraform 1.5+)
```bash
terraform plan -generate-config-out=generated.tf
```

### Drift detection
```bash
terraform plan -refresh-only    # show drift without making changes
terraform apply -refresh-only   # update state to match reality
```

---

## 25. Moved Block (Terraform 1.1+)

```hcl
# Rename a resource without destroying/recreating it
moved {
  from = aws_instance.web
  to   = aws_instance.app
}

# Move into/out of a module
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}

# Move with count/for_each
moved {
  from = aws_instance.web
  to   = aws_instance.web["us-east-1"]
}
```

---

## 26. Terraform Versions & Compatibility

### Version constraints for Terraform itself
```hcl
terraform {
  required_version = ">= 1.5.0, < 2.0.0"
}
```

### Checking version
```bash
terraform version
terraform -v
```

### Version manager
- `tfenv` — manage multiple Terraform versions
- `tfswitch` — switch between versions

---

## 27. Security Best Practices

- **Never commit** `terraform.tfstate` or `*.tfvars` containing secrets
- Use `sensitive = true` for sensitive variables and outputs
- Encrypt state at rest (S3 SSE, Terraform Cloud)
- Use IAM roles/workload identity instead of static credentials
- Use `prevent_destroy = true` for critical resources
- Regularly rotate credentials used by Terraform
- Use Sentinel or OPA to enforce policies
- Use private registry for internal modules
- Pin provider versions with lock file

---

## 28. Common Exam Topics & Gotchas

| Topic | Key fact |
|---|---|
| `terraform init` | Must be run first; downloads providers and modules |
| `.terraform.lock.hcl` | Commit to VCS; ensures consistent provider versions |
| `terraform.tfstate` | Do NOT commit; may contain secrets |
| `terraform plan` | Never modifies infrastructure; read-only against provider APIs |
| `count` vs `for_each` | for_each preferred; count causes index shift issues on deletion |
| `depends_on` | For implicit/hidden dependencies only |
| `lifecycle.prevent_destroy` | Only prevents `terraform destroy`; won't stop you deleting the resource block |
| `lifecycle.ignore_changes` | Ignores specified attributes after initial creation |
| Variable precedence | `-var` flag beats everything else (except env vars? — No, `-var` beats env vars too) |
| `sensitive` | Hides from output; still in state plaintext |
| Workspaces | Not for strong isolation between prod/dev; use separate state backends for that |
| Provisioners | Last resort; failures can leave partial state |
| `null_resource` | No real infra; good for provisioners without a resource |
| `data` sources | Read-only; refreshed every plan/apply |
| `terraform import` | State only; must write config manually |
| Modules from Registry | Require `version` argument |
| `terraform validate` | No API calls; only syntax/logic check |
| `terraform fmt` | Canonical formatting; return code: 0=success |
| Remote backend | State stored remotely; locking mechanism required |
| `TF_VAR_name` | Environment variable for variable `name` |
| `terraform output -json` | Machine-readable output |
| `terraform console` | Test expressions interactively |
| Resource graph | DAG; independent resources created in parallel |
| `~> 1.0` | Allows 1.x but not 2.0 |
| `~> 1.0.0` | Allows 1.0.x but not 1.1.0 |
| Sentinel | TFC/TFE only; policy-as-code |
| `create_before_destroy` | New resource first, then old one destroyed |
| `terraform taint` (deprecated) | Use `terraform apply -replace=` instead |
| `terraform refresh` (deprecated) | Use `terraform apply -refresh-only` instead |

---

## 29. Version Constraints — Detailed Examples (CRITICAL EXAM TOPIC)

### The `~>` Operator Rule
The `~>` operator (pessimistic constraint) locks the **rightmost specified digit**. Everything to the LEFT is FIXED.

```
~> X.Y.Z   →  only Z can change   →  >= X.Y.Z, < X.(Y+1).0
~> X.Y     →  only Y can change   →  >= X.Y.0, < (X+1).0.0
```

### Worked Examples Table

| Constraint | Min Version | Max Version (exclusive) | Examples |
|---|---|---|---|
| `~> 3.2.0` | 3.2.0 | 3.3.0 | Allows 3.2.0, 3.2.5; **denies** 3.3.0, 4.0.0 |
| `~> 3.2` | 3.2.0 | 4.0.0 | Allows 3.2.0, 3.2.5, 3.9.9; **denies** 4.0.0 |
| `~> 6.0.0` | 6.0.0 | 6.1.0 | Allows 6.0.2, 6.0.5; **denies** 6.1.0 |
| `~> 6.0` | 6.0.0 | 7.0.0 | Allows 6.0.5, 6.1.0, 6.9.9; **denies** 7.0.0 |
| `~> 5.0` | 5.0.0 | 6.0.0 | Allows 5.0.0, 5.3.0, 5.9.9; **denies** 6.0.0 |

### `terraform init` vs `terraform init -upgrade`

| Command | What It Does |
|---|---|
| `terraform init` | Uses already-installed version (won't upgrade even if newer matches) |
| `terraform init -upgrade` | Upgrades to newest version matching constraints AND updates `.terraform.lock.hcl` |

### What `-upgrade` Does NOT Do
- Does NOT upgrade Terraform CLI itself
- Does NOT upgrade the backend
- Does NOT upgrade configuration syntax

### Exam Trap
With `~> 3.2.0` and versions 3.2.0, 3.2.5, 3.3.0, 4.0.0 available:
- `terraform init -upgrade` installs **3.2.5** (NOT 3.3.0, NOT 4.0.0)

---

## 30. for_each Type Requirements

### The Strict Rule
`for_each` accepts ONLY:
- `map(any)`
- `set(string)`

It does NOT accept:
- `list(string)` — must convert with `toset()` first
- `list(number)` — must convert
- `number` — that's `count`

### Code Examples

```hcl
variable "names" {
  type    = list(string)
  default = ["web", "api", "db"]
}

# WRONG — list not accepted
resource "aws_instance" "app" {
  for_each = var.names                    # ERROR
}

# RIGHT — convert to set
resource "aws_instance" "app" {
  for_each = toset(var.names)             # WORKS
  name     = each.value                   # for sets, each.key == each.value
}

# ALSO RIGHT — use a map
resource "aws_instance" "app" {
  for_each = { for n in var.names : n => n }   # WORKS
  name     = each.key
}
```

### Referencing for_each Resources
```hcl
aws_instance.app["web"]     # by string key (not numeric index)
aws_instance.app["api"]     # by string key
```

---

## 31. Validation Mechanisms — When to Use Each

### Four Validation Mechanisms

| Mechanism | Where | When It Runs | Use For |
|---|---|---|---|
| `validation {}` | Inside `variable` block | During input evaluation (before plan) | Validating variable values |
| `precondition {}` | Inside `lifecycle` block | Before resource/data create or update | Checking assumptions BEFORE action |
| `postcondition {}` | Inside `lifecycle` block | After resource/data create or update | Verifying results AFTER action |
| `check {}` | Top-level block | After apply (as a health check) | Ongoing assertions |

### Examples

**Variable validation** — restrict what users can pass:
```hcl
variable "instance_count" {
  type = number
  validation {
    condition     = var.instance_count > 0 && var.instance_count <= 10
    error_message = "Must be 1-10."
  }
}
```

**Precondition** — check BEFORE acting:
```hcl
resource "aws_instance" "web" {
  lifecycle {
    precondition {
      condition     = var.env != "prod" || var.instance_type != "t2.micro"
      error_message = "Cannot use t2.micro in production."
    }
  }
}
```

**Postcondition** — verify AFTER acting (perfect for data sources):
```hcl
data "azurerm_resource_group" "main" {
  name = var.rg_name
  lifecycle {
    postcondition {
      condition     = self.id != ""
      error_message = "Resource group not found."
    }
  }
}
```

### Exam Decision Tree
- Data source needs to verify it found something → **postcondition**
- Resource needs to verify inputs before creation → **precondition**
- Variable must be within a range → **validation block** in variable
- Ongoing assertion across multiple resources → **check block**

---

## 32. HCP Terraform — `cloud` Block vs `backend` Block

### The Critical Distinction

```hcl
# CORRECT — Use cloud block for HCP Terraform
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "production"
    }
  }
}

# WRONG — there is NO backend "hcp"
terraform {
  backend "hcp" { }     # DOES NOT EXIST
}

# This is for S3/GCS/etc backends, NOT HCP Terraform
terraform {
  backend "s3" { }      # For S3 backend only
}
```

### Comparison

| | `cloud {}` | `backend "s3" {}` |
|---|---|---|
| Used for | HCP Terraform / TFE | S3, GCS, azurerm, etc. |
| Execution | Remote by default | Always local |
| State location | HCP Terraform | The configured backend |
| Setup | `terraform login` → `terraform init` | `terraform init` |

### Backend Block Limitation (Major Exam Trap)
Backend blocks accept **literal values ONLY**. You CANNOT use:
- `var.bucket_name` — NO
- `local.region` — NO
- `data.aws_caller_identity.current.account_id` — NO

Only literal strings work: `bucket = "my-state-bucket"`.

For dynamic values, use **partial backend configuration**:
```bash
terraform init -backend-config="bucket=my-bucket" -backend-config="key=prod.tfstate"
```

---

## 33. Cross-Workspace Data Sharing in HCP Terraform

### Two Methods

| Method | When to Use | Syntax |
|---|---|---|
| `tfe_outputs` | HCP Terraform → HCP Terraform | `data "tfe_outputs" "x" {...}` |
| `terraform_remote_state` | Any backend → Any backend | `data "terraform_remote_state" "x" {...}` |

### tfe_outputs (HCP Terraform Specific)
```hcl
data "tfe_outputs" "webserver" {
  organization = "my-org"
  workspace    = "prod-webserver"
}

# Access outputs (note .values.):
resource "aws_route53_record" "web" {
  records = [data.tfe_outputs.webserver.values.public_ip]
}
```

### terraform_remote_state (Generic)
```hcl
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "my-state"
    key    = "vpc/terraform.tfstate"
    region = "us-east-1"
  }
}

# Access outputs (note .outputs.):
data.terraform_remote_state.vpc.outputs.vpc_id
```

### Key Difference in Syntax
- `tfe_outputs` → `data.tfe_outputs.<name>.values.<output>`
- `terraform_remote_state` → `data.terraform_remote_state.<name>.outputs.<output>`

---

## 34. HCP Terraform — Run Tasks vs Run Triggers vs Variable Sets

### Comparison Table

| Feature | What It Does | Example Use |
|---|---|---|
| **Run Triggers** | Auto-queue run in workspace B after workspace A applies | network workspace → app workspace |
| **Run Tasks** | Integrate external tools BETWEEN plan and apply | Snyk security scan, cost estimator |
| **Variable Sets** | Share variables across workspaces | Common AWS creds for a project |
| **Agents** | Execute runs in your private network | On-prem or air-gapped infra |

### Run Tasks (Detailed)
- Execute between plan and apply phases
- Call external HTTP endpoints (webhooks)
- Can be **advisory** (warning only) or **mandatory** (blocks apply)
- Use cases: Snyk, Infracost, custom compliance checks

### Run Triggers (Detailed)
- Source workspace must successfully **apply** for trigger to fire
- Multiple sources possible (AND or OR depending on config)
- Cannot chain across organizations
- Used for orchestrating dependent infrastructure

### HCP Terraform Agents
- Lightweight binary you run on YOUR infrastructure
- Receives plans from HCP Terraform
- For accessing private networks, on-prem resources, isolated VPCs
- NOT used for state management or workspace control
- Plus / Enterprise feature

### Variable Set Scopes
1. **Global** — all current and future workspaces in the org
2. **Project-specific** — all workspaces in selected projects
3. **Workspace-specific** — only selected workspaces

---

## 35. Sensitive Data — The Hard Truth

### The Absolute Rule
**No method prevents sensitive values from being stored in the Terraform state file.** Period.

| Method | Hidden in CLI? | In State File? |
|---|---|---|
| `sensitive = true` on variable | YES (redacted) | **YES — plaintext** |
| `sensitive = true` on output | YES (redacted) | **YES — plaintext** |
| Vault data source | YES (if marked sensitive) | **YES — plaintext** |
| Environment variable | Not in config files | **YES — still in state** |
| `*.tfvars` file | Depends | **YES — still in state** |

### THE Exception: Ephemeral Resources & Write-Only Attributes (Terraform 1.10+)

Ephemeral resources exist only during a Terraform operation, NOT in state:
```hcl
ephemeral "random_password" "db" {
  length = 20
}

resource "aws_db_instance" "main" {
  password_wo         = ephemeral.random_password.db.result
  password_wo_version = 1    # increment to trigger rotation
}
```

This is the **ONLY** way to keep a value completely out of state.

### How to Protect State
Since sensitive data IS in state, protect the state itself:
- Remote backend with encryption at rest (S3 + SSE)
- Access controls (IAM, RBAC)
- State locking (DynamoDB for S3)
- NEVER commit state to git
- HCP Terraform encrypts state at rest and in transit by default

---

## 36. moved Block — Detailed Usage

### When to Use
Use `moved` when you renamed or restructured a resource and don't want destroy/recreate.

### Common Patterns

```hcl
# Renamed a resource
moved {
  from = aws_s3_bucket.logs
  to   = aws_s3_bucket.app_logs
}

# Moved into a module
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}

# Moved with count or for_each
moved {
  from = aws_instance.web
  to   = aws_instance.web["us-east-1"]
}
```

### Decision Matrix

| Scenario | Correct Approach |
|---|---|
| Renamed a resource | `moved` block |
| Moved resource into a module | `moved` block |
| Want to import existing resource | `import` block |
| Remove from state without destroying | `removed` block (1.7+) or `terraform state rm` |
| Destroy a specific resource | Delete the resource block + `terraform apply` |
| Force a resource to be replaced | `terraform apply -replace=<addr>` |

### How Long to Keep moved Blocks
- Keep for at least **one full apply cycle** across ALL team members
- Removing too soon = Terraform thinks old was destroyed, new needs creation
- Safe to remove after everyone has applied with the block present

---

## 37. Plan Output Symbols

| Symbol | Meaning |
|---|---|
| `+` | Create — new resource |
| `-` | Destroy — resource will be removed |
| `~` | **Update in-place** — modified without recreation |
| `-/+` | Replace (destroy then create) — default replacement |
| `+/-` | Replace (create then destroy) — with `create_before_destroy` |
| `<=` | Read — data source will be read |

### Memory Aid
- `+` = plus = adding new
- `-` = minus = removing
- `~` = tilde = "similar" = updating in-place
- `-/+` = destroy first, create second (default)

### Common Exam Trap
Q: A resource has `~` in plan — what happens?
A: **Updated in place** (NOT created, NOT destroyed-and-recreated)

---

## 38. Commands That Modify State vs Read-Only

### Commands That MODIFY State
| Command | What It Writes |
|---|---|
| `terraform apply` | Creates/updates/deletes resources, writes new state |
| `terraform destroy` | Removes resources, writes new state |
| `terraform import` | Adds resource to state |
| `terraform state mv` | Moves resource address in state |
| `terraform state rm` | Removes resource from state |
| `terraform apply -refresh-only` | Updates state to match real infra |

### Commands That DO NOT Modify State
| Command | What It Does |
|---|---|
| `terraform plan` | Read-only comparison |
| `terraform validate` | Syntax check only |
| `terraform fmt` | Formats `.tf` files |
| `terraform show` | Displays state or plan |
| `terraform graph` | Generates dependency graph |
| `terraform output` | Reads outputs from state |
| `terraform state list` | Lists resources in state |
| `terraform state show` | Shows resource details |
| `terraform console` | Interactive expression evaluator |

### Common Exam Trap
Q: Which commands modify state? (select multiple)
- `apply` ✓
- `destroy` ✓
- `import` ✓
- `plan` ✗ (read-only)
- `fmt` ✗ (formats files only)
- `graph` ✗ (read-only)

---

## 39. terraform show vs state list vs state show

### Comparison

| Command | What It Shows | Use Case |
|---|---|---|
| `terraform show` | ENTIRE state file (verbose) | Full overview |
| `terraform state list` | Resource addresses ONLY (concise) | Quick count / inventory |
| `terraform state show <addr>` | Single resource (detailed) | Investigate one resource |

### Sample Output

```bash
# terraform state list — concise
aws_instance.web
aws_s3_bucket.logs
module.vpc.aws_vpc.main

# terraform state show aws_instance.web — single resource details
resource "aws_instance" "web" {
    ami           = "ami-12345"
    instance_type = "t3.micro"
    public_ip     = "54.1.2.3"
    ...50 more attributes...
}

# terraform show — EVERYTHING
# All resources with all attributes (potentially hundreds of lines)
```

### When to Use Which
- "How many resources do I have?" → **state list**
- "What attributes does resource X have?" → **state show <addr>**
- "Show me the full state file" → **show**

---

## 40. Module Variable Scope — The Strict Rules

### The Absolute Rule
**Child modules NEVER inherit anything from parent. NEVER.**

```
ROOT MODULE                     CHILD MODULE
variable "env" = "prod"        variable "env" — MUST BE DECLARED HERE
                                                OR YOU GET AN ERROR
module "web" {
  source = "./web"
  env    = var.env  ◄── MUST explicitly pass
}
```

### What DOES Flow Between Modules

| Direction | What Flows | How |
|---|---|---|
| Root → Child | Variables | Pass in module block |
| Child → Root | Outputs | `module.<name>.<output>` |
| Root → Child | Provider configs | Automatic (default) or `providers = {}` |
| Anywhere | `terraform.workspace` | Available everywhere |
| Root ↛ Child | Locals | NEVER inherited |
| Root ↛ Child | Data sources | NEVER inherited |
| Child ↛ Child | Nothing direct | Must go through root |

### Example: Cross-Module Wiring

```hcl
# Root module
module "network" {
  source = "./modules/network"
  cidr   = "10.0.0.0/16"
}

module "app" {
  source    = "./modules/app"
  vpc_id    = module.network.vpc_id   # network output → app input
  subnet_id = module.network.subnet_id
}

# modules/network/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

# modules/app/variables.tf
variable "vpc_id" {
  type = string
}
```

---

## 41. Backend Locking Support

### Which Backends Support Locking?

| Backend | Supports Locking? | How |
|---|---|---|
| **S3** + DynamoDB | YES | Need `dynamodb_table` config |
| **S3** alone (no DynamoDB) | **NO** | State corruption risk |
| **azurerm** | YES | Built-in (blob lease) |
| **gcs** | YES | Built-in |
| **consul** | YES | Built-in |
| **HCP Terraform** | YES | Built-in |
| **pg** (PostgreSQL) | YES | Built-in |
| **local** | **NO** | No concurrent protection |
| **http** | **DEPENDS** | Only if server implements it |

### Common Exam Trap
"All remote backends support locking" = **FALSE**
- S3 alone does NOT lock; needs DynamoDB
- http depends on the implementation

### What Happens with Concurrent Apply
- **With locking**: Second user gets "Error acquiring state lock"
- **Without locking**: Both succeed concurrently → state corruption possible

---

## 42. IaC vs Configuration Management

### The Distinction

| | IaC (Terraform) | Config Mgmt (Chef/Puppet/Ansible) |
|---|---|---|
| **Focus** | Provisioning infrastructure | Configuring software |
| **Manages** | VMs, networks, storage, IAM | Packages, services, files on existing servers |
| **Approach** | Declarative (desired state) | Often imperative or mixed |
| **Lifecycle** | Create / update / destroy infra | Install / configure / patch software |
| **Example** | Create an EC2 instance | Install Nginx on that instance |
| **Agents** | Agentless (uses APIs) | Often agent-based |

### They Are Complementary
Terraform provisions the VM → Chef/Ansible configures the software on it.
Many orgs use both: Terraform for the infrastructure, then Ansible for the software stack.

---

## 43. Ephemeral Resources & Write-Only Attributes (Terraform 1.10+)

### The Problem They Solve
Regular resources/variables ALWAYS write to state. Even `sensitive = true` doesn't help.

### Ephemeral Resources
Resources that exist only during a Terraform operation. NOT stored in state.

```hcl
ephemeral "random_password" "db" {
  length = 20
}
```

### Write-Only Attributes
Resource attributes that are written to the provider but NOT stored in state.

```hcl
resource "aws_db_instance" "main" {
  password_wo         = ephemeral.random_password.db.result
  password_wo_version = 1     # increment to trigger password change
}
```

### Key Points
- This is the **ONLY** way to keep a value completely out of state
- Use for short-lived secrets, OTP-style credentials, etc.
- New in Terraform 1.10
- Mark variables as `ephemeral = true` for true ephemerality

---

## 44. terraform plan — What It Actually Does

### The Complete Sequence
1. Loads the configuration
2. **Refreshes state** by querying provider APIs (unless `-refresh=false`)
3. **Compares** refreshed state to desired config
4. **Creates an execution plan** showing required changes
5. Does NOT modify state or infrastructure

### The Exam-Correct Answer
"Creates an execution plan and determines what changes are required to achieve the desired state."

NOT just "reconciles state" — that's only step 1, an incomplete answer.

### Plan Flags
```bash
terraform plan                          # standard plan
terraform plan -out=tfplan              # save plan for later apply
terraform plan -refresh=false           # skip state refresh
terraform plan -refresh-only            # only show drift, no proposed changes
terraform plan -destroy                 # show destroy plan
terraform plan -target=aws_instance.web # plan only one resource
terraform plan -var="env=prod"          # set variable
```

---

## 45. Reference Syntax Cheat Sheet

### Reference Patterns Table

| Block Type | Declaration | Reference |
|---|---|---|
| Resource | `resource "aws_vpc" "main"` | `aws_vpc.main.id` |
| Data source | `data "aws_vpc" "main"` | `data.aws_vpc.main.id` |
| Module output | `module "vpc"` | `module.vpc.vpc_id` |
| Variable | `variable "name"` | `var.name` |
| Local | `locals { x = 1 }` | `local.x` |
| Workspace name | — | `terraform.workspace` |
| With count | `count = 3` | `aws_instance.web[0]` |
| With for_each | `for_each = {...}` | `aws_instance.web["key"]` |
| Splat (count) | — | `aws_instance.web[*].id` |
| Self reference | inside resource | `self.attribute` |

### Map Key with Hyphen
```hcl
var.metadata["build-tag"]     # CORRECT — bracket notation
var.metadata.build-tag        # WRONG — HCL parses hyphen as minus
```

### Module Output Syntax
```hcl
module.vpc.vpc_id              # CORRECT
output.vpc.vpc_id              # WRONG — no "output." prefix exists
var.vpc.vpc_id                 # WRONG — var. is for input variables
module.vpc.aws_vpc.main.id     # WRONG — can't reach into child resources
```

---

## 46. HCP Terraform Workspaces — Quick Facts

### Workspace Mapping Rules
- 1 workspace = 1 VCS repo (max)
- Multiple workspaces CAN point to the same repo
- 1 workspace = 1 state file
- 1 workspace = its own variables, team access, run history

### Workflow Types

| Workflow | How Runs Are Triggered | Where Runs Execute |
|---|---|---|
| **VCS-driven** | Git push / pull request | HCP Terraform (Remote) by default |
| **CLI-driven** | `terraform plan/apply` from local | HCP Terraform (Remote) by default |
| **API-driven** | REST API calls | HCP Terraform (Remote) by default |

### Execution Modes

| Mode | Plans/Applies Run On | State Stored In |
|---|---|---|
| **Remote** (default) | HCP Terraform infrastructure | HCP Terraform |
| **Local** | Your machine | HCP Terraform |
| **Agent** | Self-hosted agent | HCP Terraform |

### Major Exam Trap
CLI-driven workflow with default settings runs **REMOTELY on HCP Terraform**, NOT on your local machine. Output is streamed back to your terminal.

---

## 47. Quick Reference — Things Often Confused

### Provider tier definitions
| Tier | Maintained By | Indicator |
|---|---|---|
| Official | HashiCorp | `hashicorp/` namespace |
| Partner | Tech vendor | Verified badge |
| Community | Community | No badge |

### Provider plugin = the Terraform plugin concept
- Terraform plugins ARE providers
- Modules are NOT plugins
- Backends are NOT plugins
- Variables are NOT plugins

### Default state file location
With no backend configured, Terraform uses local backend → `terraform.tfstate` in the **current working directory**.

### Default state file format
JSON, plaintext (NOT encrypted, NOT compressed, NOT YAML, NOT HCL).

### State file lineage
A unique ID for state file history. Helps detect when two state files have diverged.

### State file backup
`terraform.tfstate.backup` is created automatically before each state change.

### What `terraform init` does (3 things)
1. Downloads required providers/plugins
2. Downloads required modules
3. Initializes the backend

NOT: provisions resources, validates config, formats code.

### When to re-run `terraform init`
- Added/removed/changed a provider
- Added/removed/changed a module
- Updated provider or module versions
- Changed the backend

---

## 48. Final Exam Survival Tips

### Top 18 Things Not to Forget

1. `~> X.Y.Z` locks ONLY Z; `~> X.Y` locks ONLY Y
2. `for_each` needs map or set; convert lists with `toset()`
3. Use `cloud {}` block for HCP Terraform, NOT `backend "hcp"`
4. `tfe_outputs` for cross-workspace HCP data
5. Run **tasks** = external tools; Run **triggers** = workspace chaining
6. `sensitive = true` does NOT keep values out of state
7. Ephemeral resources are the ONLY way to keep secrets out of state
8. `moved` block for renames; keep for one full apply cycle
9. `~` = update in-place; `-/+` = replace
10. Plan, validate, fmt, show DON'T modify state
11. `state list` (concise) vs `state show` (one resource) vs `show` (all state)
12. Child modules NEVER inherit from parent — pass explicitly
13. S3 backend needs DynamoDB for locking; not all backends lock
14. IaC = infrastructure provisioning; Config Mgmt = software config
15. Postcondition for data source verification (after fetch)
16. Map keys with hyphens need brackets: `var.x["build-tag"]`
17. `module.<name>.<output>` is the ONLY way to access child outputs
18. CLI-driven HCP Terraform runs REMOTELY on HCP, not locally
