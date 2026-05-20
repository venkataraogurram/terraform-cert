# Terraform Associate — Mock Exam Cheatsheet

**Every concept tested across all 6 mock exams (342 questions), organized by topic.**
**Use this as your final review document.**

---

## 1. TERRAFORM WORKFLOW

### Core Workflow (3 steps)
```
Write → Plan → Apply
```

### Full Command Sequence
```
terraform init → terraform validate → terraform plan → terraform apply → terraform destroy
```

### What Each Command Does

| Command | Purpose | Modifies State? | Contacts Provider API? |
|---|---|---|---|
| `terraform init` | Downloads providers + modules, sets up backend | No | Yes (registry) |
| `terraform validate` | Syntax + type check | No | **No** |
| `terraform fmt` | Canonical formatting | No | No |
| `terraform plan` | Shows execution plan | **No** (but acquires lock!) | Yes |
| `terraform apply` | Creates/updates/destroys resources | **Yes** | Yes |
| `terraform destroy` | Destroys all managed resources | **Yes** | Yes |
| `terraform import` | Adds existing resource to state | **Yes** | Yes |
| `terraform show` | Displays full state | No | No |
| `terraform state list` | Lists resource addresses | No | No |
| `terraform state show` | Shows one resource | No | No |
| `terraform output` | Shows outputs | No | No |
| `terraform graph` | Generates DOT dependency graph | No | No |
| `terraform console` | Interactive expression evaluator | No | No |

### When to Re-run `terraform init`
- Added/removed/changed a provider
- Added/removed/changed a module source or version
- Changed the backend configuration
- Want to upgrade providers (`-upgrade` flag)

### terraform init Flags

| Flag | Purpose |
|---|---|
| `-upgrade` | Upgrade providers/modules to latest matching constraints |
| `-migrate-state` | Copy state to new backend |
| `-reconfigure` | Reset backend, ignore existing state |
| `-backend=false` | Skip backend initialization |

### terraform plan Behavior
1. **Acquires state lock** (even though read-only!)
2. Refreshes state by querying real infrastructure (in memory only)
3. Compares refreshed state to desired config
4. Creates execution plan
5. Does NOT write to state file
6. Releases lock

### terraform plan Flags

| Flag | Purpose |
|---|---|
| `-out=file.tfplan` | Save plan for later apply |
| `-refresh=false` | Skip state refresh |
| `-refresh-only` | Show drift only, no changes proposed |
| `-destroy` | Show destroy plan |
| `-target=<addr>` | Plan only specific resource |
| `-var="k=v"` | Set variable |
| `-var-file=<file>` | Load variables from file |
| `-replace=<addr>` | Force replacement of specific resource |

### terraform apply Flags

| Flag | Purpose |
|---|---|
| `file.tfplan` | Apply a saved plan exactly |
| `-auto-approve` | Skip confirmation |
| `-replace=<addr>` | Force destroy and recreate specific resource |
| `-refresh-only` | Update state to match reality (write) |
| `-destroy` | Same as `terraform destroy` |
| `-parallelism=N` | Concurrency (default 10) |
| `-target=<addr>` | Apply only specific resource |

### terraform fmt

| Flag | Purpose |
|---|---|
| (no flags) | Formats current directory ONLY |
| `-recursive` | Formats current directory AND subdirectories |
| `-check` | Check formatting without modifying (CI/CD) |
| `-diff` | Show differences |

### Plan Output Symbols

| Symbol | Meaning |
|---|---|
| `+` | Create |
| `-` | Destroy |
| `~` | **Update in-place** |
| `-/+` | Replace (destroy then create) |
| `+/-` | Replace (create then destroy, with `create_before_destroy`) |
| `<=` | Read (data source) |

---

## 2. STATE MANAGEMENT

### State File Facts
- Format: **JSON**, plaintext, unencrypted
- Default name: `terraform.tfstate`
- Default location: current working directory (local backend)
- Contains: resource IDs, attributes, dependencies, **sensitive values in plaintext**
- Backup: `terraform.tfstate.backup` (auto-created before each change)
- **NEVER commit to version control**
- **NEVER edit manually** — use `terraform state` commands

### After `terraform destroy`
- State file **remains** but contains no resources
- State file is NOT deleted automatically

### State Commands

| Command | Purpose | Modifies State? |
|---|---|---|
| `terraform state list` | List all resource addresses (concise) | No |
| `terraform state show <addr>` | Show one resource details | No |
| `terraform show` | Show entire state (verbose) | No |
| `terraform state mv` | Rename/move in state | **Yes** |
| `terraform state rm` | Remove from state (keeps real resource) | **Yes** |
| `terraform state pull` | Download state to stdout | No |
| `terraform state push` | Upload state | **Yes** |
| `terraform state replace-provider` | Change provider source in state | **Yes** |
| `terraform force-unlock <ID>` | Release stuck lock | **Yes** |

### State Locking
- Prevents concurrent runs from corrupting state
- `terraform plan` DOES acquire a lock
- `terraform apply` DOES acquire a lock
- Use `-lock=false` to disable (dangerous)
- Stuck lock: `terraform force-unlock <LOCK_ID>`

### Backend Locking Support

| Backend | Locking? | How? |
|---|---|---|
| S3 + DynamoDB | Yes | DynamoDB table with `LockID` key |
| S3 alone | **NO** | Must add DynamoDB |
| azurerm | Yes | Built-in (blob lease) |
| gcs | Yes | Built-in |
| consul | Yes | Built-in |
| HCP Terraform | Yes | Built-in |
| pg (PostgreSQL) | Yes | Built-in |
| local | **NO** | OS-level file lock only |
| http | Depends | Only if server implements it |

### Sensitive Data in State
- `sensitive = true` on variable/output → hides from CLI only
- Value is **STILL stored as plaintext in state**
- No method prevents sensitive data from being in state
- **Exception**: ephemeral resources + write-only attributes (Terraform 1.10+)

### Drift Detection
```bash
terraform plan -refresh-only    # show drift (read-only)
terraform apply -refresh-only   # update state to match reality (writes state)
```

### Empty Config + Existing State
If you remove all resource blocks and run `terraform apply`:
- Terraform sees desired state = empty
- Plans to **destroy all resources** in state

---

## 3. BACKENDS

### Local Backend (Default)
- State in `terraform.tfstate` in working directory
- No encryption, no RBAC, minimal locking
- Fine for solo dev, bad for teams

### S3 Backend
```hcl
terraform {
  backend "s3" {
    bucket         = "my-state"        # required
    key            = "prod.tfstate"    # required
    region         = "us-east-1"      # required
    dynamodb_table = "tf-lock"        # for locking
    encrypt        = true             # encryption at rest
  }
}
```

### Backend Block Limitation
Backend blocks accept **ONLY literal values**:
- `var.bucket` → **NO**
- `local.region` → **NO**
- Only: `bucket = "my-literal-string"` → YES
- For dynamic values: use `-backend-config` flags

### HCP Terraform Backend (`cloud` block)
```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces { name = "prod" }
  }
}
```
- **NOT** `backend "hcp"` — that doesn't exist
- Setup: `terraform login` → `terraform init`

### Backend Migration
```bash
terraform init -migrate-state    # copy state to new backend
terraform init -reconfigure      # fresh start, ignore existing state
```

### Partial Backend Configuration
```bash
terraform init -backend-config="bucket=my-bucket" \
               -backend-config="key=prod.tfstate"
```

---

## 4. PROVIDERS

### What Providers Are
- **Plugins** that talk to platform APIs
- Separate from Terraform Core (downloaded during `init`)
- NOT modules, NOT backends, NOT variables

### Provider Configuration
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

### Two Blocks, Two Purposes
| Block | Where | Purpose |
|---|---|---|
| `required_providers` | Inside `terraform {}` | Declare source + version constraints |
| `provider` | Top-level | Configure settings (region, creds, tags) |

### Provider Tiers

| Tier | Maintained By | Indicator |
|---|---|---|
| Official | HashiCorp | `hashicorp/` namespace |
| Partner | Tech vendor | Verified badge |
| Community | Community | No badge |

### Provider Aliases (Multiple Configs)
```hcl
provider "aws" {
  region = "us-east-1"          # default (no alias)
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

resource "aws_vpc" "west" {
  provider = aws.west            # use aliased provider
}
```

### Provider Selection Without `provider` Argument
When a resource doesn't specify `provider`, Terraform uses the **default provider** matching the resource type prefix (e.g., `aws_instance` → default `aws` provider).

### Provider Config vs Environment Variable
Explicit value in `provider` block **WINS** over environment variables:
- `provider { region = "us-west-2" }` + `AWS_REGION=eu-west-1` → **us-west-2 wins**

### Provider Install Sources
1. Public Terraform Registry (default)
2. Private registries (HCP Terraform, custom)
3. Filesystem mirror (local directory)
4. Provider mirror (network mirror)
- NOT: GitHub releases, source code compilation

### Each Platform = Its Own Provider
AWS needs `aws` provider, GCP needs `google`, Azure needs `azurerm`. One provider per platform. TRUE.

### Lock File (`.terraform.lock.hcl`)
- Records exact provider versions + checksums
- **Commit to version control** (ensures team consistency)
- Updated by `terraform init` and `terraform init -upgrade`
- NOT a state lock, NOT a concurrency lock

### Provider Storage
- Downloaded to `.terraform/providers/` in working directory
- `.terraform/` directory → do NOT commit

---

## 5. VERSION CONSTRAINTS

### Operators

| Operator | Meaning |
|---|---|
| `= 5.0.0` | Exactly 5.0.0 |
| `!= 5.0.0` | Anything except 5.0.0 |
| `> 5.0.0` | Greater than |
| `>= 5.0.0` | Greater or equal |
| `< 5.0.0` | Less than |
| `~> 5.0` | >= 5.0, < 6.0 (minor + patch) |
| `~> 5.0.0` | >= 5.0.0, < 5.1.0 (patch only) |

### `~>` Rule: Only RIGHTMOST Digit Can Change

| Constraint | Allows | Denies |
|---|---|---|
| `~> 3.2.0` | 3.2.0, 3.2.5 | 3.3.0, 4.0.0 |
| `~> 3.2` | 3.2.0, 3.3.0, 3.9.9 | 4.0.0 |
| `~> 6.0.0` | 6.0.0, 6.0.5 | 6.1.0, 7.0.0 |
| `~> 6.0` | 6.0.5, 6.1.0, 6.9.9 | 7.0.0 |

### Multiple Constraints in One String
```hcl
version = ">= 1.0.0, < 2.0.0"    # both must be satisfied (AND)
```
Cannot use `version` argument multiple times in same block.

### `terraform init` vs `terraform init -upgrade`
- `init` alone → uses locked version (won't upgrade)
- `init -upgrade` → upgrades to newest matching constraints, updates lock file

---

## 6. VARIABLES

### Declaration
```hcl
variable "region" {
  type        = string
  description = "AWS region"
  default     = "us-east-1"
  sensitive   = true
  nullable    = false

  validation {
    condition     = can(regex("^us-", var.region))
    error_message = "Must be a US region."
  }
}
```

### Variable Types

| Category | Types |
|---|---|
| Primitives | `string`, `number`, `bool` |
| Collections | `list(type)`, `set(type)`, `map(type)` |
| Structural | `object({...})`, `tuple([...])` |
| Dynamic | `any` |

### Syntax for Types
- Lists: `["a", "b", "c"]` (square brackets)
- Maps/Objects: `{ key = "value" }` (curly braces)
- Accessing list elements: `var.list[0]` (zero-indexed)
- Accessing map values: `var.map["key"]`
- Map keys with hyphens: `var.map["build-tag"]` (NOT `var.map.build-tag`)

### Variable Precedence (lowest → highest)
1. `default` in variable block
2. `TF_VAR_<name>` environment variables
3. `terraform.tfvars` / `terraform.tfvars.json`
4. `*.auto.tfvars` / `*.auto.tfvars.json` (alphabetical)
5. `-var-file` flag
6. `-var` flag ← **WINS**

### Setting Variables via Environment
```bash
export TF_VAR_region="us-east-1"    # prefix is TF_VAR_
```
- `TF_VAR_` = sets variable value
- `TF_LOG` = sets log level (different purpose!)
- `TF_LOG_PATH` = sets log file location (different purpose!)

### Variable Validation
```hcl
variable "instance_count" {
  type = number
  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "Must be 1-10."
  }
}
```
- Evaluated during `terraform validate` and `terraform plan`
- Fails BEFORE any resources are created

### Sensitive Variables
- `sensitive = true` → hides from CLI output
- Still stored in state as plaintext
- Does NOT encrypt anything

---

## 7. OUTPUTS

```hcl
output "instance_ip" {
  value       = aws_instance.web.public_ip
  description = "Public IP"
  sensitive   = true
}
```

### Purpose
- Display values after apply
- Expose values from child modules to parent
- Machine-readable via `terraform output -json`

### Commands
```bash
terraform output                  # all outputs
terraform output instance_ip      # specific output
terraform output -json            # JSON format
terraform output -raw instance_ip # raw string (no quotes)
```

---

## 8. RESOURCES & DATA SOURCES

### Resource Block (Creates/Manages Infrastructure)
```hcl
resource "aws_instance" "web" {
  ami           = "ami-123"
  instance_type = "t3.micro"
}
```
Reference: `aws_instance.web.id`

### Data Source (Reads Existing Infrastructure)
```hcl
data "aws_vpc" "prod" {
  tags = { Name = "production" }
}
```
Reference: `data.aws_vpc.prod.id` (note `data.` prefix!)

### Key Differences
| | Resource | Data Source |
|---|---|---|
| Purpose | Create/manage | Read-only query |
| Prefix | None | `data.` |
| In state? | Yes (managed) | Yes (cached) |
| Provider modifies it? | Yes | No |

---

## 9. REFERENCE SYNTAX

| Block Type | Reference Pattern |
|---|---|
| Resource | `aws_instance.web.id` |
| Data source | `data.aws_vpc.prod.id` |
| Module output | `module.vpc.vpc_id` |
| Variable | `var.region` |
| Local | `local.common_tags` |
| Workspace | `terraform.workspace` |
| Count resource | `aws_instance.web[0].id` |
| for_each resource | `aws_instance.web["key"].id` |
| Splat | `aws_instance.web[*].id` |
| Self (in provisioner) | `self.id` |

### Common WRONG Syntax
- `output.module.x` → WRONG
- `var.module.x` → WRONG
- `module.x.outputs.y` → WRONG
- `resource.aws_instance.web.id` → WRONG
- `module.x.aws_instance.web.id` → WRONG (can't reach into child resources)

---

## 10. DEPENDENCIES

### Implicit (Automatic)
Created when one resource references another's attribute:
```hcl
resource "aws_subnet" "main" {
  vpc_id = aws_vpc.main.id    # implicit dependency on aws_vpc.main
}
```

### Explicit (`depends_on`)
For hidden dependencies not expressible through references:
```hcl
resource "aws_instance" "web" {
  depends_on = [aws_iam_role_policy.example]
}
```

### Destroy Order
Terraform destroys in **reverse dependency order**:
- VPC → Subnet → Instance (create order)
- Instance → Subnet → VPC (destroy order)

---

## 11. META-ARGUMENTS

### count
```hcl
resource "aws_instance" "web" {
  count         = 3
  instance_type = "t3.micro"
  tags = { Name = "web-${count.index}" }
}
# Reference: aws_instance.web[0].id
```

### for_each
```hcl
resource "aws_instance" "web" {
  for_each      = toset(["dev", "staging", "prod"])
  instance_type = "t3.micro"
  tags = { Name = each.key }
}
# Reference: aws_instance.web["dev"].id
```

### for_each Type Requirements
- Accepts: `map(any)` or `set(string)`
- Does NOT accept: `list` → convert with `toset()`

### count vs for_each

| | count | for_each |
|---|---|---|
| Accepts | number | map or set |
| Index | integer [0], [1] | string ["key"] |
| On removal | Shifts indices (destroys/recreates) | Only removes affected item |
| Best for | Identical resources | Distinct identities |

### Conditional Resource Creation
```hcl
resource "aws_instance" "web" {
  count = var.create ? 1 : 0
}
```

---

## 12. LIFECYCLE

```hcl
lifecycle {
  create_before_destroy = true     # zero-downtime replacement
  prevent_destroy       = true     # error if plan would destroy
  ignore_changes        = [tags]   # ignore drift on these attrs
  replace_triggered_by  = [aws_sg.main]  # replace when dependency changes

  precondition {
    condition     = var.memory >= 4096
    error_message = "Need 4GB+ memory."
  }
  postcondition {
    condition     = self.public_ip != ""
    error_message = "Must have public IP."
  }
}
```

---

## 13. VALIDATION MECHANISMS

| Mechanism | Where | When | On Failure |
|---|---|---|---|
| `validation {}` | `variable` block | Before plan | **Error** (halts) |
| `precondition {}` | `lifecycle` block | Before create/update | **Error** (halts) |
| `postcondition {}` | `lifecycle` block | After create/update | **Error** (halts) |
| `check {}` | Top-level block | After apply | **Warning** (continues) |

### When to Use Each
- Validate variable input → `validation` block
- Check assumptions before creating → `precondition`
- Verify results after creating → `postcondition`
- Verify data source found something → `postcondition`
- Ongoing health monitoring → `check` block (warning only!)

---

## 14. MODULES

### Module Scope Rules
- Child modules **NEVER** inherit parent variables
- Child modules **NEVER** inherit parent locals
- Child modules **NEVER** inherit parent data sources
- Child can only access what's passed via module block
- Parent can only access child's declared outputs

### Module Output Syntax
```hcl
# ONLY correct way:
module.<MODULE_NAME>.<OUTPUT_NAME>

# Example:
module.vpc.vpc_id
```

### Module Sources
```hcl
source = "./modules/vpc"                     # local
source = "terraform-aws-modules/vpc/aws"     # public registry
source = "app.terraform.io/org/vpc/aws"      # private registry
source = "git::https://...?ref=v1.0"         # git with tag
```

### Module Version
- Optional but **strongly recommended**
- Only works with **registry** sources (not local/git)
- Without it: Terraform downloads **latest** (risky!)

### `terraform init -upgrade` for Modules
Required when you change the `version` constraint to get the new version.

### Module Testing (Terraform 1.6+)
- Use `.tftest.hcl` files with `run` and `assert` blocks
- Run with `terraform test` command

### Nested Module Outputs
Root → Module A → Module B: Root CANNOT access Module B outputs directly. Module A must re-export them.

---

## 15. MOVED, IMPORT, REMOVED BLOCKS

### moved (Terraform 1.1+) — Rename/Restructure
```hcl
moved {
  from = aws_instance.web
  to   = aws_instance.app
}
```
- Works within **same state** only
- Keep for at least one full apply cycle
- Can remove after all team members have applied

### import block (Terraform 1.5+) — Bring Existing Resource Under Management
```hcl
import {
  to = aws_instance.web
  id = "i-1234567890"
}
```
- Syntax: `to` and `id` (NOT `resource`/`resource_id`, NOT `from`)
- Run `terraform plan` then `terraform apply` to execute
- Can remove import block after successful import
- For `for_each` resources: need one import block per instance

### removed block (Terraform 1.7+) — Stop Managing Without Destroying
```hcl
removed {
  from = aws_instance.web
  lifecycle { destroy = false }
}
```

### Cross-State Migration
- Source state: `removed` block (stop managing)
- Destination state: `import` block (start managing)
- `moved` block does NOT work across different states

### Decision Matrix

| Scenario | Correct Approach |
|---|---|
| Rename resource | `moved` block |
| Move into module | `moved` block |
| Import existing resource | `import` block |
| Stop managing, keep alive | `removed` block |
| Destroy specific resource | Delete block + `terraform apply` |
| Force recreate | `terraform apply -replace=<addr>` |

---

## 16. HCP TERRAFORM

### Three Workflow Types
1. **VCS-driven** — Git push triggers runs
2. **CLI-driven** — `terraform plan/apply` from terminal
3. **API-driven** — REST API triggers runs

(NOT "agent-driven" — agents are an execution mode, not a workflow)

### Three Execution Modes

| Mode | Runs On | State In |
|---|---|---|
| Remote (default) | HCP Terraform | HCP Terraform |
| Local | Your machine | HCP Terraform |
| Agent | Self-hosted runner | HCP Terraform |

### CLI-Driven Workflow
- Runs execute **REMOTELY on HCP Terraform** (NOT locally!)
- Output streamed back to your terminal
- Config uploaded to HCP Terraform

### Local Execution Mode
- HCP Terraform **only stores/syncs state**
- Plans/applies run on YOUR machine
- No Sentinel, no cost estimation in this mode

### Configuration
```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces { name = "prod" }
  }
}
```
Setup: `terraform login` → `terraform init`

### Key Features

| Feature | Free | Plus | Enterprise |
|---|---|---|---|
| Remote state | ✓ | ✓ | ✓ |
| Remote runs | ✓ | ✓ | ✓ |
| VCS integration | ✓ | ✓ | ✓ |
| Private registry | ✓ | ✓ | ✓ |
| Run triggers | ✓ | ✓ | ✓ |
| Sentinel/OPA | ✗ | **✓** | ✓ |
| Cost estimation | ✗ | ✓ | ✓ |
| SSO | ✗ | ✓ | ✓ |
| Agents | ✗ | ✓ | ✓ |
| Audit logging | ✗ | ✗ | **✓** |

### Run Triggers
- Auto-queue run in workspace B after workspace A **applies successfully**
- For chaining dependent infrastructure
- Scoped within same organization

### Run Tasks
- Integrate **external tools** between plan and apply
- Use case: security scanning, cost estimation, compliance
- Can be advisory (warning) or mandatory (blocks apply)

### Variable Sets
- Reusable groups of variables shared across workspaces
- Scopes: Global → Project → Workspace-specific

### Speculative Plans
- Triggered by pull requests in VCS-driven workflow
- Shows what would happen if PR is merged
- Posted as PR comment
- **Cannot be applied** — merge first

### Health Checks / Drift Detection
- Detects when real infra differs from state
- Does **NOT** auto-fix
- Notifies team; manual remediation required

### Cross-Workspace Data Sharing
```hcl
data "tfe_outputs" "network" {
  organization = "my-org"
  workspace    = "networking"
}
# Access: data.tfe_outputs.network.values.vpc_id
```

### HCP Terraform Explorer
- Search across all workspaces for specific resources
- Find which workspace manages a given resource

### HCP Terraform Agents
- Run on YOUR infrastructure (behind your firewall)
- For private/on-prem network access
- Plus/Enterprise feature

### Projects
- Group related workspaces
- Apply permissions at project level
- 1 workspace = exactly 1 project

### Workspace Rules
- 1 workspace = 1 VCS repo (max)
- 1 workspace = 1 state file
- Workspace has its own Terraform version setting
- `required_version` in config must match workspace version setting

### Change Requests
- Tracked backlog of planned changes for workspaces
- Used to manage infrastructure work items
- NOT Sentinel, NOT cost estimation

### Sensitive Variables
- Write-only in HCP Terraform UI (can edit but never view after setting)

---

## 17. LOGGING

### Environment Variables
| Variable | Purpose | Example |
|---|---|---|
| `TF_LOG` | Set log **level** | `TF_LOG=TRACE` |
| `TF_LOG_PATH` | Set log **file location** | `TF_LOG_PATH=/tmp/tf.log` |
| `TF_LOG_PROVIDER` | Provider-specific log level | `TF_LOG_PROVIDER=DEBUG` |

### Log Levels (most → least verbose)
`TRACE` > `DEBUG` > `INFO` > `WARN` > `ERROR`

### Disable Logging
```bash
unset TF_LOG     # NOT "OFF", NOT "NONE" — just unset it
```

### What Verbose Logs Show
- Provider API requests and responses
- Internal state file operations (reads, writes, locks)
- Plugin communications between Core and providers
- NOT: auto-fixes, cost estimates, team member lists

### When to Use Logging
- Provider API calls failing
- Troubleshooting state locking issues
- Investigating unexpected plan changes
- NOT: routine deployments, `terraform fmt`, `terraform init`

---

## 18. FUNCTIONS (Most Tested)

### String
- `join("-", ["a", "b"])` → `"a-b"`
- `split(",", "a,b")` → `["a", "b"]`
- `upper("hi")` / `lower("HI")`
- `format("Hello %s", name)`
- `replace(s, old, new)`
- `trimspace(" hi ")`

### Collection
- `length(list_or_map)` — count elements
- `lookup(map, key, default)` — safe map access
- `merge(map1, map2)` — combine maps (later wins)
- `concat(list1, list2)` — combine lists
- `flatten([[a],[b]])` — flatten nested lists
- `distinct(list)` — remove duplicates
- `contains(list, value)` — boolean
- `keys(map)` / `values(map)`
- `element(list, index)` — by index

### Type Conversion
- `toset(list)` — convert list to set (for `for_each`)
- `tolist(set)` — convert set to list
- `tomap(obj)` — convert to map
- `tonumber("42")` / `tostring(42)`

### Network
- `cidrsubnet("10.0.0.0/16", 8, 1)` → `"10.0.1.0/24"`
- `cidrhost("10.0.0.0/16", 5)` → `"10.0.0.5"`

### Type Checking
- `can(expr)` — returns bool (did expression succeed?)
- `try(expr, fallback)` — returns fallback on error

### Conditional
```hcl
var.env == "prod" ? "t3.large" : "t3.micro"
```

### for Expression
```hcl
[for s in var.list : upper(s)]                    # list
[for s in var.list : upper(s) if s != "skip"]     # with filter
{for k, v in var.map : k => upper(v)}             # map
```

### Common Function Confusion
| To Combine | Use |
|---|---|
| Maps | `merge()` |
| Lists | `concat()` |
| List elements → string | `join()` |
| String → list | `split()` |
| Nested lists → flat list | `flatten()` |

---

## 19. INFRASTRUCTURE AS CODE CONCEPTS

### IaC vs Configuration Management
| | IaC (Terraform) | Config Mgmt (Chef/Puppet/Ansible) |
|---|---|---|
| Focus | Provision infrastructure | Configure software on servers |
| Creates | VMs, networks, storage | Installs packages, manages services |
| Approach | Declarative | Often imperative |

### IaC Advantages
- Version control / history / reviews
- Repeatability / consistency across environments
- Self-service for developers
- API-driven workflows
- Collaboration through code reviews
- Immutable infrastructure reduces drift
- Self-documenting infrastructure

### IaC does NOT:
- Eliminate API communication (it USES APIs)
- Auto-fix vulnerabilities in application code
- Automatically ensure security
- Automatically optimize costs

### Multi-Cloud Benefits
- Same workflow (init/plan/apply) across all providers
- Same language (HCL) for all providers
- Manage dependencies across clouds
- Reduce learning curve (one tool)
- Does NOT: auto-sync between clouds, eliminate per-provider credentials

### Immutable Infrastructure Advantages
- Eliminates configuration drift (servers never modified after creation)
- Simplifies rollback (deploy previous version)
- Reduces inconsistent environments

---

## 20. KEY FILES

| File | Commit? | Purpose |
|---|---|---|
| `*.tf` | Yes | Configuration files |
| `terraform.tfvars` | Caution | Variable values (may have secrets) |
| `*.auto.tfvars` | Caution | Auto-loaded variable values |
| `.terraform.lock.hcl` | **YES** | Provider version lock |
| `terraform.tfstate` | **NO** | State (has secrets) |
| `terraform.tfstate.backup` | No | Previous state |
| `.terraform/` | No | Provider/module cache |
| `*.tfplan` | No | Saved plan files |
| `.tftest.hcl` | Yes | Module test files |

---

## 21. PROVISIONERS (Last Resort)

| Type | Runs On | Purpose |
|---|---|---|
| `local-exec` | Machine running Terraform | Run local commands |
| `remote-exec` | Remote resource | Run remote commands |
| `file` | Remote resource | Copy files |

- Use sparingly — prefer cloud-init/user-data
- `when = destroy` for destroy-time provisioners
- `on_failure = continue` to ignore errors
- Destroy-time provisioners can only reference `self`

---

## 22. EPHEMERAL RESOURCES (Terraform 1.10+)

The ONLY way to keep secrets completely out of state:
```hcl
ephemeral "random_password" "db" {
  length = 20
}

resource "aws_db_instance" "main" {
  password_wo         = ephemeral.random_password.db.result
  password_wo_version = 1
}
```

---

## 23. PARTIAL APPLY BEHAVIOR

When `terraform apply` fails midway:
- Successfully created resources → in state, remain in cloud
- Failed resources → NOT in state
- Run `terraform apply` again → only creates remaining resources
- Terraform does NOT roll back

---

## 24. DYNAMIC BLOCKS

```hcl
dynamic "ingress" {
  for_each = var.rules
  content {
    from_port = ingress.value.port
    protocol  = ingress.value.protocol
  }
}
```

---

## 25. WORKSPACES (OSS)

```bash
terraform workspace new dev
terraform workspace select prod
terraform workspace list
terraform workspace show        # returns current name
terraform workspace delete dev  # errors if state has resources
```

- `terraform.workspace` → current workspace name string
- NOT for strong prod/dev isolation (use separate backends)
- State stored in `terraform.tfstate.d/<workspace>/`

---

## FINAL EXAM REMINDERS

1. `~> X.Y.Z` = only Z can change; `~> X.Y` = only Y can change
2. `for_each` needs map or set — use `toset()` for lists
3. `cloud {}` for HCP Terraform, NOT `backend "hcp"`
4. Module outputs: `module.<name>.<output>` — ONLY syntax
5. Child modules NEVER inherit parent variables
6. `sensitive = true` does NOT keep secrets out of state
7. Plan DOES acquire state lock
8. `terraform fmt` = current dir only; `-recursive` for subdirs
9. Provider block values WIN over environment variables
10. `state list` (concise) vs `state show <addr>` (one resource) vs `show` (everything)
11. Run triggers = chain workspaces; Run tasks = external tools
12. VCS PR → speculative plan (posted as comment, can't apply)
13. HCP CLI workflow runs REMOTELY, not locally
14. Empty config + existing state = destroys all
15. After destroy: state file remains (empty, not deleted)
16. `TF_LOG` = level; `TF_LOG_PATH` = file; unset to disable
17. Import block syntax: `to` + `id`
18. Each platform needs its own provider (TRUE)
19. `check` blocks = warnings only; `precondition`/`postcondition` = errors
20. `moved` = same state only; cross-state = `removed` + `import`
