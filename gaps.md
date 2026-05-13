# Terraform Associate — Exam Gaps & Missing Topics

**This file covers ONLY the topics you got wrong across all 3 mock exams that were NOT adequately covered in your existing cheatsheets.**

---

## GAP 1: Version Constraints (Missed 4 times: Exam2-Q19, Exam3-Q56, Exam1-Q35, Exam2-Q28)

### The Rule You Keep Forgetting

The `~>` operator locks the **rightmost** specified digit. Everything to the left is FIXED.

```
~> X.Y.Z  →  only Z can change  →  >= X.Y.Z, < X.(Y+1).0
~> X.Y    →  only Y can change  →  >= X.Y.0, < (X+1).0.0
```

### Worked Examples (Memorize These)

| Constraint | Min | Max (exclusive) | Would install |
|---|---|---|---|
| `~> 3.2.0` | 3.2.0 | 3.3.0 | 3.2.0, 3.2.5 — NOT 3.3.0, NOT 4.0.0 |
| `~> 3.2` | 3.2.0 | 4.0.0 | 3.2.0, 3.2.5, 3.3.0, 3.9.9 — NOT 4.0.0 |
| `~> 6.0.0` | 6.0.0 | 6.1.0 | 6.0.2, 6.0.5 — NOT 6.1.0 |
| `~> 6.0` | 6.0.0 | 7.0.0 | 6.0.5, 6.1.0, 6.9.9 — NOT 7.0.0 |
| `~> 5.0` | 5.0.0 | 6.0.0 | 5.0.0, 5.3.0, 5.9.9 — NOT 6.0.0 |

### `terraform init` vs `terraform init -upgrade`
- `terraform init` — uses already-installed version (won't upgrade even if newer exists)
- `terraform init -upgrade` — upgrades to newest version matching constraints AND updates lock file

### What `-upgrade` Actually Does
Updates **plugins and modules** to newest version matching constraints. It does NOT:
- Upgrade Terraform CLI itself
- Upgrade the backend
- Upgrade configuration file syntax

---

## GAP 2: `for_each` Requires Map or Set (Missed 2 times: Exam3-Q17, Exam1-Q43)

### The Rule
`for_each` accepts ONLY:
- `map(any)`
- `set(string)`

It DOES NOT accept:
- `list(string)` — must convert with `toset()`
- `list(number)` — must convert
- `number` — that's `count`

### Common Pattern

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
  name     = each.value                   # each.key == each.value for sets
}

# ALSO RIGHT — use a map
resource "aws_instance" "app" {
  for_each = { for n in var.names : n => n }   # WORKS
}
```

### Referencing for_each Resources
```hcl
aws_instance.app["web"]     # by string key
aws_instance.app["api"]     # by string key
```

---

## GAP 3: Precondition vs Postcondition vs Validation vs Check (Missed 3 times: Exam3-Q3, Exam1-Q14, Exam2-Q13)

### The Four Validation Mechanisms

| Mechanism | Where | When it Runs | Use For |
|---|---|---|---|
| `validation {}` | Inside `variable` block | During input evaluation (before plan) | Validating variable values |
| `precondition {}` | Inside `lifecycle` block | Before resource/data create or update | Checking assumptions before action |
| `postcondition {}` | Inside `lifecycle` block | After resource/data create or update | Verifying results after action |
| `check {}` | Top-level block | After apply (as a health check) | Ongoing assertions across resources |

### When to Use Each

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

**Precondition** — check assumptions BEFORE acting:
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

**Postcondition** — verify results AFTER acting:
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

**Key exam distinction**:
- Data source that needs to verify it found something → **postcondition** (check AFTER query)
- Resource that needs to verify inputs before creation → **precondition** (check BEFORE create)
- Variable that must be within a range → **validation block** in variable

---

## GAP 4: HCP Terraform — cloud Block vs backend Block (Missed 2 times: Exam3-Q53, Exam2-Q39)

### The Rule

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
  backend "s3" { }      # For S3 backend
}
```

### cloud Block vs backend Block

| | `cloud {}` | `backend "s3" {}` |
|---|---|---|
| Used for | HCP Terraform / TFE | S3, GCS, azurerm, etc. |
| Execution | Remote by default | Always local |
| State | Stored in HCP Terraform | Stored in configured backend |
| Cannot use | Variables, locals, data sources inside | Variables, locals, data sources inside |
| Setup steps | `terraform login` → `terraform init` | `terraform init` |

### Backend Block Limitation (Exam Favorite)
Backend blocks ONLY accept **literal values**. You CANNOT use:
- `var.bucket_name` — NO
- `local.region` — NO
- `data.aws_caller_identity.current.account_id` — NO
- Only literal strings: `bucket = "my-state-bucket"` — YES

For dynamic values, use **partial backend configuration**:
```bash
terraform init -backend-config="bucket=my-bucket" -backend-config="key=prod.tfstate"
```

---

## GAP 5: HCP Terraform — tfe_outputs vs terraform_remote_state (Missed: Exam3-Q55)

### Cross-Workspace Data Sharing

| Method | When to Use | Syntax |
|---|---|---|
| `tfe_outputs` | HCP Terraform workspace → HCP Terraform workspace | `data "tfe_outputs" "x" { workspace = "other" }` |
| `terraform_remote_state` | Any backend → Any backend (S3, GCS, etc.) | `data "terraform_remote_state" "x" { backend = "s3" }` |

### tfe_outputs (HCP Terraform specific)
```hcl
data "tfe_outputs" "webserver" {
  organization = "my-org"
  workspace    = "prod-webserver"
}

# Access outputs
resource "aws_route53_record" "web" {
  records = [data.tfe_outputs.webserver.values.public_ip]
}
```

### terraform_remote_state (generic)
```hcl
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "my-state"
    key    = "vpc/terraform.tfstate"
    region = "us-east-1"
  }
}

# Access outputs
data.terraform_remote_state.vpc.outputs.vpc_id
```

---

## GAP 6: HCP Terraform — Run Tasks vs Run Triggers (Missed 3 times: Exam1-Q5, Exam2-Q53, Exam1-Q39)

| Feature | What It Does | Example |
|---|---|---|
| **Run Triggers** | Auto-queue run in workspace B after workspace A applies | network workspace → app workspace |
| **Run Tasks** | Integrate external tools between plan and apply | Security scanner, cost estimator |
| **Variable Sets** | Share variables across workspaces | Common AWS creds for a project |

### Run Tasks Detail
- Execute between plan and apply phases
- Call external HTTP endpoints (webhooks)
- Can be **advisory** (warning only) or **mandatory** (blocks apply)
- Use case: Snyk security scan, Infracost cost check, custom compliance

### HCP Terraform Agents
- Run on YOUR infrastructure (behind your firewall)
- For accessing private/on-premises resources
- NOT for state management, NOT for workspace management
- Enterprise/Plus feature

---

## GAP 7: Sensitive Data — NOTHING Prevents It From Being in State (Missed 2 times: Exam3-Q46, Exam2-Q47)

### The Absolute Rule
**No method prevents sensitive values from being stored in the Terraform state file.** Period.

| Method | In CLI output? | In state file? |
|---|---|---|
| `sensitive = true` on variable | Hidden | **YES — plaintext** |
| `sensitive = true` on output | Hidden | **YES — plaintext** |
| Vault data source | Hidden (if sensitive) | **YES — plaintext** |
| Environment variable | Not in config | **YES — still in state** |
| tfvars file | Depends | **YES — still in state** |

### The ONLY Exception (New in Terraform 1.10+)
**Ephemeral resources** and **write-only attributes** keep secrets out of state:

```hcl
ephemeral "random_password" "db" {
  length = 20
}

resource "aws_db_instance" "main" {
  password_wo         = ephemeral.random_password.db.result
  password_wo_version = 1
}
```

### How to Protect State
Since sensitive data IS in state, protect the state itself:
- Remote backend with encryption at rest (S3 + SSE)
- Access controls (IAM, RBAC)
- State locking (DynamoDB)
- Never commit to git
- HCP Terraform encrypts state at rest and in transit

---

## GAP 8: moved Block (Missed 3 times: Exam2-Q26, Exam3-Q51, Exam1-Q23)

### When to Use moved
You renamed or restructured a resource and don't want Terraform to destroy/recreate it.

```hcl
# You renamed aws_s3_bucket.logs → aws_s3_bucket.app_logs
moved {
  from = aws_s3_bucket.logs
  to   = aws_s3_bucket.app_logs
}

# You moved a resource into a module
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}
```

### moved vs Other Approaches

| Scenario | Correct Approach |
|---|---|
| Renamed a resource | `moved` block |
| Moved resource into a module | `moved` block |
| Want to import existing resource | `import` block |
| Want to remove from state without destroying | `removed` block or `terraform state rm` |
| Want to destroy a specific resource | Delete the resource block + `terraform apply` |

### How Long to Keep moved Blocks
- Keep for at least **one full apply cycle** across ALL team members
- Removing too soon = Terraform thinks old was destroyed, new needs creation
- Safe to remove after everyone has applied with the block present

---

## GAP 9: Plan Output Symbols (Missed: Exam2-Q50)

| Symbol | Meaning | Description |
|---|---|---|
| `+` | Create | New resource will be created |
| `-` | Destroy | Resource will be destroyed |
| `~` | **Update in-place** | Resource modified without recreation |
| `-/+` | Replace (destroy then create) | Must be destroyed and recreated |
| `+/-` | Replace (create then destroy) | With `create_before_destroy` |
| `<=` | Read | Data source will be read |

### Quick Memory Aid
- `+` = plus = adding something new
- `-` = minus = removing something
- `~` = tilde = "similar" = updating to similar version (in-place)
- `-/+` = destroy first, then create (default replacement)

---

## GAP 10: Commands That Modify State vs Read-Only (Missed: Exam3-Q36)

### Commands That MODIFY State
| Command | What it writes |
|---|---|
| `terraform apply` | Creates/updates/deletes resources, writes new state |
| `terraform destroy` | Removes resources, writes new state |
| `terraform import` | Adds resource to state |
| `terraform state mv` | Moves resource address in state |
| `terraform state rm` | Removes resource from state |
| `terraform apply -refresh-only` | Updates state to match real infra |

### Commands That DO NOT Modify State
| Command | What it does |
|---|---|
| `terraform plan` | Read-only comparison (refreshes internally but doesn't save) |
| `terraform validate` | Syntax check only |
| `terraform fmt` | Formats .tf files |
| `terraform show` | Displays state or plan |
| `terraform graph` | Generates dependency graph |
| `terraform output` | Reads outputs from state |
| `terraform state list` | Lists resources in state |
| `terraform state show` | Shows resource details |
| `terraform console` | Interactive expression eval |
| `terraform providers` | Lists providers |

---

## GAP 11: terraform show vs terraform state list vs terraform state show (Missed 3 times)

| Command | What it shows | Use when |
|---|---|---|
| `terraform show` | ENTIRE state file (verbose, all attributes) | Full overview |
| `terraform state list` | Resource addresses ONLY (concise) | Quick count / inventory |
| `terraform state show <addr>` | Single resource (detailed) | Investigate one resource |

### Example Output

```bash
# terraform state list
aws_instance.web
aws_s3_bucket.logs
module.vpc.aws_vpc.main

# terraform state show aws_instance.web
resource "aws_instance" "web" {
    ami           = "ami-12345"
    instance_type = "t3.micro"
    public_ip     = "54.1.2.3"
    ...50 more attributes...
}

# terraform show
# Shows EVERYTHING — all resources with all attributes
# Can be hundreds of lines
```

---

## GAP 12: Module Variable Scope (Missed 2 times: Exam3-Q9, Exam2-Q39)

### The Absolute Rule
**Child modules NEVER inherit variables from parent. NEVER.**

```
Root Module                     Child Module
variable "env" = "prod"        variable "env" — MUST BE DECLARED HERE
                                               OR ERROR
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
| Child → Root | `terraform.workspace` | Available everywhere (but discouraged in modules) |
| Root ↛ Child | Locals | NEVER inherited |
| Root ↛ Child | Data sources | NEVER inherited |
| Child ↛ Child | Nothing | Must go through root |

---

## GAP 13: Not All Backends Support Locking (Missed: Exam3-Q57)

### Backend Locking Support

| Backend | Supports Locking? | How? |
|---|---|---|
| **S3** | Only with DynamoDB | Need `dynamodb_table` config |
| **S3** (alone, no DynamoDB) | **NO** | State corruption risk |
| **azurerm** | Yes | Built-in (blob lease) |
| **gcs** | Yes | Built-in |
| **consul** | Yes | Built-in |
| **HCP Terraform** | Yes | Built-in |
| **pg** (PostgreSQL) | Yes | Built-in |
| **local** | **NO** | No concurrent protection |
| **http** | **Depends** | Only if server implements it |

### Key Exam Point
"All remote backends support locking" = **FALSE**
S3 without DynamoDB does NOT lock. http depends on implementation.

---

## GAP 14: IaC vs Configuration Management (Missed: Exam3-Q8)

### The Distinction

| | IaC (Terraform) | Config Management (Chef/Puppet/Ansible) |
|---|---|---|
| **Focus** | Provisioning infrastructure (VMs, networks, storage) | Configuring software on existing servers |
| **Approach** | Declarative (desired state) | Often imperative (Chef) or mixed |
| **Lifecycle** | Create, update, destroy infra | Install, configure, patch software |
| **Example** | Create an EC2 instance | Install Nginx on that instance |
| **Agents** | Agentless | Often agent-based |

### They're Complementary
Terraform provisions the VM → Chef/Ansible configures the software on it.
They solve different problems and are often used together.

---

## GAP 15: Ephemeral Resources & Write-Only Attributes (New Topic from Exam3-Q50)

### Ephemeral Resources (Terraform 1.10+)
Resources that exist only during a Terraform operation and are NOT stored in state:

```hcl
ephemeral "random_password" "db" {
  length = 20
}
```

### Write-Only Attributes
Resource attributes that are written to the provider but NOT stored in state:

```hcl
resource "aws_db_instance" "main" {
  password_wo         = ephemeral.random_password.db.result
  password_wo_version = 1    # increment to trigger password change
}
```

### Key Point
This is the ONLY way to keep a value completely out of state. Regular `sensitive = true` still writes to state.

---

## GAP 16: HCP Terraform — Workspace Mapping (Missed: Exam3-Q4)

- 1 workspace = 1 VCS repo (max)
- Multiple workspaces CAN point to the same repo
- 1 workspace = 1 state file
- 1 workspace = its own variables, team access, run history

### Variable Sets Scope Levels
1. **Global** — all current and future workspaces in the org
2. **Project-specific** — all workspaces in selected projects
3. **Workspace-specific** — only selected workspaces

---

## GAP 17: terraform plan — What It Actually Does (Missed: Exam3-Q6)

### The Complete Answer
`terraform plan`:
1. Refreshes state by querying provider APIs (unless `-refresh=false`)
2. Compares refreshed state to desired config
3. **Creates an execution plan** showing what changes are needed
4. Does NOT modify state or infrastructure

The key exam answer is: **"creates an execution plan and determines what changes are required"**

NOT just "reconciles state" (that's only step 1, incomplete answer).

---

## GAP 18: Data Source Reference Syntax (Missed pattern across exams)

### Correct Reference Patterns

| Block Type | Declaration | Reference |
|---|---|---|
| Resource | `resource "aws_vpc" "main"` | `aws_vpc.main.id` |
| Data source | `data "aws_vpc" "main"` | `data.aws_vpc.main.id` |
| Module output | `module "vpc"` | `module.vpc.vpc_id` |
| Variable | `variable "name"` | `var.name` |
| Local | `locals { x = 1 }` | `local.x` |
| Terraform | — | `terraform.workspace` |

### Map Key with Hyphen (Exam3-Q42)
```hcl
var.metadata["build-tag"]     # CORRECT — bracket notation for hyphens
var.metadata.build-tag        # WRONG — HCL interprets hyphen as minus
```

---

## Quick Summary: Your 18 Gaps Ranked by Frequency

| # | Gap | Times Missed | Priority |
|---|---|---|---|
| 1 | Version constraints (`~>`) | 4 | CRITICAL |
| 2 | moved block usage | 3 | HIGH |
| 3 | HCP run tasks vs run triggers | 3 | HIGH |
| 4 | show vs state list vs state show | 3 | HIGH |
| 5 | Precondition vs postcondition | 3 | HIGH |
| 6 | for_each needs map/set | 2 | HIGH |
| 7 | Sensitive data always in state | 2 | HIGH |
| 8 | Module variable scope | 2 | HIGH |
| 9 | cloud block vs backend | 2 | HIGH |
| 10 | tfe_outputs for cross-workspace | 1 | MEDIUM |
| 11 | Plan output symbols (~, +, -) | 1 | MEDIUM |
| 12 | Commands that modify state | 1 | MEDIUM |
| 13 | Backend locking support | 1 | MEDIUM |
| 14 | IaC vs config management | 1 | LOW |
| 15 | Ephemeral resources | 1 | LOW |
| 16 | HCP workspace mapping | 1 | LOW |
| 17 | What terraform plan does | 1 | LOW |
| 18 | Data source reference syntax | 1 | LOW |

---

**Focus on the CRITICAL and HIGH items first. Fixing those 9 gaps alone would have given you 70%+ on all three exams.**
