# 🎯 Terraform Associate Exam – Last-Minute Cheatsheet

> **Exam Version:** Terraform Associate (003/004)  
> **Format:** Multiple choice & multi-select | 57 questions | 60 minutes  
> **Passing Score:** ~70%  
> **Good luck tomorrow, Venkata Rao! 🍀**

---

## 1️⃣ Infrastructure as Code (IaC)

### What is IaC?
- Managing infrastructure through **code/configuration files** instead of manual processes
- Terraform uses **declarative** approach (you define desired state, Terraform figures out how)
- Imperative = step-by-step instructions (scripts); Declarative = define end state

### Advantages of IaC
| Benefit | Description |
|---------|-------------|
| **Versioning** | Track changes in VCS (Git) |
| **Consistency** | Same config = same result every time |
| **Reusability** | Modules, shared configs |
| **Automation** | CI/CD pipelines |
| **Reduced errors** | No manual clicks/typos |
| **Collaboration** | Team can review, approve changes |
| **Self-documenting** | Code IS the documentation |

### Why Terraform?
- **Cloud-agnostic** — works with AWS, Azure, GCP, and 3000+ providers
- **Single workflow** — same CLI for all providers
- **State management** — tracks real-world resources
- **Execution plans** — preview before apply
- **Resource graph** — parallel resource creation
- **HCL** — human-readable configuration language

---

## 2️⃣ Terraform Fundamentals

### Core Components
```
┌─────────────────────────────────────────────┐
│  Configuration (.tf files)                   │
│  ├── Providers (AWS, Azure, GCP...)         │
│  ├── Resources (what to create)             │
│  ├── Data Sources (read existing infra)     │
│  ├── Variables (inputs)                     │
│  ├── Outputs (expose values)                │
│  └── Modules (reusable packages)            │
└─────────────────────────────────────────────┘
```

### Providers
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"    # <NAMESPACE>/<TYPE>
      version = "~> 5.0"           # Version constraint
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

**Key Points:**
- Providers are **plugins** that interact with APIs
- Downloaded during `terraform init`
- Source format: `<HOSTNAME>/<NAMESPACE>/<TYPE>` (default hostname: `registry.terraform.io`)
- Version constraints: `=`, `!=`, `>`, `>=`, `<`, `<=`, `~>` (pessimistic)
- `~> 5.0` means `>= 5.0, < 6.0`
- `~> 5.1` means `>= 5.1, < 5.2` (only rightmost increments)

### Dependency Lock File (`.terraform.lock.hcl`)
- Created/updated by `terraform init`
- Records provider versions + hashes
- **Should be committed to VCS**
- Use `terraform init -upgrade` to update providers

### Multiple Providers (Alias)
```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

resource "aws_instance" "west_server" {
  provider = aws.west
  # ...
}
```

---

## 3️⃣ Core Terraform Workflow

### The Three Steps
```
Write  →  Plan  →  Apply
```

### Essential Commands
| Command | Purpose |
|---------|---------|
| `terraform init` | Initialize working directory, download providers/modules |
| `terraform validate` | Check syntax & internal consistency (no API calls) |
| `terraform plan` | Preview changes (+ create, ~ update, - destroy) |
| `terraform apply` | Execute changes (prompts for approval) |
| `terraform apply -auto-approve` | Skip confirmation prompt |
| `terraform destroy` | Remove all managed resources |
| `terraform fmt` | Format code to canonical style |
| `terraform fmt -check` | Check formatting without changing files |
| `terraform show` | Display current state or plan |
| `terraform output` | Show output values |
| `terraform refresh` | Update state to match real-world (deprecated in favor of `-refresh-only`) |
| `terraform plan -refresh-only` | Detect drift without changing infra |

### Init Details
- Downloads providers → `.terraform/providers/`
- Downloads modules → `.terraform/modules/`
- Configures backend
- Creates/updates `.terraform.lock.hcl`
- **Must run** when: new provider added, module source changed, backend changed

---

## 4️⃣ Terraform Configuration

### Resources
```hcl
resource "aws_instance" "web" {    # <TYPE>.<NAME>
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "HelloWorld"
  }
}
```

### Data Sources (read-only)
```hcl
data "aws_ami" "latest" {          # data.<TYPE>.<NAME>
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*"]
  }
}

# Reference: data.aws_ami.latest.id
```

### Variables
```hcl
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
  sensitive   = false
  
  validation {
    condition     = contains(["t2.micro", "t2.small"], var.instance_type)
    error_message = "Must be t2.micro or t2.small."
  }
}
```

**Variable Precedence (lowest → highest):**
1. Default value in variable block
2. `terraform.tfvars` file
3. `*.auto.tfvars` files (alphabetical order)
4. `-var-file` flag
5. `-var` flag
6. `TF_VAR_<name>` environment variable
7. **Interactive prompt** (if no default & not provided)

> ⚠️ **Actually**: Environment variables have LOWER precedence than tfvars files. The correct order is:
> 1. Default values
> 2. Environment variables (`TF_VAR_name`)
> 3. `terraform.tfvars`
> 4. `*.auto.tfvars` (alphabetical)
> 5. `-var-file`
> 6. `-var` CLI flag

### Variable Types
```hcl
# Primitive
string, number, bool

# Complex/Collection
list(type)      # Ordered, e.g. list(string)
set(type)       # Unordered, unique
map(type)       # Key-value, e.g. map(string)
tuple([types])  # Fixed-length, mixed types
object({attrs}) # Named attributes with types

# Special
any             # Accept any type
```

### Outputs
```hcl
output "instance_ip" {
  description = "Public IP of the instance"
  value       = aws_instance.web.public_ip
  sensitive   = true   # Hides from CLI output
}
```

### Local Values
```hcl
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}

# Reference: local.common_tags
```

### Expressions & Functions
```hcl
# Conditional
condition ? true_val : false_val

# For expressions
[for s in var.list : upper(s)]              # list
{for k, v in var.map : k => upper(v)}       # map

# Splat
aws_instance.web[*].public_ip               # Same as for loop

# Common Functions
length(), toset(), tolist(), tomap()
lookup(map, key, default)
element(list, index)
file("path")                    # Read file content
templatefile("path", vars)      # Render template
merge(map1, map2)
concat(list1, list2)
join(separator, list)
split(separator, string)
format(spec, values...)
replace(string, search, replace)
regex(pattern, string)
cidrsubnet(prefix, newbits, netnum)
try(expr1, expr2, ...)          # Return first non-error
can(expr)                       # Returns true/false
```

### Dynamic Blocks
```hcl
resource "aws_security_group" "example" {
  dynamic "ingress" {
    for_each = var.ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

### Resource Dependencies
```hcl
# Implicit (automatic via references)
resource "aws_instance" "web" {
  subnet_id = aws_subnet.main.id   # Terraform knows this depends on subnet
}

# Explicit (when no direct reference)
resource "aws_instance" "web" {
  depends_on = [aws_iam_role_policy.example]
}
```

### Meta-Arguments (apply to any resource)
| Meta-Argument | Purpose |
|---------------|---------|
| `depends_on` | Explicit dependency |
| `count` | Create multiple instances by number |
| `for_each` | Create multiple instances from map/set |
| `provider` | Select non-default provider |
| `lifecycle` | Customize resource behavior |

### Lifecycle Rules
```hcl
lifecycle {
  create_before_destroy = true    # New resource first, then delete old
  prevent_destroy       = true    # Error if terraform tries to destroy
  ignore_changes        = [tags]  # Don't update on these changes
  replace_triggered_by  = [...]   # Force replace when referenced changes
}
```

### Sensitive Data & Secrets
- Mark variables as `sensitive = true`
- Mark outputs as `sensitive = true`
- Use **Vault provider** for secrets management
- State file contains secrets in **plaintext** → secure your state!
- Use environment variables for credentials
- Never commit secrets to VCS

---

## 5️⃣ Terraform Modules

### What is a Module?
- A **container for multiple resources** used together
- Every Terraform configuration is a module (root module)
- Child modules are called from root module

### Module Sources
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"   # Terraform Registry
  version = "5.0.0"
  
  # Variables passed to module
  cidr = "10.0.0.0/16"
}
```

**Source Types:**
- Local path: `source = "./modules/vpc"`
- Terraform Registry: `source = "hashicorp/consul/aws"`
- GitHub: `source = "github.com/org/repo"`
- S3 bucket: `source = "s3::https://bucket.s3.amazonaws.com/module.zip"`
- Generic Git: `source = "git::https://example.com/module.git"`

### Module Structure
```
my-module/
├── main.tf          # Resources
├── variables.tf     # Input variables
├── outputs.tf       # Output values
├── README.md        # Documentation
└── versions.tf      # Provider requirements
```

### Key Module Concepts
- Variables in modules have their own **scope** (encapsulated)
- Parent passes values via module block arguments
- Child exposes values via `output` blocks
- Reference module outputs: `module.<MODULE_NAME>.<OUTPUT_NAME>`
- **Version pinning** is critical for published modules
- Run `terraform init` when adding/changing module sources
- Run `terraform get` to download modules only

---

## 6️⃣ Terraform State

### Purpose of State
- Maps real-world resources to your configuration
- Tracks metadata (dependencies, resource attributes)
- Enables performance optimization (caching)
- Stored in `terraform.tfstate` (JSON format)

### State Backends

| Backend | Description |
|---------|-------------|
| **local** (default) | `terraform.tfstate` in working directory |
| **remote** | S3, GCS, Azure Blob, Consul, HCP Terraform |

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"   # For locking
    encrypt        = true
  }
}
```

### State Locking
- Prevents concurrent state modifications
- **DynamoDB** for S3 backend
- Automatic with HCP Terraform
- Force unlock: `terraform force-unlock <LOCK_ID>`

### State Commands
| Command | Purpose |
|---------|---------|
| `terraform state list` | List resources in state |
| `terraform state show <resource>` | Show resource details |
| `terraform state mv` | Rename/move resource in state |
| `terraform state rm` | Remove resource from state (doesn't destroy) |
| `terraform state pull` | Download remote state |
| `terraform state push` | Upload local state to remote |

### Managing Drift
- `terraform plan` detects drift automatically
- `terraform apply -refresh-only` — update state without changing infra
- `terraform plan -refresh-only` — preview drift detection

### Import
```hcl
# New way (Terraform 1.5+) - import block
import {
  to = aws_instance.web
  id = "i-1234567890abcdef0"
}

# Old way - CLI
terraform import aws_instance.web i-1234567890abcdef0
```
- Import brings existing resources under Terraform management
- You must write the configuration manually (import doesn't generate config)
- `terraform plan -generate-config-out=generated.tf` (1.5+) can help

### Moved Block (Refactoring)
```hcl
moved {
  from = aws_instance.old_name
  to   = aws_instance.new_name
}
```

### Removed Block
```hcl
removed {
  from = aws_instance.temporary

  lifecycle {
    destroy = false   # Keep the real resource, just remove from state
  }
}
```

---

## 7️⃣ HCP Terraform (Terraform Cloud)

### What is HCP Terraform?
- HashiCorp's managed service for Terraform
- Remote execution, state management, team collaboration

### Key Features
| Feature | Description |
|---------|-------------|
| **Remote State** | Encrypted, versioned state storage |
| **Remote Execution** | Runs happen on HCP servers |
| **VCS Integration** | Auto-trigger runs from Git commits |
| **Private Registry** | Host private modules/providers |
| **Policy as Code** | Sentinel / OPA policies |
| **Team Management** | RBAC, SSO |
| **Cost Estimation** | Preview infrastructure costs |
| **Run Triggers** | Chain workspace executions |

### Execution Modes
| Mode | Where Plans/Applies Run |
|------|------------------------|
| **Remote** (default) | HCP Terraform servers |
| **Local** | Your machine (state still remote) |
| **Agent** | Self-hosted agents in your network |

### Workspaces

**CLI Workspaces** vs **HCP Terraform Workspaces:**

| Feature | CLI Workspaces | HCP Terraform Workspaces |
|---------|---------------|--------------------------|
| Purpose | Multiple state files for same config | Separate configurations |
| State | Separate `.tfstate` per workspace | Remote, encrypted per workspace |
| Variables | Shared | Independent per workspace |
| VCS | N/A | Can link to different repos/branches |

**CLI Workspace Commands:**
```bash
terraform workspace list
terraform workspace new <name>
terraform workspace select <name>
terraform workspace show           # Current workspace
terraform workspace delete <name>
```

- Default workspace: `default` (cannot be deleted)
- Reference: `terraform.workspace` in configs

### HCP Terraform Configuration
```hcl
terraform {
  cloud {
    organization = "my-org"
    
    workspaces {
      name = "my-workspace"
      # OR
      tags = ["app:web", "env:prod"]
    }
  }
}
```

---

## 8️⃣ Provisioners (Use as Last Resort!)

### ⚠️ Important Exam Point
- Provisioners are a **last resort** — use provider-native features first
- HashiCorp recommends alternatives: `user_data`, `cloud-init`, config management tools

### Types
```hcl
resource "aws_instance" "web" {
  # ...

  # Runs ON the remote resource
  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
    ]
  }

  # Runs on YOUR machine (where Terraform runs)
  provisioner "local-exec" {
    command = "echo ${self.private_ip} >> ip_list.txt"
  }

  # Copy files to remote resource
  provisioner "file" {
    source      = "conf/app.conf"
    destination = "/etc/app.conf"
  }

  connection {
    type     = "ssh"
    user     = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host     = self.public_ip
  }
}
```

### Provisioner Behavior
- **Creation-time** (default): Run only when resource is created
- **Destroy-time**: `when = destroy` — runs before resource is destroyed
- **On failure**: `on_failure = continue` or `on_failure = fail` (default)
- If a creation-time provisioner fails → resource is **tainted**

---

## 9️⃣ Key CLI Commands Quick Reference

```bash
# Core Workflow
terraform init              # Initialize
terraform plan              # Preview
terraform apply             # Create/Update
terraform destroy           # Destroy all

# Inspect
terraform show              # Show state/plan
terraform output            # Show outputs
terraform state list        # List resources
terraform state show <res>  # Resource details
terraform graph             # Generate DOT graph

# Manipulate
terraform taint <res>       # (Deprecated) Mark for recreation
terraform untaint <res>     # Remove taint
terraform import <res> <id> # Import existing resource
terraform state mv          # Move/rename in state
terraform state rm          # Remove from state

# Utilities
terraform fmt               # Format code
terraform validate          # Validate config
terraform console           # Interactive console
terraform providers         # Show required providers
terraform version           # Show version

# Flags
-target=<resource>          # Only apply to specific resource
-var="key=value"            # Set variable
-var-file="file.tfvars"     # Use variable file
-auto-approve               # Skip confirmation
-lock=false                 # Disable state locking
-refresh=false              # Skip state refresh
-parallelism=N              # Limit concurrent operations (default 10)
-replace=<resource>         # Force replace (replaces taint)
```

---

## 🔟 Common Exam Traps & Tips

### Must-Know Facts
1. **`terraform init`** is the FIRST command (always)
2. **State file contains secrets** in plaintext → encrypt & restrict access
3. **`terraform.tfvars`** is auto-loaded; other `.tfvars` need `-var-file`
4. **`*.auto.tfvars`** files are also auto-loaded
5. **Provisioners are LAST RESORT** — not recommended
6. **`count` and `for_each` cannot be used together** on same resource
7. **`count`** uses index: `aws_instance.web[0]`
8. **`for_each`** uses key: `aws_instance.web["prod"]`
9. **`terraform plan`** does NOT modify infrastructure
10. **`terraform apply`** refreshes state, creates plan, then applies

### Sensitive Values
- `sensitive = true` on variable/output → masked in CLI output
- Still stored in state file as plaintext!
- Cannot use `sensitive` output in other expressions without marking them sensitive too

### Taint (Deprecated → use `-replace`)
```bash
# Old way
terraform taint aws_instance.web
terraform apply

# New way (preferred)
terraform apply -replace="aws_instance.web"
```

### Backend Migration
- Change backend config → run `terraform init`
- Terraform will offer to migrate state
- Use `terraform init -migrate-state`

### Debugging
```bash
export TF_LOG=TRACE          # Most verbose
export TF_LOG=DEBUG
export TF_LOG=INFO
export TF_LOG=WARN
export TF_LOG=ERROR
export TF_LOG_PATH=./terraform.log   # Save to file
```

### Terraform vs Other Tools
| Tool | Approach | Language |
|------|----------|----------|
| **Terraform** | Declarative, multi-cloud | HCL |
| **CloudFormation** | Declarative, AWS only | JSON/YAML |
| **Ansible** | Procedural, config mgmt | YAML |
| **Pulumi** | Declarative, multi-cloud | Python/JS/Go |
| **Chef/Puppet** | Procedural, config mgmt | Ruby/DSL |

---

## 📋 Quick Formula Sheet

```
# Resource address format
<resource_type>.<resource_name>
<resource_type>.<resource_name>[<index>]
module.<module_name>.<resource_type>.<resource_name>

# Reference syntax
var.<variable_name>          # Input variable
local.<local_name>           # Local value
data.<type>.<name>.<attr>    # Data source
module.<name>.<output>       # Module output
<type>.<name>.<attr>         # Resource attribute
self.<attr>                  # Within provisioner
each.key / each.value        # In for_each
count.index                  # In count
terraform.workspace          # Current workspace name
path.module                  # Path of current module
path.root                    # Path of root module
path.cwd                     # Current working directory
```

---

## ✅ Last-Minute Reminders

- [ ] `terraform init` → always first, re-run on backend/provider/module changes
- [ ] State = single source of truth for Terraform
- [ ] Plan shows `+` create, `~` update, `-` destroy, `~/-+` replace
- [ ] Modules encapsulate scope — variables don't leak
- [ ] Lock file (`.terraform.lock.hcl`) → commit to VCS
- [ ] `.terraform/` directory → do NOT commit to VCS
- [ ] `terraform.tfstate` → do NOT commit to VCS (use remote backend)
- [ ] HCP Terraform = remote state + remote execution + collaboration
- [ ] Sentinel = policy as code (Enterprise/HCP feature)
- [ ] `count.index` starts at 0
- [ ] Workspace isolation: each workspace has its own state
- [ ] `terraform console` — great for testing expressions/functions

---

**You've got this! 💪 Focus on understanding concepts, not memorizing syntax.**
