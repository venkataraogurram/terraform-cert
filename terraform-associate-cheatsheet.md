# Terraform Associate Exam Cheatsheet

A condensed reference covering the core exam objectives. Based on Terraform 1.x.

---

## 1. Core Workflow

```
write → init → plan → apply → destroy
```

| Command | Purpose |
|---|---|
| `terraform init` | Download providers, init backend, install modules |
| `terraform validate` | Check syntax & internal consistency (no API calls) |
| `terraform fmt` | Auto-format `.tf` files |
| `terraform plan` | Preview changes (refresh + diff) |
| `terraform apply` | Apply planned changes |
| `terraform destroy` | Destroy all managed resources (alias of `apply -destroy`) |
| `terraform show` | Human-readable view of state or plan file |
| `terraform graph` | Output dependency graph (DOT format) |
| `terraform output` | Show output values from state |
| `terraform console` | Interactive expression evaluator |
| `terraform taint` | **Deprecated** — use `apply -replace=ADDR` instead |
| `terraform untaint` | Remove taint marker (still valid) |
| `terraform login / logout` | Auth with Terraform Cloud |
| `terraform version` | Show CLI + provider versions |

### Useful flags
| Flag | Effect |
|---|---|
| `-auto-approve` | Skip interactive approval |
| `-target=ADDR` | Operate on a specific resource (use sparingly) |
| `-replace=ADDR` | Force replacement of a resource (modern `taint`) |
| `-refresh=false` | Skip state refresh against real infra |
| `-refresh-only` | Update state to match reality, no infra changes |
| `-out=FILE` | Save plan for later apply |
| `-var "k=v"` | Set a variable on CLI |
| `-var-file=FILE` | Load variables from a file |
| `-parallelism=N` | Limit concurrent operations (default 10) |
| `-input=false` | Disable interactive prompts (CI mode) |

---

## 2. Variable Precedence (LOW → HIGH)

```
defaults (variable block)
   ↓
TF_VAR_<name>  (env vars)
   ↓
terraform.tfvars
   ↓
terraform.tfvars.json
   ↓
*.auto.tfvars            (alphabetical)
   ↓
*.auto.tfvars.json       (alphabetical)
   ↓
-var-file=  (CLI, in order)
   ↓
-var=       (CLI, in order)  ← HIGHEST
```

**Auto-loaded files:**
- `terraform.tfvars` / `terraform.tfvars.json`
- `*.auto.tfvars` / `*.auto.tfvars.json`

Other `.tfvars` need `-var-file=`.

---

## 3. Input Variables, Outputs, Locals

```hcl
variable "region" {
  type        = string
  default     = "us-east-1"
  description = "AWS region"
  sensitive   = false
  nullable    = false

  validation {
    condition     = can(regex("^us-", var.region))
    error_message = "Region must start with us-."
  }
}

output "instance_ip" {
  value       = aws_instance.web.public_ip
  description = "Public IP"
  sensitive   = false
}

locals {
  common_tags = {
    Environment = var.env
    Owner       = "platform"
  }
}
```

### Variable types
- Primitive: `string`, `number`, `bool`
- Collection: `list(T)`, `set(T)`, `map(T)`
- Structural: `object({...})`, `tuple([...])`
- `any` — escape hatch

---

## 4. State

| File | Purpose |
|---|---|
| `terraform.tfstate` | Current state |
| `terraform.tfstate.backup` | Previous state (one version back) |
| `errored.tfstate` | Crash recovery dump |
| `.terraform.lock.hcl` | Provider version lock |

### State commands
| Command | Use |
|---|---|
| `terraform state list` | List resources in state |
| `terraform state show ADDR` | Inspect one resource |
| `terraform state mv SRC DST` | Rename / move in state |
| `terraform state rm ADDR` | Remove from state (resource not destroyed) |
| `terraform state pull` | Download remote state to stdout |
| `terraform state push FILE` | Upload state to backend (dangerous) |
| `terraform state replace-provider OLD NEW` | Change provider source |

### Drift handling
- **Default plan/apply:** refreshes, plans to revert drift
- **`terraform apply -refresh-only`:** updates state to match reality (no infra change), needs approval
- **`terraform plan -refresh-only -detailed-exitcode`:** drift detection in CI (exit 2 = drift)
- **`terraform refresh`** standalone is **deprecated**

### `terraform import`
```bash
terraform import aws_instance.web i-0abc1234
```
Brings existing resource into state. You still must write the matching `resource` block in config.

In Terraform 1.5+, you can use **import blocks** for declarative imports:
```hcl
import {
  to = aws_instance.web
  id = "i-0abc1234"
}
```

---

## 5. Backends

### Local (default)
```hcl
terraform {
  backend "local" {
    path = "terraform.tfstate"
  }
}
```

### Remote — S3 (canonical AWS pattern)
```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "tf-lock"   # state locking
  }
}
```

### Backend categories
- **Standard backends:** state storage + locking (s3, azurerm, gcs, consul, http, …)
- **Enhanced backends:** standard + remote operations (`remote` for Terraform Cloud)
- **Local:** state on disk

### State locking
- Prevents concurrent writes.
- Built into most remote backends.
- DynamoDB for S3 backend.
- Can override with `-lock=false` (risky) or `-lock-timeout=10m`.

### Backend reinit
After changing backend config:
```bash
terraform init -migrate-state    # move state to new backend
terraform init -reconfigure      # ignore old config, start fresh
```

---

## 6. Providers

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# Multiple provider configs (alias)
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

resource "aws_s3_bucket" "west" {
  provider = aws.west
  bucket   = "my-west-bucket"
}
```

### Version constraint operators
| Operator | Meaning |
|---|---|
| `= 1.2.3` | Exactly 1.2.3 |
| `!= 1.2.3` | Anything but 1.2.3 |
| `> 1.2.3` / `<` | Greater / less than |
| `>= 1.2.3` / `<=` | Inclusive |
| `~> 1.2` | >= 1.2, < 2.0 (pessimistic, allows minor) |
| `~> 1.2.3` | >= 1.2.3, < 1.3.0 (pessimistic, allows patch) |

### Lock file
- `.terraform.lock.hcl` — commit to VCS.
- Pins provider versions and checksums.
- Update with `terraform init -upgrade`.

---

## 7. Resource Meta-Arguments

| Meta-arg | Purpose |
|---|---|
| `count` | Create N copies (use `count.index`) |
| `for_each` | Create one per map key / set value (use `each.key`, `each.value`) |
| `depends_on` | Explicit dependency |
| `provider` | Use a non-default provider config |
| `lifecycle` | See below |

### `lifecycle` block
```hcl
lifecycle {
  create_before_destroy = true
  prevent_destroy       = true
  ignore_changes        = [tags, ami]
  replace_triggered_by  = [aws_security_group.web]   # 1.2+
  precondition {
    condition     = data.aws_ami.latest.architecture == "x86_64"
    error_message = "Must be x86_64."
  }
  postcondition {
    condition     = self.public_ip != ""
    error_message = "Instance must have a public IP."
  }
}
```

### `count` vs `for_each`
| Use case | Choice |
|---|---|
| Identical N copies | `count` |
| Distinct objects keyed by name | `for_each` |
| Want stable identity if list changes | `for_each` (count uses index, churns on reorder) |

---

## 8. Expressions & Functions

### `for` expression
```hcl
[for s in var.list : upper(s)]                # list
{for k, v in var.map : k => upper(v)}         # map
[for s in var.list : s if s != ""]            # with filter
```

### Splat
```hcl
aws_instance.web[*].id    # all ids
```

### Conditional
```hcl
var.enabled ? "yes" : "no"
```

### Common functions
| Category | Functions |
|---|---|
| String | `format`, `join`, `split`, `lower`, `upper`, `replace`, `regex`, `trimspace`, `substr` |
| Collection | `length`, `concat`, `merge`, `flatten`, `lookup`, `keys`, `values`, `contains`, `distinct`, `setunion`, `slice`, `sort`, `compact` |
| Numeric | `min`, `max`, `abs`, `ceil`, `floor` |
| Encoding | `jsonencode`, `jsondecode`, `yamlencode`, `yamldecode`, `base64encode`, `base64decode` |
| Filesystem | `file`, `templatefile`, `fileset`, `pathexpand` |
| Type / null | `try`, `can`, `coalesce`, `defaults`, `tomap`, `tolist`, `toset`, `tostring`, `tonumber` |
| Date | `timestamp`, `timeadd`, `formatdate` |
| IP | `cidrsubnet`, `cidrhost`, `cidrnetmask` |

### Quick examples
```hcl
flatten([["a","b"],[],["c"]])              # ["a","b","c"]
merge({a=1},{b=2})                          # {a=1,b=2}
lookup(map, "key", "default")
try(local.maybe.value, "fallback")
cidrsubnet("10.0.0.0/16", 8, 1)             # "10.0.1.0/24"
templatefile("${path.module}/init.sh", {ip = aws_instance.web.private_ip})
```

---

## 9. Dynamic Blocks

```hcl
dynamic "ingress" {
  for_each = var.ingress_rules
  iterator = rule          # optional
  content {
    from_port   = rule.value.from_port
    to_port     = rule.value.to_port
    protocol    = rule.value.protocol
    cidr_blocks = rule.value.cidr_blocks
  }
}
```

- For **nested blocks** only, not top-level resources.
- Pass `[]` to `for_each` for **zero blocks** (conditional).
- Use sparingly — hurts readability.

---

## 10. Modules

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  cidr = "10.0.0.0/16"
  azs  = ["us-east-1a", "us-east-1b"]
}
```

### Source types
| Source | Example |
|---|---|
| Local | `./modules/vpc` |
| Registry | `terraform-aws-modules/vpc/aws` |
| Git | `git::https://github.com/x/y.git//modules/vpc?ref=v1.2.0` |
| GitHub | `github.com/x/y` |
| HTTP | `https://example.com/m.zip` |
| S3 / GCS | `s3::https://...` |

### Module outputs
```hcl
output "vpc_id" {
  value = module.vpc.vpc_id
}
```

### Module meta-arguments
- `source` (required), `version`, `count`, `for_each`, `providers`, `depends_on`

### Public Module Registry
- `registry.terraform.io`
- Verified modules have a blue checkmark.
- Format: `<NAMESPACE>/<NAME>/<PROVIDER>`

---

## 11. Provisioners (last resort)

| Type | Runs on |
|---|---|
| `local-exec` | Machine running Terraform |
| `remote-exec` | Remote resource (SSH/WinRM) |
| `file` | Copy local → remote |

```hcl
provisioner "local-exec" {
  command = "echo ${self.id}"
  when    = create     # or "destroy"
  on_failure = fail    # or "continue"
}
```

- `self` only valid in provisioners.
- Destroy provisioners can only reference `self`, `count.index`, `each.key`.
- Failed provisioner **taints** the resource by default.
- Prefer `user_data`, cloud-init, or config-management tools instead.

---

## 12. Workspaces (CLI workspaces)

```bash
terraform workspace list
terraform workspace new dev
terraform workspace select prod
terraform workspace show
terraform workspace delete dev
```

- Default workspace: `default`.
- Each workspace has its own state file.
- Reference in config: `terraform.workspace`.
- **Not recommended for environment separation in production** — use separate root configs/backends.

---

## 13. Logging / Debugging

| Env Var | Purpose |
|---|---|
| `TF_LOG` | Log level: `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR` |
| `TF_LOG_PATH` | Log file location |
| `TF_LOG_CORE` | Log level for core only |
| `TF_LOG_PROVIDER` | Log level for providers only |
| `TF_VAR_<name>` | Set input variables |
| `TF_INPUT=0` | Disable interactive prompts |
| `TF_CLI_ARGS` / `TF_CLI_ARGS_<cmd>` | Default CLI args |
| `TF_DATA_DIR` | Override `.terraform/` location |
| `TF_WORKSPACE` | Select workspace |

```bash
export TF_LOG=DEBUG
export TF_LOG_PATH=./tf.log
terraform plan
```

---

## 14. Sensitive Data

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

output "secret" {
  value     = var.db_password
  sensitive = true
}
```

- `sensitive` hides from CLI output (state still contains plaintext).
- Use `nonsensitive()` to unmark when needed.
- **Never commit state to Git** — it can contain secrets.
- Store secrets in Vault, AWS Secrets Manager, or similar.

---

## 15. Terraform Cloud / Enterprise

Key features:
- **Remote state storage** (free).
- **Remote operations** (plan/apply on TFC infrastructure).
- **VCS integration** (GitHub, GitLab, Bitbucket).
- **Sentinel / OPA policies** (policy as code).
- **Private module registry**.
- **Workspaces** (different concept than CLI workspaces — closer to "environment").
- **Run triggers** between workspaces.
- **Cost estimation** (paid).
- **Teams & SSO** (paid).

```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "prod"
    }
  }
}
```

Login: `terraform login`.

### Sentinel
- Policy-as-code framework.
- Three enforcement levels: `advisory`, `soft-mandatory`, `hard-mandatory`.

---

## 16. Common Patterns

### Conditional resource
```hcl
resource "aws_instance" "web" {
  count = var.create_instance ? 1 : 0
  # ...
}
```

### Force replacement
```bash
terraform apply -replace=aws_instance.web
```

### Read-only data source
```hcl
data "aws_ami" "latest" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*"]
  }
}
```

### Cross-stack references via remote state
```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

# Use: data.terraform_remote_state.network.outputs.vpc_id
```

---

## 17. Exam-Favorite Gotchas

| Gotcha | Truth |
|---|---|
| `terraform refresh` standalone | **Deprecated** — use `apply -refresh-only` |
| `terraform taint` | **Deprecated** — use `apply -replace=ADDR` |
| `prod.tfvars` auto-loaded? | **No** — needs `-var-file=` |
| Variable defaults override env vars? | **No** — defaults are lowest |
| `-target` for routine use? | **No** — exceptional / recovery only |
| `count.index` stable on list reorder? | **No** — use `for_each` for stable identity |
| Workspaces for prod vs dev? | **Discouraged** — use separate configs |
| Provisioners | **Last resort** per HashiCorp |
| `dynamic` for top-level resources? | **No** — only for nested blocks |
| Sensitive value hidden from state? | **No** — only from CLI output |
| `TF_LOG=trace` (lowercase)? | Case matters; use uppercase |

---

## 18. Quick Command Reference Card

```bash
# Init & format
terraform init [-upgrade] [-reconfigure] [-migrate-state] [-backend-config=...]
terraform fmt [-recursive] [-check]
terraform validate

# Plan & apply
terraform plan [-out=plan.tfplan] [-refresh=false] [-refresh-only] [-target=ADDR] [-replace=ADDR]
terraform apply [plan.tfplan] [-auto-approve] [-refresh-only] [-replace=ADDR]
terraform destroy [-target=ADDR] [-auto-approve]

# State
terraform state list
terraform state show ADDR
terraform state mv SRC DST
terraform state rm ADDR
terraform state pull > backup.tfstate
terraform state push backup.tfstate

# Import
terraform import aws_instance.web i-0abc1234

# Output / show / graph
terraform output [-json] [NAME]
terraform show [-json] [plan.tfplan]
terraform graph

# Workspace
terraform workspace {list|show|new|select|delete} [NAME]

# Login (Terraform Cloud)
terraform login
terraform logout

# Force re-creation
terraform apply -replace=aws_instance.web

# Drift detection in CI
terraform plan -refresh-only -detailed-exitcode
# Exit codes: 0 = no changes, 1 = error, 2 = changes
```

---

## 19. Last-Minute Memory Aids

- **Variable precedence:** Earth → Tar → Auto → CLI (env → `terraform.tfvars` → `*.auto.tfvars` → `-var`).
- **Provisioners:** last resort.
- **`-target`:** exceptional, not routine.
- **`taint` / `refresh`:** deprecated standalone — use `apply -replace` / `apply -refresh-only`.
- **`for_each`:** stable identity. **`count`:** indexed identity.
- **`~> 1.2`:** allows minor. **`~> 1.2.3`:** allows patch only.
- **Backend with state locking:** S3 + DynamoDB.
- **Lock file** (`.terraform.lock.hcl`) → commit. **State file** → never commit, use remote backend.
- **Sentinel:** advisory / soft-mandatory / hard-mandatory.
- **`TF_LOG`:** TRACE > DEBUG > INFO > WARN > ERROR.

---

Good luck on the exam!
