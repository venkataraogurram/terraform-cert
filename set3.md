# Terraform Associate — Practice Set 3 (50 Questions)

**Focus**: Hard scenarios, edge cases, tricky distractors
**Difficulty**: Hard

> These questions test deeper understanding. Read carefully — distractors are designed to look plausible.

---

## Section 1: Tricky Module Scenarios (Q1-10)

### Q1
You have this code:

```hcl
module "vpc" {
  source = "./modules/vpc"
  count  = var.create_vpc ? 1 : 0
}

resource "aws_subnet" "main" {
  vpc_id = module.vpc.vpc_id
}
```

What's the issue?

A. `count` is not allowed on modules
B. `module.vpc.vpc_id` should be `module.vpc[0].vpc_id` because count was used
C. Subnets must be inside the module
D. Nothing is wrong

<details>
<summary>Answer</summary>

**B. `module.vpc.vpc_id` should be `module.vpc[0].vpc_id` because count was used**

When `count` is on a module, you index into it. Same rules as count on a resource. `module.vpc[0].vpc_id` is correct.
</details>

---

### Q2
A child module has this output:

```hcl
output "instance_ids" {
  value = aws_instance.web[*].id
}
```

In the root, you write:

```hcl
output "first_id" {
  value = module.web.instance_ids[0]
}
```

What happens?

A. Returns the first instance ID
B. Errors — splat output cannot be indexed
C. Returns null
D. Returns a list

<details>
<summary>Answer</summary>

**A. Returns the first instance ID**

The child output is a list, so root can index into it normally. Outputs preserve their type.
</details>

---

### Q3
You add a `for_each` to a module:

```hcl
module "buckets" {
  source   = "./modules/bucket"
  for_each = toset(["logs", "backups", "data"])
  name     = each.key
}
```

How do you reference the "logs" bucket's ARN?

A. `module.buckets.logs.arn`
B. `module.buckets["logs"].arn`
C. `module.buckets[0].arn`
D. `module.logs.arn`

<details>
<summary>Answer</summary>

**B. `module.buckets["logs"].arn`**

`for_each` produces a map; you access by string key. `count` produces a list; you access by integer index.
</details>

---

### Q4
A module's `versions.tf`:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}
```

Your root module:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

What happens?

A. Both versions are downloaded
B. Terraform errors because version constraints conflict
C. The root version takes precedence
D. The module version takes precedence

<details>
<summary>Answer</summary>

**B. Terraform errors because version constraints conflict**

Constraint intersection: `~> 4.0` (= 4.x) AND `~> 5.0` (= 5.x) = empty set. Terraform requires ONE version that satisfies ALL constraints across all modules.
</details>

---

### Q5
What does this `moved` block do?

```hcl
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}
```

A. Moves the resource block in the file
B. Records that the resource was moved into a module — preserves state without recreation
C. Migrates state to a new backend
D. Renames the resource

<details>
<summary>Answer</summary>

**B. Records that the resource was moved into a module — preserves state without recreation**

`moved` blocks tell Terraform "the resource at address X is now at address Y." Without it, Terraform would destroy and recreate.
</details>

---

### Q6
You publish a module to a private registry. To use a beta version, what's the syntax?

A. `version = "1.0.0-beta"`
B. `version = "beta"`
C. `version = "1.0.0-beta.1"`
D. Both A and C work — pre-release versions follow semver

<details>
<summary>Answer</summary>

**D. Both A and C work — pre-release versions follow semver**

The Terraform Registry follows semver, so any valid semver string works. Note: pre-release versions are NOT included in `~>` constraints by default.
</details>

---

### Q7
A child module declares:

```hcl
variable "name" {
  type = string
}
```

The root module calls it without setting `name`:

```hcl
module "thing" {
  source = "./modules/thing"
}
```

What happens?

A. Terraform errors — required variable not set
B. The variable defaults to null
C. The variable defaults to ""
D. The module is skipped

<details>
<summary>Answer</summary>

**A. Terraform errors — required variable not set**

Variables without `default` are required. If a caller doesn't provide a value, Terraform errors at plan time.
</details>

---

### Q8
You're refactoring and want to remove a child module entirely without destroying the resources it created. What's the cleanest approach?

A. Delete the module block — Terraform destroys the resources
B. Use `removed` blocks to remove resources from state without destroying
C. Run `terraform state rm` for each resource
D. Both B and C work; `removed` block is declarative and safer

<details>
<summary>Answer</summary>

**D. Both B and C work; `removed` block is declarative and safer**

Terraform 1.7+ supports `removed` blocks:
```hcl
removed {
  from = module.old_thing.aws_instance.web
  lifecycle {
    destroy = false
  }
}
```
Declarative, version-controlled, and removes from state without destroying.
</details>

---

### Q9
A nested module structure:
```
root → module A → module B
```

Module B declares `output "id"`. How does root access this?

A. `module.A.module.B.id`
B. `module.B.id`
C. Module A must re-expose B's output, then root accesses `module.A.id`
D. Outputs from grandchildren are not accessible

<details>
<summary>Answer</summary>

**C. Module A must re-expose B's output, then root accesses `module.A.id`**

Outputs only propagate one level. To pass data up the tree, each level must explicitly re-export.
</details>

---

### Q10
True or False: A child module can use `terraform.workspace` to behave differently per workspace.

<details>
<summary>Answer</summary>

**True.**

`terraform.workspace` is available in any module. However, this is generally discouraged for modules that are meant to be reused — pass workspace-specific values as variables instead, so the module is environment-agnostic.
</details>

---

## Section 2: HCP Terraform Edge Cases (Q11-20)

### Q11
You move a workspace from VCS-driven to CLI-driven workflow. What changes?

A. State is migrated automatically
B. The VCS connection is removed; runs are now triggered from CLI/API
C. All previous runs are deleted
D. The workspace becomes read-only

<details>
<summary>Answer</summary>

**B. The VCS connection is removed; runs are now triggered from CLI/API**

State remains. Run history is preserved. The change just affects how new runs are triggered.
</details>

---

### Q12
In HCP Terraform, you set a Terraform variable as `sensitive = true` in the workspace UI. What can you do later?

A. View the value in the UI
B. Edit the value but not view it
C. View it via API only
D. Both view and edit it

<details>
<summary>Answer</summary>

**B. Edit the value but not view it**

Sensitive variables are write-only in HCP Terraform. You can update or delete them, but the existing value is never displayed.
</details>

---

### Q13
What's the relationship between an HCP Terraform Project and Workspace?

A. They are aliases
B. A project contains workspaces; permissions can be set at project level
C. A workspace contains projects
D. Projects are deprecated

<details>
<summary>Answer</summary>

**B. A project contains workspaces; permissions can be set at project level**

Project = grouping of related workspaces. Useful for assigning team permissions to a logical unit.
</details>

---

### Q14
True or False: HCP Terraform's run triggers can chain across organizations.

<details>
<summary>Answer</summary>

**False.**

Run triggers are scoped to within a single organization. Cross-org chaining isn't supported.
</details>

---

### Q15
Which API operation is idempotent in HCP Terraform's run lifecycle?

A. Creating a run
B. Approving a run
C. Cancelling a run
D. Reading run status

<details>
<summary>Answer</summary>

**D. Reading run status**

Read operations are always idempotent. Create/approve/cancel modify state and aren't.
</details>

---

### Q16
What's the maximum number of providers you can declare in HCP Terraform's private registry?

A. 10
B. 100
C. There's no hard limit, but soft limits may apply by tier
D. Unlimited on Free tier

<details>
<summary>Answer</summary>

**C. There's no hard limit, but soft limits may apply by tier**

This is a deliberately tricky question. The answer typically isn't tested by exact number — focus on knowing the registry supports both modules and providers, with various tier limits.
</details>

---

### Q17
You use HCP Terraform agents. Where does the agent run?

A. On HCP Terraform infrastructure
B. On infrastructure you operate, behind your firewall — for accessing private resources
C. Inside the Terraform binary
D. On the cloud provider's network

<details>
<summary>Answer</summary>

**B. On infrastructure you operate, behind your firewall — for accessing private resources**

Agents are useful when Terraform needs to manage resources in private networks (e.g., on-premise vSphere, isolated VPCs). HCP Terraform sends jobs to your agent which executes them with internal access.
</details>

---

### Q18
A workspace's "Auto Apply" setting is enabled. What's the workflow?

A. Plans run but never apply
B. Plans run, and if successful, apply automatically without manual approval
C. Plans require approval, applies don't
D. The workspace is read-only

<details>
<summary>Answer</summary>

**B. Plans run, and if successful, apply automatically without manual approval**

Convenient for non-prod, dangerous for prod. Most teams disable auto-apply on production workspaces.
</details>

---

### Q19
What is "speculative plan" in HCP Terraform?

A. A plan that runs without state
B. A plan triggered by a pull request that doesn't actually apply — for review purposes
C. A plan that estimates costs only
D. A plan that uses outdated state

<details>
<summary>Answer</summary>

**B. A plan triggered by a pull request that doesn't actually apply — for review purposes**

Speculative plans run on PRs. They show what would happen if the PR were merged. They cannot be applied — you have to merge first.
</details>

---

### Q20
True or False: HCP Terraform agents are required for the Plus tier.

<details>
<summary>Answer</summary>

**False.**

Agents are an OPTIONAL feature, not required. Many Plus customers use only Remote execution. Agents are useful when you need to access private networks.
</details>

---

## Section 3: State Management Deep Dive (Q21-30)

### Q21
You see this in state for an EC2 instance: `"ami": "ami-12345"`. The cloud actually has `ami-67890`. What's this called and what should you do?

A. Configuration drift; run `terraform apply` to fix it
B. State drift; investigate before deciding
C. State corruption; restore from backup
D. Provider bug; report it

<details>
<summary>Answer</summary>

**B. State drift; investigate before deciding**

Drift means real infrastructure differs from state. Could be intentional (someone fixed something manually) or a problem. `terraform plan -refresh-only` shows the drift; you decide whether to update state or fix the cloud.
</details>

---

### Q22
A state file becomes 100MB. What's likely happening?

A. Normal for large infrastructures
B. Could be drift or many resources; review and consider splitting state
C. Always indicates corruption
D. Need to encrypt it

<details>
<summary>Answer</summary>

**B. Could be drift or many resources; review and consider splitting state**

Large state slows down all operations. Best practice: keep state files focused (e.g., per service or environment). Split using separate root modules + remote state references.
</details>

---

### Q23
You run `terraform state pull > current.tfstate`. What did you do?

A. Locked the state
B. Downloaded a copy of the current state for inspection
C. Pushed local state to remote
D. Forced a state refresh

<details>
<summary>Answer</summary>

**B. Downloaded a copy of the current state for inspection**

`terraform state pull` outputs current state to stdout, which you redirected to a file. Useful for backups or analysis.
</details>

---

### Q24
You modify the state file directly with a text editor. What's the risk?

A. No risk if you're careful
B. State serial number won't increment, breaking subsequent operations and possibly causing data loss
C. Terraform won't notice
D. Only formatting issues

<details>
<summary>Answer</summary>

**B. State serial number won't increment, breaking subsequent operations and possibly causing data loss**

NEVER manually edit state. Always use `terraform state` subcommands. They handle serial numbers, lineage, and locking correctly.
</details>

---

### Q25
What does `terraform state replace-provider` do?

A. Switches providers for all resources
B. Updates state to use a new provider source address (e.g., after registry rename)
C. Reinstalls the provider
D. Replaces the lock file

<details>
<summary>Answer</summary>

**B. Updates state to use a new provider source address (e.g., after registry rename)**

When a provider's namespace changes (e.g., from `terraform-providers/aws` to `hashicorp/aws`), this command updates state to reflect the new source.
</details>

---

### Q26
Your CI runs `terraform plan` and `terraform apply` in separate jobs. The plan generates a `.tfplan` file. What must you do?

A. Pass the plan file as an artifact to the apply job
B. Re-run plan inside the apply job
C. Use `--auto-approve` and skip the plan
D. Use a remote backend with locking

<details>
<summary>Answer</summary>

**A. Pass the plan file as an artifact to the apply job**

Plan files are binary and tied to specific state versions. Save the .tfplan as an artifact in CI, then download it in the apply job and run `terraform apply plan.tfplan`.

State locking helps prevent concurrent runs but doesn't replace passing the plan file.
</details>

---

### Q27
True or False: A plan file can be applied multiple times.

<details>
<summary>Answer</summary>

**False.**

A plan file is single-use. Once applied, the state has changed. Re-applying the same plan would either error or attempt invalid changes.
</details>

---

### Q28
What is the `terraform.tfstate` file's `lineage` field?

A. The Terraform version that created it
B. A unique ID for this state file's history; helps detect when two states have diverged
C. The IAM role used
D. The user who created it

<details>
<summary>Answer</summary>

**B. A unique ID for this state file's history; helps detect when two states have diverged**

If you accidentally have two state files with the same path but different lineages, Terraform warns you — they were created by different `terraform init` instances.
</details>

---

### Q29
You see this error: `Error acquiring state lock`. What does it likely mean?

A. State is corrupted
B. Another Terraform process is currently holding the lock; wait or use `force-unlock`
C. Backend is unavailable
D. Credentials are wrong

<details>
<summary>Answer</summary>

**B. Another Terraform process is currently holding the lock; wait or use `force-unlock`**

Common in CI when a previous job crashed. Verify no run is actually in progress before using `force-unlock`.
</details>

---

### Q30
You enabled S3 versioning on your state bucket. Why is this valuable?

A. Better performance
B. Allows recovery if state is accidentally deleted or corrupted
C. Compresses state automatically
D. Encrypts state

<details>
<summary>Answer</summary>

**B. Allows recovery if state is accidentally deleted or corrupted**

S3 versioning keeps every version of the state file. If a bad apply corrupts state, you can restore a prior version. Combined with encryption + IAM, this is the standard production setup.
</details>

---

## Section 4: Tricky Variables & Functions (Q31-40)

### Q31
What does this validation do?

```hcl
variable "instance_type" {
  type = string
  validation {
    condition     = contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
    error_message = "Invalid instance type."
  }
}
```

A. Allows any instance type
B. Restricts the variable to one of three specific values
C. Validates only if the variable is set
D. Allows wildcards

<details>
<summary>Answer</summary>

**B. Restricts the variable to one of three specific values**

A common pattern for enforcing allowed enums. Pair with a clear error message that lists allowed values.
</details>

---

### Q32
What is a `tuple` in Terraform?

A. A pair of values
B. An ordered, fixed-length sequence where each position has a specific type
C. A synonym for list
D. A read-only list

<details>
<summary>Answer</summary>

**B. An ordered, fixed-length sequence where each position has a specific type**

```hcl
type = tuple([string, number, bool])
default = ["alice", 42, true]
```
Different positions can have different types — unlike a list where all elements share one type.
</details>

---

### Q33
What's the difference between `null` and an empty string `""` in Terraform?

A. They are the same
B. `null` means "not set" — Terraform may use defaults; `""` is a valid empty string
C. `null` errors in HCL
D. `""` is converted to `null`

<details>
<summary>Answer</summary>

**B. `null` means "not set" — Terraform may use defaults; `""` is a valid empty string**

Setting an argument to `null` is equivalent to omitting it. Important for optional arguments where the provider applies a default.
</details>

---

### Q34
What does `optional()` do in a type constraint?

A. Makes a variable optional
B. Marks an attribute in an object type as optional with a default
C. Creates a nullable variable
D. Skips validation

<details>
<summary>Answer</summary>

**B. Marks an attribute in an object type as optional with a default**

```hcl
variable "settings" {
  type = object({
    name    = string
    enabled = optional(bool, true)
  })
}
```
Callers can omit `enabled`; it defaults to `true`.
</details>

---

### Q35
You have a variable:

```hcl
variable "tags" {
  type    = map(string)
  default = {}
}
```

You set `TF_VAR_tags=foo`. What happens?

A. `tags` is set to `{}`
B. Terraform errors because `foo` isn't valid map syntax
C. `tags` is set to a map containing one key
D. The env var is ignored

<details>
<summary>Answer</summary>

**B. Terraform errors because `foo` isn't valid map syntax**

For complex types via env vars, use HCL or JSON syntax: `TF_VAR_tags='{Environment="prod"}'` or `TF_VAR_tags='{"Environment":"prod"}'`.
</details>

---

### Q36
What does `coalesce("", null, "hello", "world")` return?

A. `""`
B. `null`
C. `"hello"`
D. `"world"`

<details>
<summary>Answer</summary>

**C. `"hello"`**

`coalesce` returns the first non-null, non-empty argument. Skips `""` and `null`, returns `"hello"`.

There's also `coalescelist` which works on lists, returning the first non-empty list.
</details>

---

### Q37
Which is true about `local` values?

A. They can be overridden by command line
B. They can reference variables, resources, data sources, and other locals
C. They are stored in state separately
D. They are deprecated

<details>
<summary>Answer</summary>

**B. They can reference variables, resources, data sources, and other locals**

Locals are computed values, like local variables in code. Useful for DRY-ing up repeated expressions.
</details>

---

### Q38
What's the result of `length({a = 1, b = 2})`?

A. `0`
B. `1`
C. `2`
D. Error — `length` doesn't work on maps

<details>
<summary>Answer</summary>

**C. `2`**

`length` works on strings, lists, sets, and maps. For maps, it returns the number of keys.
</details>

---

### Q39
Examine this code:

```hcl
locals {
  servers = ["web1", "web2", "web3"]
  with_index = [for i, s in local.servers : "${i}: ${s}"]
}
```

What is `local.with_index`?

A. `["web1", "web2", "web3"]`
B. `["0: web1", "1: web2", "2: web3"]`
C. Error — for expressions don't accept indices
D. `[0, 1, 2]`

<details>
<summary>Answer</summary>

**B. `["0: web1", "1: web2", "2: web3"]`**

`for i, s in list` gives both index and value. Same with maps: `for k, v in map`.
</details>

---

### Q40
What does `flatten()` do?

A. Compresses a list
B. Removes one level of nesting from lists
C. Sorts a list
D. Converts a map to a list

<details>
<summary>Answer</summary>

**B. Removes one level of nesting from lists**

`flatten([[1,2],[3,4],[5]])` → `[1,2,3,4,5]`. Useful with nested for expressions producing list-of-lists.
</details>

---

## Section 5: Final Real-World Scenarios (Q41-50)

### Q41
You're on-call. A teammate ran `terraform apply` and it crashed mid-apply. What's the FIRST thing to check?

A. Run `terraform apply` again
B. Check state lock status; if stuck, investigate before unlocking
C. Restore state from backup
D. Run `terraform destroy`

<details>
<summary>Answer</summary>

**B. Check state lock status; if stuck, investigate before unlocking**

A crashed apply might:
- Have left a lock in place
- Have partially modified state
- Have created/destroyed real resources

Run `terraform plan` to see current state vs reality. NEVER blindly re-run apply.
</details>

---

### Q42
Your config has 200 resources. Apply takes 10 minutes. You only need to update one resource quickly. What should you do?

A. `terraform apply -target=resource.address`
B. `terraform apply -parallelism=50`
C. Both A and B can help, but `-target` is discouraged for routine work
D. Run apply in parallel with another instance

<details>
<summary>Answer</summary>

**C. Both A and B can help, but `-target` is discouraged for routine work**

`-parallelism=50` increases concurrency (default 10). `-target` works for emergencies but breaks the dependency graph and is meant for exceptional situations only.
</details>

---

### Q43
You inherit a Terraform codebase using AWS provider 2.x. The module you want to add requires AWS provider 5.x. What's your strategy?

A. Refuse to add the module
B. Upgrade the AWS provider, fix any breaking changes, then add the module
C. Use two providers in the same config
D. Fork the module and downgrade

<details>
<summary>Answer</summary>

**B. Upgrade the AWS provider, fix any breaking changes, then add the module**

Provider upgrades require care — read the upgrade guide. AWS 2.x → 5.x has many breaking changes. Plan, test in dev, document.
</details>

---

### Q44
You see this in a config:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t3.micro"

  user_data = <<-EOF
    #!/bin/bash
    echo "Starting up"
  EOF
}
```

You change `user_data`. What does Terraform plan?

A. Update in place
B. Replace the instance (destroy and create new)
C. No change — user_data is metadata only
D. Error

<details>
<summary>Answer</summary>

**B. Replace the instance (destroy and create new)**

`user_data` is "ForceNew" in the AWS provider — it can only be set at creation. Changing it requires replacing the instance.

Use `user_data_replace_on_change = false` (newer feature) to override this in some cases.
</details>

---

### Q45
You want to manage S3 bucket policies separately from buckets. Pattern?

A. Use `aws_s3_bucket_policy` resource attached to the bucket via `bucket = aws_s3_bucket.main.id`
B. Embed policy inline in the bucket resource
C. Use a data source
D. Both A and B work; A is preferred for separation of concerns

<details>
<summary>Answer</summary>

**D. Both A and B work; A is preferred for separation of concerns**

Newer AWS provider versions (4.x+) split many bucket settings into separate resources (`aws_s3_bucket_versioning`, `aws_s3_bucket_policy`, etc.) for clearer dependency graphs.
</details>

---

### Q46
True or False: You can use `terraform import` on a resource defined inside a module.

<details>
<summary>Answer</summary>

**True.**

```bash
terraform import module.network.aws_vpc.main vpc-12345
```

The address includes the module path. Use `import` blocks for declarative imports of module resources too.
</details>

---

### Q47
A team uses long-lived AWS access keys in `terraform.tfvars`. Risk?

A. Performance issues
B. Security: keys committed to git get exposed; keys don't rotate
C. Both A and B
D. No risk if encrypted

<details>
<summary>Answer</summary>

**B. Security: keys committed to git get exposed; keys don't rotate**

Best practices:
- Use IAM roles (instance profiles, OIDC for CI, AssumeRole)
- Use AWS SSO short-lived credentials
- Use Vault for dynamic secrets
- NEVER commit credentials to git
</details>

---

### Q48
You want to enforce that all resources have a `Owner` tag. Which approach is best?

A. Code review only
B. Sentinel policy (HCP Terraform Plus+)
C. AWS provider's `default_tags` block
D. Both B and C — defense in depth

<details>
<summary>Answer</summary>

**D. Both B and C — defense in depth**

`default_tags` ensures all resources created by the provider get the tags. Sentinel adds policy enforcement at the org level. Combine for the strongest guarantee.
</details>

---

### Q49
A new junior engineer writes:
```hcl
resource "aws_security_group" "web" {
  ingress {
    from_port   = 0
    to_port     = 65535
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

What's the issue?

A. Syntax error
B. Opens all TCP ports to the entire internet — major security risk
C. Missing required attribute
D. Should use `egress`

<details>
<summary>Answer</summary>

**B. Opens all TCP ports to the entire internet — major security risk**

Always question overly permissive rules in code review. Sentinel policies can prevent this from being applied.
</details>

---

### Q50
You're preparing for the exam tomorrow. What's the most efficient last-minute review?

A. Read the entire Terraform docs
B. Take 1-2 timed practice exams; review missed topics; review your weak areas; sleep well
C. Memorize all Terraform functions
D. Refactor your codebase

<details>
<summary>Answer</summary>

**B. Take 1-2 timed practice exams; review missed topics; review your weak areas; sleep well**

Practice exams under time pressure are the best preparation. Identify gaps, fix them, and rest. Don't try to learn new topics the night before — consolidate what you know.
</details>

---

## Final Tips

After completing all 3 sets (150 questions), you should be well-prepared. Key reminders for exam day:

1. **Read questions carefully** — distractors are designed to look right
2. **"Always" and "never" answers** are usually wrong
3. **HCP-specific features** — know what's Free vs Plus vs Enterprise
4. **Module syntax** — `module.<NAME>.<OUTPUT>` is the only way
5. **State precedence** — refresh-only, lock errors, drift detection
6. **CLI flags** — `-target`, `-replace`, `-refresh-only`, `-out`
7. **Functions** — at least know `length`, `lookup`, `merge`, `flatten`, `cidrsubnet`

Good luck on your exam!
