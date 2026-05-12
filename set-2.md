# Terraform Associate — Practice Set 2 (50 Questions)

**Focus**: Scenario-based, real-world situations
**Difficulty**: Medium

> **How to use**: Try to answer before reading the explanation. Click `<details>` to reveal the answer.

---

## Section 1: Providers & Initialization (Q1-10)

### Q1
You add a new provider to your config and run `terraform plan`. The plan fails. What's the most likely cause?

A. The provider doesn't support `terraform plan`
B. You need to run `terraform init` again to download the new provider
C. The state file is corrupted
D. You need to delete `.terraform.lock.hcl`

<details>
<summary>Answer</summary>

**B. You need to run `terraform init` again to download the new provider**

`terraform init` must be re-run when:
- New providers are added
- Modules change source or version
- Backend changes
</details>

---

### Q2
What is the purpose of `.terraform.lock.hcl`?

A. Locks the state file from concurrent edits
B. Records exact provider versions and hashes for reproducibility
C. Stores cached provider plugins
D. Prevents accidental `terraform destroy`

<details>
<summary>Answer</summary>

**B. Records exact provider versions and hashes for reproducibility**

This file SHOULD be committed to version control. It ensures everyone on the team uses identical provider versions.
</details>

---

### Q3
You declare a provider but don't specify a `source`:

```hcl
terraform {
  required_providers {
    aws = {
      version = "~> 5.0"
    }
  }
}
```

What happens?

A. Terraform errors immediately
B. Terraform assumes `hashicorp/aws` (the default namespace)
C. Terraform searches all namespaces and picks the first match
D. The provider is treated as a community provider

<details>
<summary>Answer</summary>

**B. Terraform assumes `hashicorp/aws` (the default namespace)**

If `source` is omitted, Terraform defaults to `registry.terraform.io/hashicorp/<NAME>`. Best practice is to always specify it explicitly.
</details>

---

### Q4
What is the difference between Tier 1 (Official) and Tier 2 (Partner) providers?

A. Official providers are faster
B. Official providers are maintained by HashiCorp; Partner providers are maintained by the technology vendor
C. Partner providers cost money
D. Partner providers cannot be used in production

<details>
<summary>Answer</summary>

**B. Official providers are maintained by HashiCorp; Partner providers are maintained by the technology vendor**

- **Official** (HashiCorp namespace) — built/maintained by HashiCorp
- **Partner** — maintained by the tech provider, verified by HashiCorp
- **Community** — maintained by community, no verification badge
</details>

---

### Q5
You see this error: `Error: Failed to query available provider packages`. What's a likely cause?

A. The state file is missing
B. Network can't reach the Terraform Registry
C. The HCL syntax is wrong
D. The cloud credentials are invalid

<details>
<summary>Answer</summary>

**B. Network can't reach the Terraform Registry**

`terraform init` needs to fetch provider metadata from `registry.terraform.io`. Common causes: corporate proxy, firewall, or no internet.
</details>

---

### Q6
Where does Terraform install provider plugins by default?

A. `/usr/local/terraform/plugins/`
B. `~/.terraform.d/plugins/`
C. `.terraform/providers/` in the working directory
D. `/etc/terraform/`

<details>
<summary>Answer</summary>

**C. `.terraform/providers/` in the working directory**

Each working directory is self-contained — its own providers, modules, and lock file.
</details>

---

### Q7
What is a provider mirror?

A. A backup copy of the state
B. A local or network directory holding cached provider plugins for offline/restricted environments
C. A read-only replica of HCP Terraform
D. A failover provider for redundancy

<details>
<summary>Answer</summary>

**B. A local or network directory holding cached provider plugins for offline/restricted environments**

Use `terraform providers mirror <dir>` to populate it. Useful in air-gapped networks where direct registry access isn't allowed.
</details>

---

### Q8
You want to use two different versions of the AWS provider in the same config. Is this possible?

A. Yes, with provider aliases
B. Yes, with provider mirrors
C. No, only one version of a given provider can exist per configuration
D. Yes, with separate workspaces

<details>
<summary>Answer</summary>

**C. No, only one version of a given provider can exist per configuration**

Aliases let you have multiple CONFIGURATIONS of the same provider version, but you cannot have two different VERSIONS simultaneously.
</details>

---

### Q9
What does `terraform init -backend=false` do?

A. Disables the backend permanently
B. Initializes everything except the backend (useful for syntax validation)
C. Uses local state instead of remote
D. Skips downloading providers

<details>
<summary>Answer</summary>

**B. Initializes everything except the backend (useful for syntax validation)**

Useful in CI when you want to run `terraform validate` without configuring the actual remote backend.
</details>

---

### Q10
Which command updates providers to the latest version allowed by your version constraints?

A. `terraform init`
B. `terraform init -upgrade`
C. `terraform providers upgrade`
D. `terraform refresh`

<details>
<summary>Answer</summary>

**B. `terraform init -upgrade`**

Plain `init` keeps using already-installed versions. The `-upgrade` flag fetches the latest matching version and updates the lock file.
</details>

---

## Section 2: Resources, Data & Lifecycle (Q11-20)

### Q11
You write this lifecycle block. What does it do?

```hcl
lifecycle {
  prevent_destroy = true
}
```

A. Prevents the resource from being modified
B. Causes Terraform to error if a plan would destroy this resource
C. Prevents `terraform destroy` only; `terraform apply` can still destroy it
D. Encrypts the resource

<details>
<summary>Answer</summary>

**B. Causes Terraform to error if a plan would destroy this resource**

This is a safety guard. You must remove the `prevent_destroy` flag or delete the entire resource block to actually destroy it.
</details>

---

### Q12
What does `create_before_destroy = true` accomplish?

A. Creates the new resource before destroying the old one (zero-downtime replacement)
B. Validates the resource before creation
C. Creates a backup before destroying
D. Prevents destruction entirely

<details>
<summary>Answer</summary>

**A. Creates the new resource before destroying the old one (zero-downtime replacement)**

Useful for resources that must always be available, like load balancers or NAT gateways.
</details>

---

### Q13
You set `ignore_changes = [tags]`. What happens if someone manually changes tags in the cloud console?

A. Terraform reverts the tags on next apply
B. Terraform ignores the drift in tags during plan
C. Terraform errors on next plan
D. Terraform updates state to match but never modifies real resources

<details>
<summary>Answer</summary>

**B. Terraform ignores the drift in tags during plan**

`ignore_changes` tells Terraform "I don't care if this attribute changes externally." Common for tags managed by other automation.
</details>

---

### Q14
What is the difference between a `resource` block and a `data` block?

A. `resource` is read-only; `data` creates infrastructure
B. `resource` creates/manages infrastructure; `data` reads existing infrastructure
C. They are aliases
D. `data` blocks can only query AWS resources

<details>
<summary>Answer</summary>

**B. `resource` creates/manages infrastructure; `data` reads existing infrastructure**

Use `data` to fetch info about resources NOT managed by your Terraform (e.g., looking up an existing AMI ID).
</details>

---

### Q15
You have a data source that depends on a managed resource. How do you express this?

A. Use `depends_on` in the data block
B. Data sources cannot depend on managed resources
C. Reference the resource's attribute in the data source's filter
D. Both A and C work

<details>
<summary>Answer</summary>

**D. Both A and C work**

Reference an attribute (implicit dependency) OR use `depends_on` (explicit dependency). The latter is for cases where there's no natural attribute reference.
</details>

---

### Q16
What happens if you reference a `count.index` in a resource without setting `count`?

A. Defaults to index 0
B. Terraform errors — `count.index` is only valid when `count` is set
C. The resource is skipped
D. It's interpreted as `for_each.key`

<details>
<summary>Answer</summary>

**B. Terraform errors — `count.index` is only valid when `count` is set**

`count.index` and `each.key`/`each.value` only exist within their respective contexts.
</details>

---

### Q17
You want to create one S3 bucket per environment. The environments are stored as a map. Which meta-argument is most appropriate?

A. `count`
B. `for_each`
C. `depends_on`
D. `lifecycle`

<details>
<summary>Answer</summary>

**B. `for_each`**

For map/set inputs with distinct identities, `for_each` is preferred. It avoids the "shifting index" problem that `count` has when items are removed.
</details>

---

### Q18
What is the difference between `count` and `for_each` when removing one item?

A. They behave identically
B. `count` shifts indices, potentially destroying/recreating later items; `for_each` only affects the removed item
C. `for_each` is faster
D. `count` cannot remove items

<details>
<summary>Answer</summary>

**B. `count` shifts indices, potentially destroying/recreating later items; `for_each` only affects the removed item**

Classic gotcha. If you have `count = 5` and remove the middle item, items at indices 3 and 4 shift to 2 and 3, causing destruction and recreation. `for_each` avoids this.
</details>

---

### Q19
What does the `replace_triggered_by` lifecycle argument do?

A. Replaces the resource when triggered by another resource's change
B. Schedules replacement at a specific time
C. Replaces the provider
D. Triggers a replacement on every apply

<details>
<summary>Answer</summary>

**A. Replaces the resource when triggered by another resource's change**

```hcl
lifecycle {
  replace_triggered_by = [aws_security_group.main]
}
```
If the SG changes, the resource using this lifecycle block gets replaced.
</details>

---

### Q20
You see this in code:
```hcl
resource "aws_instance" "web" {
  count = var.create_instance ? 1 : 0
  # ...
}
```

What pattern is this?

A. A bug
B. Conditional resource creation — creates the resource only if `var.create_instance` is true
C. Multi-region deployment
D. Lifecycle management

<details>
<summary>Answer</summary>

**B. Conditional resource creation — creates the resource only if `var.create_instance` is true**

Common pattern. Reference it as `aws_instance.web[0]` (when count = 1) or use `aws_instance.web[*].id` for splat.
</details>

---

## Section 3: HCL Functions & Expressions (Q21-30)

### Q21
What does `lookup({a=1, b=2}, "c", 99)` return?

A. `null`
B. An error
C. `99`
D. `0`

<details>
<summary>Answer</summary>

**C. `99`**

`lookup(map, key, default)` returns the default if the key isn't found. Without the default, it would error.
</details>

---

### Q22
You need to merge two maps with the second taking precedence on conflicts. Which function?

A. `concat()`
B. `merge()`
C. `combine()`
D. `union()`

<details>
<summary>Answer</summary>

**B. `merge()`**

```hcl
merge({a = 1, b = 2}, {b = 99, c = 3})
# Result: {a = 1, b = 99, c = 3}
```
Later arguments override earlier ones.
</details>

---

### Q23
What does this for expression produce?
```hcl
[for s in ["a", "b", "c"] : upper(s) if s != "b"]
```

A. `["A", "B", "C"]`
B. `["A", "C"]`
C. `["a", "c"]`
D. `["A", "b", "C"]`

<details>
<summary>Answer</summary>

**B. `["A", "C"]`**

The `if` clause filters out "b". Then `upper()` transforms remaining items.
</details>

---

### Q24
What does the splat expression `aws_instance.web[*].id` do?

A. Returns the ID of the first instance
B. Returns a list of IDs from all instances created with `count`
C. Multiplies the IDs
D. Returns a random ID

<details>
<summary>Answer</summary>

**B. Returns a list of IDs from all instances created with `count`**

Splat expressions extract a single attribute from all elements. Equivalent to `[for i in aws_instance.web : i.id]`.
</details>

---

### Q25
What does `cidrsubnet("10.0.0.0/16", 8, 1)` return?

A. `"10.0.0.0/16"`
B. `"10.0.1.0/24"`
C. `"10.1.0.0/16"`
D. `"10.0.0.1/24"`

<details>
<summary>Answer</summary>

**B. `"10.0.1.0/24"`**

`cidrsubnet(prefix, newbits, netnum)`:
- prefix = `10.0.0.0/16`
- newbits = 8 → result is /24
- netnum = 1 → second subnet (10.0.1.0)
</details>

---

### Q26
What's the purpose of the `try()` function?

A. Try to validate config
B. Attempt an expression and return a fallback if it errors
C. Test for null
D. Throw an exception

<details>
<summary>Answer</summary>

**B. Attempt an expression and return a fallback if it errors**

```hcl
try(var.config.region, "us-east-1")
```
If `var.config.region` doesn't exist or errors, returns "us-east-1".
</details>

---

### Q27
What does `can()` return?

A. The result of an expression
B. A boolean indicating whether the expression succeeded
C. The error message if any
D. The type of an expression

<details>
<summary>Answer</summary>

**B. A boolean indicating whether the expression succeeded**

```hcl
can(tonumber("abc"))  # false
can(tonumber("42"))   # true
```
Useful in validation conditions.
</details>

---

### Q28
What's the difference between `toset()` and `tolist()`?

A. They are aliases
B. `toset` removes duplicates and unorders; `tolist` preserves order and allows duplicates
C. `toset` is for strings only
D. `tolist` is deprecated

<details>
<summary>Answer</summary>

**B. `toset` removes duplicates and unorders; `tolist` preserves order and allows duplicates**

Sets are unordered with unique elements. Lists are ordered with possible duplicates.
</details>

---

### Q29
Which function reads a file's contents as a string?

A. `read("path")`
B. `file("path")`
C. `load("path")`
D. `cat("path")`

<details>
<summary>Answer</summary>

**B. `file("path")`**

Returns the file content as a string. Use `filebase64()` for binary files. Use `templatefile()` to render with variables.
</details>

---

### Q30
What does `templatefile("user-data.tftpl", { name = "alice" })` do?

A. Reads the file and replaces `${name}` with "alice"
B. Creates a new file
C. Validates the template syntax
D. Encrypts the file

<details>
<summary>Answer</summary>

**A. Reads the file and replaces `${name}` with "alice"**

Templates use HCL interpolation syntax. Common for user_data scripts and config files.
</details>

---

## Section 4: Backends & Workspaces (Q31-40)

### Q31
What's a benefit of using a remote backend over a local backend?

A. Faster `terraform apply`
B. Shared state, locking, and encryption at rest
C. Cheaper
D. No need for credentials

<details>
<summary>Answer</summary>

**B. Shared state, locking, and encryption at rest**

Remote backends enable team collaboration safely. Local backends are fine for solo development but don't scale to teams.
</details>

---

### Q32
For an S3 backend, what's typically used to provide state locking?

A. S3 versioning
B. DynamoDB table
C. Lambda function
D. Locking is not supported for S3

<details>
<summary>Answer</summary>

**B. DynamoDB table**

Configure a DynamoDB table with a partition key `LockID` (String). Reference it in the backend config:
```hcl
backend "s3" {
  bucket         = "my-state"
  key            = "prod.tfstate"
  region         = "us-east-1"
  dynamodb_table = "tf-state-lock"
}
```
</details>

---

### Q33
You have separate dev/staging/prod environments. Which approach offers the STRONGEST isolation?

A. Use OSS workspaces in a single config
B. Use separate root modules with separate backends per environment
C. Use environment variables to differentiate
D. Use git branches

<details>
<summary>Answer</summary>

**B. Use separate root modules with separate backends per environment**

OSS workspaces share the same config. For real environment isolation (different AWS accounts, different state files, different access controls), separate root modules + backends is the best practice.
</details>

---

### Q34
True or False: HCP Terraform workspaces are equivalent to OSS workspaces.

<details>
<summary>Answer</summary>

**False.**

OSS workspaces = multiple state instances of the same config.
HCP Terraform workspaces = independent unit with its own config, state, variables, team access, and run history.

They share a name, but they're different concepts.
</details>

---

### Q35
What does `terraform.workspace` return?

A. The path to the current workspace
B. The name of the current workspace as a string
C. A list of all workspaces
D. The state file location

<details>
<summary>Answer</summary>

**B. The name of the current workspace as a string**

In the default workspace, returns `"default"`. You can use it for conditional config:
```hcl
instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"
```
</details>

---

### Q36
You delete a workspace with `terraform workspace delete dev`. What happens?

A. The state and resources are destroyed
B. The state file for that workspace is deleted, but cloud resources remain
C. Terraform errors if the workspace has resources
D. Both B and C — by default Terraform errors if state has resources, but you can force with `-force`

<details>
<summary>Answer</summary>

**D. Both B and C — by default Terraform errors if state has resources, but you can force with `-force`**

Terraform protects you from accidentally deleting a workspace whose state still references real resources. `-force` overrides this.
</details>

---

### Q37
What's a partial backend configuration?

A. A backend that only handles part of the state
B. A backend block where some values are omitted and provided at init time via `-backend-config`
C. A backup backend
D. A backend that's not yet finalized

<details>
<summary>Answer</summary>

**B. A backend block where some values are omitted and provided at init time via `-backend-config`**

```hcl
terraform {
  backend "s3" {
    bucket = "my-state"
    region = "us-east-1"
    # key is omitted
  }
}
```
Then: `terraform init -backend-config="key=prod.tfstate"`

Useful for reusing config across environments.
</details>

---

### Q38
A team accidentally configures two workspaces to use the same state file path. What's the risk?

A. They will overwrite each other's state, causing corruption
B. Terraform will detect and prevent this
C. State is automatically merged
D. Performance degradation only

<details>
<summary>Answer</summary>

**A. They will overwrite each other's state, causing corruption**

Each workspace MUST have its own state file path. State locking helps but doesn't prevent misconfiguration.
</details>

---

### Q39
What backend is used by default if you don't specify one?

A. `s3`
B. `local`
C. `cloud`
D. `remote`

<details>
<summary>Answer</summary>

**B. `local`**

Default is the local backend, which stores state in `terraform.tfstate` in the working directory.
</details>

---

### Q40
You have local state and want to migrate to S3. After updating the backend block, what command do you run?

A. `terraform apply`
B. `terraform init -migrate-state`
C. `terraform state push`
D. `terraform backend migrate`

<details>
<summary>Answer</summary>

**B. `terraform init -migrate-state`**

Terraform reads the local state, copies it to S3, and switches to using S3 going forward. You'll be prompted for confirmation.
</details>

---

## Section 5: Mixed Real-World Scenarios (Q41-50)

### Q41
Your colleague writes:
```hcl
resource "aws_instance" "app" {
  ami = data.aws_ami.latest.id
}

data "aws_ami" "latest" {
  most_recent = true
  owners      = ["amazon"]
}
```

The data block references a managed resource? Or vice versa?

A. The resource references the data source — implicit dependency
B. The data references the resource — implicit dependency
C. They are independent
D. This causes a circular dependency error

<details>
<summary>Answer</summary>

**A. The resource references the data source — implicit dependency**

The resource depends on the data source. Terraform reads the data source first, then creates the resource using its result.
</details>

---

### Q42
You're using HCP Terraform with the cloud block:
```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      tags = ["prod"]
    }
  }
}
```

What does this do?

A. Connects to all workspaces tagged "prod"
B. Connects to a single workspace named "prod"
C. Tags the workspace with "prod"
D. Filters runs by tag

<details>
<summary>Answer</summary>

**A. Connects to all workspaces tagged "prod"**

Using `tags` (instead of `name`) lets you work with multiple workspaces. `terraform workspace list` will show all matching ones, and you can switch between them.
</details>

---

### Q43
You need to deploy the same module to 5 AWS regions. Which approach is best?

A. Copy-paste the module 5 times
B. Use a `for_each` over a map of regions, with a provider alias per region
C. Use a single workspace and switch providers
D. Modules don't support multi-region

<details>
<summary>Answer</summary>

**B. Use a `for_each` over a map of regions, with a provider alias per region**

Or alternatively, use 5 module blocks each with a different provider. The second approach is sometimes clearer.

Note: `for_each` on modules with different providers is tricky. The most common pattern is 5 separate module blocks.
</details>

---

### Q44
What's the safest way to handle this situation: you must ROTATE an EC2 key pair without downtime?

A. Delete the old key pair, then create the new one
B. Use `create_before_destroy` lifecycle on the key pair resource
C. Manual intervention only
D. Use `taint`

<details>
<summary>Answer</summary>

**B. Use `create_before_destroy` lifecycle on the key pair resource**

```hcl
lifecycle {
  create_before_destroy = true
}
```
New key created first, then old one removed. Combined with proper naming (e.g., including a hash), this allows zero-downtime rotation.
</details>

---

### Q45
True or False: `terraform validate` makes API calls to the cloud provider.

<details>
<summary>Answer</summary>

**False.**

`terraform validate` is offline. It checks HCL syntax, references, and types without contacting any provider. Use it in CI for fast feedback before running plans.
</details>

---

### Q46
You see this in someone's CI pipeline:
```bash
terraform init -input=false
terraform plan -input=false -out=plan.tfplan
terraform apply -input=false plan.tfplan
```

What does `-input=false` do?

A. Disables stdin/interactive prompts (essential for CI/CD)
B. Skips reading the `input.tf` file
C. Disables variable inputs
D. Speeds up the run

<details>
<summary>Answer</summary>

**A. Disables stdin/interactive prompts (essential for CI/CD)**

CI environments don't have a terminal. `-input=false` ensures Terraform fails immediately rather than hanging waiting for user input.
</details>

---

### Q47
You want to debug why Terraform is making strange decisions. What's the best first step?

A. Delete the state file
B. Set `TF_LOG=DEBUG` (or TRACE) and re-run
C. Reinstall Terraform
D. Run with `--verbose`

<details>
<summary>Answer</summary>

**B. Set `TF_LOG=DEBUG` (or TRACE) and re-run**

Combine with `TF_LOG_PATH=/tmp/tf.log` to write to a file. TRACE is most verbose; useful for deep debugging including provider API calls.
</details>

---

### Q48
A teammate's plan shows a resource will be replaced unexpectedly. What's the most useful command to investigate?

A. `terraform validate`
B. `terraform plan -refresh-only`
C. `terraform show` to inspect current state
D. `terraform state show <address>` to see the current state of that specific resource

<details>
<summary>Answer</summary>

**D. `terraform state show <address>` to see the current state of that specific resource**

You can compare the state's view of the resource against the config to spot the mismatch. Combined with the plan output, you can pinpoint which attribute changed.
</details>

---

### Q49
You're writing a module that creates an S3 bucket with versioning. What's the BEST way to expose the bucket name to consumers?

A. As a hardcoded string in the documentation
B. As an output named `bucket_name` (or `id`)
C. As a variable
D. As a data source

<details>
<summary>Answer</summary>

**B. As an output named `bucket_name` (or `id`)**

Outputs are the standard way modules expose data to callers. Best practice: include `description` for each output.
</details>

---

### Q50
Final scenario: You inherit a Terraform codebase with no documentation. What's a good order to learn it?

A. Read all .tf files top-to-bottom
B. Run `terraform init`, then `terraform plan` to see what would be created/changed; then explore by module
C. Run `terraform destroy` to start fresh
D. Rewrite from scratch

<details>
<summary>Answer</summary>

**B. Run `terraform init`, then `terraform plan` to see what would be created/changed; then explore by module**

A clean plan tells you the codebase matches reality. From there, walk through modules, variables, and outputs. `terraform graph` can also help visualize dependencies.
</details>

---

## Scoring

- **45-50**: Excellent — schedule the exam
- **40-44**: Good — review missed topics
- **35-39**: Fair — focus on weak sections
- **<35**: Need more study

Move on to **Set 3** for harder, edge-case questions when you're ready.
