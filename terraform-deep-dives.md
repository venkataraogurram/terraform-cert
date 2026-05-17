# Terraform Associate — Deep Dives

In-depth explanations of the topics most commonly tested and most often misunderstood.

---

## Deep Dive 1: `count` vs `for_each` — Identity, Stability, and Indexing

This is one of the most consequential design decisions in Terraform. Both produce multiple resource instances, but they have very different semantics.

### `count` — index-based identity

```hcl
variable "users" {
  default = ["alice", "bob", "carol"]
}

resource "aws_iam_user" "this" {
  count = length(var.users)
  name  = var.users[count.index]
}
```

Resources are addressed by **integer index**:
- `aws_iam_user.this[0]` → alice
- `aws_iam_user.this[1]` → bob
- `aws_iam_user.this[2]` → carol

### The problem with `count`: index churn

If you remove `"bob"` from the list:

```hcl
default = ["alice", "carol"]
```

Terraform sees:
- `[0]` was `alice`, still `alice` ✅
- `[1]` was `bob`, **now `carol`** → must update `name` from `bob` to `carol`
- `[2]` was `carol`, **now removed** → destroy

Result: `bob` is destroyed AND `carol`'s identity moves from index 2 to index 1, causing an in-place update or replacement depending on the resource.

This is **index churn** — a classic Terraform footgun.

### `for_each` — keyed identity

```hcl
resource "aws_iam_user" "this" {
  for_each = toset(["alice", "bob", "carol"])
  name     = each.key
}
```

Resources are addressed by **string key**:
- `aws_iam_user.this["alice"]`
- `aws_iam_user.this["bob"]`
- `aws_iam_user.this["carol"]`

Removing `"bob"`:
- `["alice"]` unchanged ✅
- `["bob"]` destroyed
- `["carol"]` unchanged ✅

Only `bob` is affected. Stable identity.

### When to use which

| Use case | Best |
|---|---|
| N identical copies of a thing | `count` |
| Distinct named objects | `for_each` |
| Iterating over a list of strings/objects | `for_each` (with `toset` or map) |
| Conditional inclusion (0 or 1) | `count = var.enabled ? 1 : 0` |
| Avoiding identity churn | `for_each` |

### `for_each` over a map

```hcl
variable "users" {
  type = map(object({
    role = string
  }))
  default = {
    alice = { role = "admin" }
    bob   = { role = "dev" }
  }
}

resource "aws_iam_user" "this" {
  for_each = var.users
  name     = each.key
  tags = {
    Role = each.value.role
  }
}
```

### Mixing them with modules

Modules also support `count` and `for_each`:

```hcl
module "vpc" {
  for_each = var.vpcs
  source   = "./modules/vpc"
  cidr     = each.value.cidr
}
```

### Conversion patterns

List → for_each-compatible map:
```hcl
for_each = { for u in var.users : u.name => u }
```

List → set (when only the value matters):
```hcl
for_each = toset(var.user_names)
```

### Exam takeaway

- `count` for simple replication. `for_each` for distinct named instances.
- `for_each` requires a **set of strings** or a **map**, not a list.
- Removing a list item with `count` causes cascading changes.

---

## Deep Dive 2: Modules — Composition Patterns

### The four module types

| Type | Source | Purpose |
|---|---|---|
| Root module | The directory you run Terraform from | Entry point |
| Child module | Called by another module | Reusable logic |
| Published module | Registry / Git / etc. | Distributed |
| Local module | `./path` | In-repo organization |

### Module structure

```
modules/vpc/
├── main.tf            # primary resources
├── variables.tf       # input variables
├── outputs.tf         # exposed outputs
├── versions.tf        # required_providers, required_version
├── README.md          # documentation
└── examples/          # usage examples
```

### Calling a module

```hcl
module "network" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "main-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
}
```

### Module outputs

In module:
```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  value = aws_vpc.this.id
}
```

In root:
```hcl
resource "aws_subnet" "extra" {
  vpc_id = module.network.vpc_id
}
```

### Passing providers to modules

When a module needs a non-default provider config:

```hcl
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

module "west_vpc" {
  source = "./modules/vpc"
  providers = {
    aws = aws.west
  }
}
```

In the module, declare it:
```hcl
# modules/vpc/versions.tf
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      configuration_aliases = [aws.west]   # optional
    }
  }
}
```

### Module versioning

Always pin module versions in production:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}
```

Git sources support `?ref=` for tags/commits:
```hcl
source = "git::https://github.com/x/y.git//modules/vpc?ref=v1.2.3"
```

### Module composition patterns

**Wrapper module** — adds opinionated defaults:
```hcl
# modules/standard-vpc/main.tf
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  cidr   = var.cidr
  tags   = merge(var.tags, { ManagedBy = "terraform" })
}
```

**Composition** — combine multiple modules:
```hcl
module "network" {
  source = "./modules/vpc"
  # ...
}

module "compute" {
  source     = "./modules/ec2"
  vpc_id     = module.network.vpc_id
  subnet_ids = module.network.private_subnet_ids
}
```

### Anti-patterns to avoid

- **Module with hardcoded values** — should accept variables.
- **Module with side effects on apply** (provisioners, local-execs) — usually a smell.
- **Excessive nesting** — keep module hierarchies shallow.
- **Coupling modules** — modules should be independently testable.

### Exam takeaway

- Modules expose inputs (`variables`) and outputs (`output`).
- Use `source` and pin `version`.
- Reference outputs with `module.NAME.OUTPUT`.
- Pass alternate providers via `providers = { ... }`.

---

## Deep Dive 3: State — Backends, Locking, and Remote State

### Why state exists

State maps the resources in your config (e.g., `aws_instance.web`) to real-world resource IDs (e.g., `i-0abc123`). Without state, Terraform couldn't:
- Know what already exists
- Generate plans
- Detect drift
- Track dependencies

### Local vs remote backends

**Local (default)** — state on disk:
- Good for: solo dev, small projects.
- Bad: no team collaboration, no locking, easy to lose.

**Remote** — state in object storage or service:
- Good: team sharing, locking, encryption, versioning.
- Required for any real production setup.

### S3 backend — the canonical AWS pattern

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "tf-state-lock"
    
    # Optional
    role_arn       = "arn:aws:iam::123:role/StateAccess"
    workspace_key_prefix = "env"
  }
}
```

The S3 bucket should have:
- **Versioning enabled** — recover any state version.
- **Server-side encryption** — `aws:kms` or `AES256`.
- **Block public access**.
- **Restricted IAM policies** — only CI and platform team.

The DynamoDB table for locking:
```hcl
resource "aws_dynamodb_table" "tf_lock" {
  name         = "tf-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

### State locking — what happens

When you run `terraform apply`:
1. Terraform writes a lock entry to DynamoDB.
2. Other concurrent runs see the lock and refuse to proceed.
3. After apply succeeds (or fails cleanly), the lock is released.

If a process crashes and leaves a stale lock:
```bash
terraform force-unlock <LOCK_ID>
```

⚠️ Use `force-unlock` carefully — only when you're sure no one else is running.

### State manipulation commands

```bash
terraform state list                          # list addresses
terraform state show aws_instance.web         # inspect one resource
terraform state mv aws_instance.web aws_instance.api    # rename
terraform state rm aws_instance.web           # remove from state (no destroy)
terraform state pull > backup.tfstate         # download
terraform state push backup.tfstate           # upload (dangerous)
terraform state replace-provider OLD NEW      # change provider source
```

### Cross-stack data — `terraform_remote_state`

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-tf-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "web" {
  subnet_id = data.terraform_remote_state.network.outputs.private_subnet_id
}
```

⚠️ This couples the consumer to the producer's state file format and outputs. Alternatives:
- **SSM Parameter Store** / **AWS Secrets Manager** for shared values.
- **Data sources** that query the cloud directly.

### Migrating between backends

```bash
# Change backend block in code, then:
terraform init -migrate-state    # copy state to new backend
# OR
terraform init -reconfigure      # ignore old, start fresh
```

### State file rotation/recovery

| Scenario | Action |
|---|---|
| Lost local state, no remote | Hope for `terraform.tfstate.backup` |
| Crash mid-apply | Look for `errored.tfstate`, push it |
| Need an older version | S3 versioning console, or `aws s3api get-object --version-id` |
| Resource deleted from state by mistake | `terraform import` to recreate state entry |
| State corrupt | Restore from S3 versioning |

### Exam takeaway

- Remote backend with locking is **the** production pattern.
- S3 + DynamoDB is the canonical AWS combo.
- `terraform.tfstate.backup` is a single-step undo — not a real safety net.
- Never commit state to Git.

---

## Deep Dive 4: Variable Precedence — All Edge Cases

### The full ladder (low → high)

```
1. variable defaults (in variable block)
2. TF_VAR_<name> environment variables
3. terraform.tfvars
4. terraform.tfvars.json
5. *.auto.tfvars (alphabetical)
6. *.auto.tfvars.json (alphabetical)
7. -var-file=<file> CLI flags (in command-line order)
8. -var "name=value" CLI flags (in command-line order)
```

### Edge cases worth knowing

#### Within the same precedence level
- `*.auto.tfvars`: alphabetical, **later overrides earlier**.
- `-var-file` / `-var`: **last-specified wins**.

#### What's NOT auto-loaded
- `prod.tfvars` ❌ (no `.auto`)
- `dev.tfvars` ❌
- `variables.tfvars` ❌

These all need `-var-file=<file>`.

#### Mixing HCL and JSON at same level
```
terraform.tfvars       # loads first
terraform.tfvars.json  # loads second (would override .tfvars)

dev.auto.tfvars         # loads first
dev.auto.tfvars.json    # loads second (would override .auto.tfvars)
```

#### Setting complex types via env
```bash
export TF_VAR_ports='[80, 443]'
export TF_VAR_tags='{Env="prod", Team="ops"}'
```

#### Setting complex types via CLI
```bash
terraform apply \
  -var='ports=[80, 443]' \
  -var='tags={Env="prod", Team="ops"}'
```

#### What if no value is provided?
- If a `default` exists → uses default.
- Otherwise → interactive prompt (unless `-input=false`).
- With `-input=false` and no value → **error**.

### Practical multi-environment setup

**File layout:**
```
.
├── main.tf
├── variables.tf
├── terraform.tfvars              # shared defaults (auto-loaded)
└── environments/
    ├── dev.tfvars
    ├── staging.tfvars
    └── prod.tfvars
```

**Apply per environment:**
```bash
terraform apply -var-file=environments/prod.tfvars
```

This is cleaner than CLI workspaces for environment separation.

### Variable validation

```hcl
variable "instance_type" {
  type = string
  validation {
    condition     = contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
    error_message = "instance_type must be t3.micro, t3.small, or t3.medium."
  }
  validation {  # multiple validations supported (1.9+)
    condition     = startswith(var.instance_type, "t3")
    error_message = "Only t3 family allowed."
  }
}
```

### Sensitive variables

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
```

- Hidden from CLI plan/apply output.
- **Still stored in plaintext in state**.
- Use `nonsensitive(value)` to unmark when needed (rare).

### Nullable variables

```hcl
variable "extra_tag" {
  type     = string
  default  = null
  nullable = true   # default since 1.1+
}
```

Setting `nullable = false` rejects null assignments.

### Exam takeaway

- **CLI flags > files > env vars > defaults**.
- `terraform.tfvars` and `*.auto.tfvars` are auto-loaded.
- Other `.tfvars` files need `-var-file=`.
- Sensitive only hides output, doesn't encrypt state.

---

## Deep Dive 5: Drift, Refresh, and Reconciliation

### Three sources of "truth"

```
┌──────────┐      ┌────────┐      ┌──────────────┐
│  Config  │      │ State  │      │   Reality    │
│  (.tf)   │ ←→  │ (.tfs) │  ←→  │  (cloud API) │
└──────────┘      └────────┘      └──────────────┘
```

### What `plan` does (default)

1. **Refresh:** query reality, update state in-memory.
2. **Compare:** state vs config.
3. **Compute plan:** changes needed to make reality match config.

### What `plan -refresh-only` does

1. Refresh: query reality, update state in-memory.
2. Compare: refreshed state vs old state.
3. Show: drift detected, no plan against config.

### What `plan -refresh=false` does

1. **Skip refresh.**
2. Compare: state (as-is) vs config.
3. Compute plan based on possibly stale state.

### The drift acceptance workflow

Someone added a manual tag in the AWS console. You want to keep it.

```bash
# 1. See the drift
terraform plan -refresh-only

# 2. Update state to reflect reality
terraform apply -refresh-only

# 3. Update your config to match (the right way)
# OR add lifecycle ignore_changes (the alternative)

# 4. Verify no diff
terraform plan
```

### Drift detection in CI

```bash
terraform plan -refresh-only -detailed-exitcode
```

| Exit code | Meaning |
|---|---|
| 0 | No drift |
| 1 | Error |
| 2 | Drift detected |

Use this to alert on out-of-band changes.

### When to use `ignore_changes`

When you intentionally allow drift on certain attributes:

```hcl
resource "aws_instance" "web" {
  # ...
  
  lifecycle {
    ignore_changes = [
      tags,                   # ignore all tags
      tags["LastModified"],   # ignore one specific tag (1.4+)
      ami,                    # ignore AMI changes (e.g., from auto-rotation)
    ]
  }
}
```

Use sparingly — it's an escape hatch that hides intent.

### Reconciliation toolkit

| Goal | Tool |
|---|---|
| Accept drift into state | `apply -refresh-only` |
| Revert drift | normal `apply` |
| Bring existing resource into state | `terraform import` (or `import` block in 1.5+) |
| Remove from state without destroying | `state rm` |
| Tell Terraform to ignore an attribute | `lifecycle.ignore_changes` |
| Force replacement | `apply -replace=ADDR` |

### `import` block (Terraform 1.5+)

Declarative imports — easier to review and version-control:

```hcl
import {
  to = aws_instance.web
  id = "i-0abc1234"
}

resource "aws_instance" "web" {
  # config that matches the imported resource
}
```

Run:
```bash
terraform plan -generate-config-out=generated.tf   # auto-generate
terraform apply
```

### Exam takeaway

- `plan` refreshes by default; `apply` does too.
- `apply -refresh-only` updates state to reality (no infra change).
- `terraform refresh` (standalone) is **deprecated**.
- `-detailed-exitcode` gives 2 when changes are detected — useful for CI.

---

## Deep Dive 6: Provisioners and Their Alternatives

### Why HashiCorp says "last resort"

Provisioners run only on **create or destroy** — not on updates. If config drifts or the script needs to re-run, Terraform won't notice unless the resource is replaced. They make Terraform less declarative and more imperative.

### When provisioners *do* make sense

- One-time bootstrap that can't be done via cloud-init.
- Trigger external systems (Slack notification, DNS update).
- Fetch a value that only exists post-creation.

### `local-exec`

```hcl
resource "aws_instance" "web" {
  # ...
  provisioner "local-exec" {
    command = "echo 'Created ${self.id}' >> creation.log"
    
    environment = {
      INSTANCE_ID = self.id
    }
    
    interpreter = ["/bin/bash", "-c"]
  }
}
```

### `remote-exec` (requires `connection`)

```hcl
resource "aws_instance" "web" {
  # ...
  
  connection {
    type        = "ssh"
    user        = "ec2-user"
    host        = self.public_ip
    private_key = file("~/.ssh/key.pem")
  }
  
  provisioner "remote-exec" {
    inline = [
      "sudo yum update -y",
      "sudo yum install -y nginx",
      "sudo systemctl start nginx",
    ]
  }
}
```

### `file`

```hcl
provisioner "file" {
  source      = "./scripts/setup.sh"
  destination = "/tmp/setup.sh"
}
```

### Destroy-time provisioners

```hcl
provisioner "local-exec" {
  when    = destroy
  command = "echo 'Destroying ${self.id}'"
}
```

**Strict rules:**
- Can only reference `self`, `count.index`, `each.key`.
- No `var.`, `local.`, or other resources.
- Designed to be self-contained because the resource is going away.

### `on_failure`

```hcl
provisioner "local-exec" {
  command    = "./might-fail.sh"
  on_failure = continue   # default: fail
}
```

`fail` (default) marks the resource **tainted** — destroyed and recreated next apply.
`continue` logs the failure but proceeds.

### `null_resource` and `terraform_data`

When you need a provisioner that isn't tied to a real resource:

**Old way** — `null_resource`:
```hcl
resource "null_resource" "trigger" {
  triggers = {
    timestamp = timestamp()
  }
  
  provisioner "local-exec" {
    command = "./run.sh"
  }
}
```

**New way (1.4+)** — `terraform_data`:
```hcl
resource "terraform_data" "trigger" {
  triggers_replace = [var.config_version]
  
  provisioner "local-exec" {
    command = "./run.sh"
  }
}
```

`terraform_data` is built into Terraform — no `null` provider needed.

### Better alternatives to provisioners

| Goal | Better than provisioner |
|---|---|
| EC2 bootstrap | `user_data` / cloud-init |
| Software install | Pre-baked AMI (Packer) |
| Configuration management | Ansible/Chef/Puppet/SSM |
| Secret retrieval | Data source from Vault/SSM |
| External trigger on resource change | Lambda + EventBridge (event-driven, not Terraform-driven) |
| Wait for resource readiness | `depends_on` + AWS waiters in cloud-init |

### Exam takeaway

- Provisioners: **last resort**.
- `local-exec` runs on the Terraform host; `remote-exec` runs on the resource.
- `self`, `count.index`, `each.key` only in destroy-time provisioners.
- `null_resource` → `terraform_data` (1.4+).
- Failed `fail` provisioner taints the resource.

---

## Deep Dive 7: Lifecycle Meta-arguments

```hcl
resource "aws_instance" "web" {
  # ...
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
    ignore_changes        = [tags]
    replace_triggered_by  = [aws_security_group.web]
    
    precondition {
      condition     = data.aws_ami.latest.architecture == "x86_64"
      error_message = "AMI must be x86_64."
    }
    
    postcondition {
      condition     = self.private_ip != ""
      error_message = "Instance must have a private IP."
    }
  }
}
```

### `create_before_destroy`

Default behavior: destroy old, then create new. Causes downtime.

With `create_before_destroy = true`:
1. Create new resource (alongside old).
2. Cut over (state updated).
3. Destroy old.

Caveats:
- Both resources exist briefly — names/CIDRs/etc. **must not collide**.
- Often paired with `name_prefix` instead of `name`, or random suffixes.

```hcl
resource "aws_security_group" "web" {
  name_prefix = "web-"   # avoids name collision
  # ...
  lifecycle {
    create_before_destroy = true
  }
}
```

### `prevent_destroy`

Hard stop on `terraform destroy` and any plan that would destroy this resource.

```hcl
resource "aws_db_instance" "prod" {
  # ...
  lifecycle {
    prevent_destroy = true
  }
}
```

To actually destroy, you must remove this argument first. Common for stateful production resources.

### `ignore_changes`

Tells Terraform to ignore drift on specific attributes.

```hcl
lifecycle {
  ignore_changes = [
    tags,                       # ignore the whole map
    tags["LastModified"],       # ignore one tag (1.4+)
    ami,
  ]
}

# Or ignore everything (risky):
ignore_changes = all
```

Common uses:
- Auto-scaling group `desired_capacity` (managed by autoscaling, not Terraform).
- Tags applied by AWS Backup or Config.
- AMI updates from automation.

### `replace_triggered_by` (1.2+)

Force replacement when another resource changes.

```hcl
resource "aws_instance" "web" {
  # ...
  lifecycle {
    replace_triggered_by = [aws_security_group.web]
  }
}
```

References:
- Resource address → triggers on replacement of that resource.
- Attribute → triggers on attribute change.

### Preconditions and postconditions (1.2+)

Enforce invariants at plan/apply time.

```hcl
data "aws_ami" "latest" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*"]
  }
  
  lifecycle {
    postcondition {
      condition     = self.architecture == "x86_64"
      error_message = "AMI must be x86_64."
    }
  }
}

resource "aws_instance" "web" {
  ami = data.aws_ami.latest.id
  
  lifecycle {
    precondition {
      condition     = data.aws_ami.latest.id != ""
      error_message = "AMI not found."
    }
  }
}
```

These run at **plan time** and stop the plan if violated.

### Exam takeaway

- `create_before_destroy` for zero-downtime replacement.
- `prevent_destroy` for hard guardrails.
- `ignore_changes` to coexist with external systems.
- `replace_triggered_by` (1.2+) for forced cascading replacement.
- `precondition`/`postcondition` (1.2+) for assertions.

---

## Deep Dive 8: Functions Worth Memorizing

### `flatten` — collapse nested lists

```hcl
flatten([["a"], [], ["b", "c"]])    # ["a", "b", "c"]
```

Best paired with nested `for` for `for_each` consumption.

### `merge` — combine maps (right wins)

```hcl
merge({a=1, b=2}, {b=3, c=4})      # {a=1, b=3, c=4}
```

Common pattern — merging tags:
```hcl
tags = merge(
  var.default_tags,
  { Environment = "prod" },
  var.extra_tags,
)
```

### `lookup` — safe map access

```hcl
lookup(var.config, "region", "us-east-1")
```

Returns the value if key exists, else the default.

### `try` — catch errors

```hcl
try(local.config.deeply.nested.value, "fallback")
```

If the expression errors (e.g., key missing), returns fallback.

### `can` — test if expression succeeds

```hcl
can(regex("^arn:", var.input))   # true if matches, false otherwise
```

Often used in `validation` blocks.

### `coalesce` and `coalescelist`

`coalesce` returns first non-null/non-empty:
```hcl
coalesce(var.name, var.default_name, "fallback")
```

`coalescelist` returns first non-empty list:
```hcl
coalescelist(var.subnets, ["10.0.1.0/24"])
```

### `cidrsubnet`, `cidrhost`

Subnet calculations:
```hcl
cidrsubnet("10.0.0.0/16", 8, 0)   # "10.0.0.0/24"
cidrsubnet("10.0.0.0/16", 8, 1)   # "10.0.1.0/24"
cidrsubnet("10.0.0.0/16", 8, 2)   # "10.0.2.0/24"

cidrhost("10.0.1.0/24", 5)        # "10.0.1.5"
```

Generate subnets dynamically:
```hcl
resource "aws_subnet" "private" {
  count      = length(var.azs)
  vpc_id     = aws_vpc.main.id
  cidr_block = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  availability_zone = var.azs[count.index]
}
```

### `templatefile` — render template files

```
# templates/init.sh.tftpl
#!/bin/bash
echo "${greeting}, instance ${instance_id}"
```

```hcl
user_data = templatefile("${path.module}/templates/init.sh.tftpl", {
  greeting    = "Hello"
  instance_id = "web-01"
})
```

### `jsonencode` / `jsondecode`

```hcl
policy = jsonencode({
  Version = "2012-10-17"
  Statement = [{
    Effect   = "Allow"
    Action   = "s3:GetObject"
    Resource = "arn:aws:s3:::my-bucket/*"
  }]
})
```

Better than embedding JSON strings — Terraform validates the structure.

### `for` expressions

```hcl
# Transform list
[for s in var.list : upper(s)]

# Filter list
[for s in var.list : s if s != ""]

# Build map from list
{for u in var.users : u.name => u.email}

# Iterate map
{for k, v in var.config : k => upper(v)}

# Nested
[
  for vpc, cidrs in var.vpcs : [
    for cidr in cidrs : { vpc = vpc, cidr = cidr }
  ]
]
```

### Exam takeaway

- `flatten` + nested `for` is the canonical many-to-many pattern.
- `try`/`can` for safe expressions.
- `jsonencode` for IAM policies and similar structured strings.
- `cidrsubnet` for dynamic subnet CIDRs.
- `templatefile` for user_data scripts.

---

## Deep Dive 9: Workspaces (CLI vs Terraform Cloud)

### CLI workspaces (open source)

Each workspace has its own state file in the same backend:

```
s3://my-state/env:/dev/terraform.tfstate
s3://my-state/env:/prod/terraform.tfstate
s3://my-state/terraform.tfstate         # default workspace
```

```bash
terraform workspace new dev
terraform workspace new prod
terraform workspace select prod
terraform workspace show
terraform workspace list
```

In config, use `terraform.workspace`:

```hcl
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"
  tags = {
    Environment = terraform.workspace
  }
}
```

### Why HashiCorp discourages CLI workspaces for prod/dev

- **Same code path for all environments** — risky for prod.
- **Same backend config and creds** — limits separation.
- **No environment-level access control**.
- **Easy to apply to wrong workspace by mistake** (`terraform.workspace` in resource names is fragile).

### Recommended environment separation

```
.
├── modules/                      # shared modules
│   └── vpc/
└── environments/
    ├── dev/
    │   ├── main.tf               # uses module "vpc"
    │   ├── terraform.tfvars
    │   └── backend.tf            # dev state in dev backend
    ├── staging/
    └── prod/
```

Each environment has its own root module and backend. You apply per directory.

### Terraform Cloud workspaces (different concept)

In TFC/TFE, a "workspace" is a full environment:
- Has its own variables (set in UI).
- Has its own VCS integration.
- Has its own state.
- Has its own runs and history.

Closer to what you actually want for environment separation.

### When CLI workspaces work fine

- Quick experiments.
- Multiple isolated test environments with the same code.
- Per-developer sandboxes.

### Exam takeaway

- `terraform workspace` manages CLI workspaces.
- `default` is the default workspace.
- Use `terraform.workspace` to reference current name.
- HashiCorp discourages CLI workspaces for prod/dev separation.

---

## Deep Dive 10: Terraform Cloud, Sentinel, and Policy

### Terraform Cloud — what it provides

| Feature | Description |
|---|---|
| Remote state | Encrypted, versioned, shared |
| State locking | Built-in |
| Remote operations | Plan/apply runs in TFC infrastructure |
| VCS integration | Auto-trigger plans on PR |
| Variables UI | Set workspace and env vars |
| Run history | Audit trail of every plan/apply |
| Notifications | Slack, email, webhooks |
| Private module registry | Share modules within org |
| Sentinel / OPA | Policy enforcement |
| Cost estimation | Shows estimated AWS/Azure/GCP costs (paid) |
| Run triggers | One workspace's apply triggers another |
| SSO / Teams | Enterprise auth (paid) |

### Connecting

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

```bash
terraform login
terraform init
```

### Sentinel — policy as code

```python
# sentinel: enforce instance types
import "tfplan/v2" as tfplan

allowed_types = ["t3.micro", "t3.small", "t3.medium"]

main = rule {
    all tfplan.resource_changes as _, rc {
        rc.type is "aws_instance" implies
        rc.change.after.instance_type in allowed_types
    }
}
```

### Enforcement levels

| Level | Behavior |
|---|---|
| `advisory` | Pass/fail logged, doesn't block |
| `soft-mandatory` | Fails by default; admins can override |
| `hard-mandatory` | Always fails on violation; cannot be overridden |

### OPA (Open Policy Agent) — alternative

TFC also supports OPA policies (Rego language). Same enforcement-level concept.

### Run triggers

In a multi-workspace setup:
- Workspace A applies → triggers plan in Workspace B.
- Useful for layered infra (network → compute → app).

### Cost estimation

```
+ aws_instance.web
    instance_type: t3.large
    Estimated monthly: $60.74
```

### Exam takeaway

- Terraform Cloud provides remote state, runs, and policy.
- Sentinel: advisory / soft-mandatory / hard-mandatory.
- OPA is an alternative policy engine.
- TFC workspaces ≠ CLI workspaces.
- `terraform login` to authenticate.

---

## Final Summary

The 10 deepest topics:
1. **`count` vs `for_each`** — identity and stability
2. **Modules** — composition, providers, versioning
3. **State** — backends, locking, recovery
4. **Variable precedence** — full ladder
5. **Drift handling** — `apply -refresh-only`, `ignore_changes`, `import`
6. **Provisioners** — last resort, alternatives
7. **Lifecycle** — `create_before_destroy`, `replace_triggered_by`, conditions
8. **Functions** — `flatten`, `try`, `templatefile`, `cidrsubnet`
9. **Workspaces** — CLI vs TFC, anti-patterns
10. **Terraform Cloud** — remote ops, Sentinel, policies

Master these and the exam should feel comfortable.
