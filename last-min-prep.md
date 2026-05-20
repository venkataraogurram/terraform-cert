# Terraform Associate — Last Minute Prep (Complete)

**Read this the night before and morning of the exam.**

This is a high-density review covering every concept tested across 6 mock exams plus all official exam objectives.

---

## 60-Second Mental Map

```
Terraform   = Declarative IaC tool by HashiCorp
Workflow    = Write → Init → Validate → Plan → Apply → Destroy
State       = Maps config to real resources (CRITICAL — protect it)
Modules     = Reusable groups of resources
Providers   = Plugins that talk to APIs (AWS, GCP, Azure, etc.)
HCP Terraform = HashiCorp's managed SaaS (state + remote runs + governance)
```

---

## Top 40 Facts to Memorize

1. `terraform init` — required first; downloads providers + modules + sets up backend
2. `terraform validate` — offline syntax/type check (no API calls, no provider validation)
3. `terraform fmt` — canonical formatting (current dir only; use `-recursive` for subdirs)
4. `terraform fmt -check` — CI-friendly; non-zero exit if files need formatting
5. `terraform plan` — shows changes; `-out=file.tfplan` to save
6. `terraform apply file.tfplan` — applies a saved plan exactly
7. `terraform destroy` — destroys all managed infrastructure (prompts unless `-auto-approve`)
8. `terraform graph` — generates DAG in DOT format for visualization
9. `terraform login` — authenticates CLI to HCP Terraform; stores token in `~/.terraform.d/credentials`
10. `terraform test` — runs `.tftest.hcl` files for module testing
11. `.terraform/` — local cache (providers + modules), DO NOT commit
12. `.terraform.lock.hcl` — provider version lock, **DO commit**
13. `terraform.tfstate` — state file, **DO NOT commit**
14. `terraform.tfstate.backup` — auto-created before each state write
15. State is plaintext — `sensitive = true` only redacts CLI output, NOT state
16. State locking prevents concurrent runs (S3 needs DynamoDB; local has none)
17. `terraform force-unlock <ID>` — manually release stuck lock
18. `terraform import` — imperative; requires matching resource block first
19. `import` block (1.5+) — declarative import; use `terraform plan` + `apply`
20. `moved` block (1.1+) — rename/move WITHIN same state without recreation
21. `removed` block (1.7+) — stop managing without destroying
22. Cross-state migration: `removed` (source) + `import` (destination)
23. `count` — integer index; `for_each` — string key (preferred)
24. `for_each` REQUIRES map or set (use `toset()` for lists)
25. `terraform.workspace` — current workspace name as string
26. OSS workspaces ≠ HCP Terraform workspaces (different concepts!)
27. `TF_VAR_<name>` — env var prefix for setting variables
28. `TF_LOG=TRACE` (or DEBUG/INFO/WARN/ERROR) — enable logging
29. `TF_LOG_PATH=/path` — write logs to file (separate from TF_LOG)
30. Variable precedence: defaults < env < tfvars < auto.tfvars < -var-file < -var
31. Module reference: `module.<NAME>.<OUTPUT>` — only way to access child output
32. Module `version` — optional but RECOMMENDED; only works with registry sources
33. `~> 5.0` means `>= 5.0, < 6.0` (allows minor); `~> 5.0.0` means `>= 5.0.0, < 5.1.0`
34. Sentinel — Plus/Enterprise only; advisory / soft-mandatory / hard-mandatory
35. Audit logging — Enterprise ONLY
36. Run trigger — auto-queues run in workspace B after A applies successfully
37. Run task — calls external tools between plan and apply (e.g., security scan)
38. CLI-driven workflow with HCP Terraform — runs execute REMOTELY by default
39. Use `cloud {}` block (NOT `backend "remote"`) for HCP Terraform integration
40. Each platform needs its own provider (AWS = aws, GCP = google, etc.)

---

## Most Common Exam Traps

### Trap 1: Module output access
- **Wrong**: `output.module.x`, `var.module.x`, `module.x.outputs.y`
- **Right**: `module.<NAME>.<OUTPUT_NAME>` — and only if child declares the output

### Trap 2: Module variable scope
- Child modules CANNOT see parent variables (no inheritance)
- Parent must explicitly pass values via `module "x" { var = value }`
- Child must declare `variable "var" {}` to receive it

### Trap 3: HCP Terraform CLI-driven mode
- **Wrong**: "runs on local machine"
- **Right**: runs on HCP Terraform infrastructure, output streamed back

### Trap 4: HCP Terraform Local execution mode
- **Wrong**: "blocks all CLI usage"
- **Right**: only stores state remotely; plans/applies run locally

### Trap 5: `sensitive = true`
- **Wrong**: "encrypts in state" or "prevents storage in state"
- **Right**: only redacts from CLI output; STILL plaintext in state
- Only `ephemeral` resources / `password_wo` keep values out of state

### Trap 6: `lifecycle.prevent_destroy`
- **Wrong**: "prevents updates"
- **Right**: errors if a plan would destroy; doesn't stop in-place updates

### Trap 7: `count` vs `for_each` deletion behavior
- `count`: removing middle item shifts indices, may destroy/recreate later items
- `for_each`: only the removed item is destroyed; others untouched

### Trap 8: OSS workspaces for prod isolation
- **Wrong**: "use workspaces for dev/staging/prod"
- **Right**: workspaces share config; for true isolation use separate root modules + backends

### Trap 9: `terraform refresh` (deprecated)
- Use `terraform plan -refresh-only` (read-only check)
- Use `terraform apply -refresh-only` (write changes to state)

### Trap 10: `terraform taint` (deprecated)
- Use `terraform apply -replace=<address>` instead

### Trap 11: Module `version` argument
- Optional, but RECOMMENDED
- Only works with registry sources (not git, local, s3)
- Without version, latest is downloaded (risky)

### Trap 12: Backend init flags
- `-migrate-state` = copy existing state to new backend
- `-reconfigure` = ignore existing state, fresh start in new backend
- `-backend=false` = skip backend init entirely

### Trap 13: validate vs plan
- `validate` — offline; syntax/types only; NO provider-specific checks
- `plan` — online; checks against real provider state and APIs
- `validate` will pass even if a required provider argument like `region` is missing

### Trap 14: Where state lives
- Local default: `terraform.tfstate` in working directory
- Local + workspaces: `terraform.tfstate.d/<workspace>/terraform.tfstate`
- Remote: in the configured backend (S3, HCP Terraform, etc.)

### Trap 15: Provider configuration
- Two DIFFERENT versions of same provider in one config = NOT allowed
- Multiple configurations of same provider via `alias` = ALLOWED
- Resources without `provider =` argument use the DEFAULT (no alias) provider

### Trap 16: Block order in files
- File order doesn't matter
- Resource block order doesn't matter
- Terraform builds dependency graph from references

### Trap 17: TF_LOG vs TF_LOG_PATH
- `TF_LOG` = sets level (TRACE, DEBUG, INFO, WARN, ERROR)
- `TF_LOG_PATH` = sets file location for logs
- They are SEPARATE variables; don't mix them up

### Trap 18: Provider config wins over env var
- `region = "us-west-2"` in provider block overrides `AWS_REGION=eu-west-1` env var
- Explicit config always beats environment variables

### Trap 19: HCP Terraform: cloud {} vs backend
- For HCP Terraform integration, use `cloud {}` inside terraform block
- NOT `backend "remote"` (legacy) and NOT `backend "hcp"` (doesn't exist)

### Trap 20: VCS-driven workflow PR behavior
- Opening a PR triggers a SPECULATIVE PLAN
- HCP Terraform posts results as PR comment
- Does NOT auto-apply; merge → real run with apply approval

### Trap 21: Drift handling on plan
- Manual change outside Terraform = drift
- `terraform plan` does NOT error
- Plan shows changes to REVERT the manual changes back to config

### Trap 22: Empty config + existing state
- If state has resources but config is empty → `terraform apply` DESTROYS them
- Desired state (empty) wins; current state (populated) gets reduced to match

### Trap 23: State file after destroy
- `terraform.tfstate` is NOT deleted after `terraform destroy`
- File remains, just contains no resources

### Trap 24: Plan acquires a lock
- Even though plan is "read-only", it acquires a state lock during refresh
- Use `-lock=false` to disable (dangerous)

### Trap 25: `terraform fmt` default scope
- Default: current directory only
- Use `-recursive` to format subdirectories

### Trap 26: HCP Terraform drift detection
- Detects drift via health checks
- Does NOT auto-remediate; only notifies
- You must manually run `terraform apply` to fix

### Trap 27: Removing a resource: which approach?
- Want to DESTROY it → delete the resource block from config + `terraform apply`
- Want to STOP managing but keep alive → use `removed` block + apply
- `removed` does NOT destroy infrastructure

### Trap 28: HCP Terraform feature names
- **Explorer** = search resources across all workspaces
- **Run Tasks** = call external tools between plan and apply
- **Run Triggers** = auto-queue run in B after A applies
- **Variable Sets** = reusable variables across workspaces/projects
- **Change Requests** = backlog of planned workspace changes
- **Health Checks** = drift detection + continuous validation
- **Projects** = folders that group workspaces
- **Agents** = self-hosted runners for private networks

### Trap 29: Lock file with init -upgrade
- `terraform init` (no flag) respects the lock file
- `terraform init -upgrade` updates lock file to newest allowed versions
- Even after changing version constraint, plain `init` won't upgrade

### Trap 30: Validation mechanisms (when does it run?)
- Variable `validation` block → during `validate` and `plan`
- `precondition` (lifecycle) → BEFORE create/update; HALTS on failure
- `postcondition` (lifecycle) → AFTER create/update; HALTS on failure
- `check` block → continuous; produces WARNINGS only, doesn't halt

---

## Validation Mechanisms — Side by Side

| Mechanism | Where | When | Behavior |
|---|---|---|---|
| Variable `validation` | Inside `variable` block | validate / plan | Halts on failure |
| `precondition` | `lifecycle` of resource/data | Before create/read | Halts on failure |
| `postcondition` | `lifecycle` of resource/data | After create/read | Halts on failure |
| `check` block | Top-level | Continuous monitoring | WARNS only, never halts |

**Use case:**
- Validate input → variable `validation`
- Validate prerequisite before resource → `precondition`
- Validate result after resource (e.g., on data source) → `postcondition`
- Monitor compliance without blocking → `check`

---

## moved / removed / import — Decision Matrix

| Scenario | Block to Use | Effect |
|---|---|---|
| Rename resource within same state | `moved` | State updated, no recreation |
| Move resource into/out of module (same state) | `moved` | State updated, no recreation |
| Stop managing, keep infra alive | `removed` | Removed from state, infra survives |
| Bring existing infra under management | `import` block | Added to state, infra unchanged |
| Cross-state migration | `removed` (source) + `import` (destination) | Ownership transferred |

**Import block syntax:**
```
import {
  to = aws_resource.name
  id = "real-resource-id"
}
```
Then run `terraform plan` + `apply` (NOT `terraform import` CLI).

**For `for_each` resources, need ONE import block per instance:**
```
import { to = aws_s3_bucket.x["dev"];     id = "..." }
import { to = aws_s3_bucket.x["staging"]; id = "..." }
```

---

## Plan Output Symbols

| Symbol | Meaning |
|---|---|
| `+` | Create |
| `-` | Destroy |
| `~` | Update in place |
| `-/+` | Destroy AND recreate |
| `<=` | Read (data source) |

---

## Commands That Modify State vs Read-Only

| Modifies State | Read-Only |
|---|---|
| `apply` | `plan` (acquires lock but doesn't write) |
| `destroy` | `validate` |
| `import` | `fmt` |
| `state mv`, `state rm`, `state push` | `graph` |
| `apply -refresh-only` | `show`, `state list`, `state show` |
| `taint`, `untaint` (deprecated) | `output` |

---

## Inspect State Commands

| Command | Output |
|---|---|
| `terraform show` | Full state file (verbose, all attributes) |
| `terraform state list` | List of resource addresses (concise) |
| `terraform state show <addr>` | Detailed attributes for one resource |
| `terraform output` | Output values only |
| `terraform output -json` | Output values as JSON |

---

## HCP Terraform — Quick Reference

### Three Workflows
1. **VCS-driven** — Git push triggers run; PR triggers speculative plan
2. **CLI-driven** — `terraform plan/apply` from local CLI runs REMOTELY
3. **API-driven** — REST API for custom CI/CD pipelines

### Three Execution Modes
| Mode | Plan/Apply runs on | State stored |
|---|---|---|
| Remote (default) | HCP Terraform infra | HCP Terraform |
| Local | Your machine | HCP Terraform |
| Agent | Self-hosted agent | HCP Terraform |

### When you connect to HCP Terraform
```
terraform {
  cloud {
    organization = "my-org"
    workspaces { name = "prod" }
  }
}
```
- Use `cloud {}`, NOT `backend "hcp"`, NOT `backend "remote"` (legacy)
- Run `terraform login` first to authenticate

### Key HCP Terraform Features
- **Workspace** — single config + state + variables + team access
- **Project** — folder grouping related workspaces
- **Variable Set** — reusable variables across workspaces/projects
- **Run Trigger** — auto-queue run in B after A applies
- **Run Task** — external tool integration between plan and apply
- **Sentinel / OPA** — policy as code (Plus/Enterprise)
- **Private Registry** — host private modules/providers (use `app.terraform.io/...`)
- **Explorer** — search resources across workspaces
- **Health Checks** — drift detection + continuous validation
- **Change Requests** — backlog of planned workspace changes

### VCS Workflow on PR
- PR opened → speculative plan runs
- Results posted as comment on PR
- Merge → real run; apply must be approved (unless auto-apply enabled)

### Cross-Workspace Data
```
data "tfe_outputs" "vpc" {
  organization = "my-org"
  workspace    = "networking"
}
# Use as: data.tfe_outputs.vpc.values.vpc_id
```

---

## HCP Terraform Tier Summary

| Feature | OSS | Free | Plus | Enterprise |
|---|---|---|---|---|
| State storage | Local | Cloud | Cloud | Self-hosted |
| Remote runs | No | Yes | Yes | Yes |
| VCS integration | No | Yes | Yes | Yes |
| Private registry | No | Yes | Yes | Yes |
| Run triggers | No | Yes | Yes | Yes |
| Run tasks | No | Yes | Yes | Yes |
| Sentinel / OPA | No | **No** | **Yes** | Yes |
| Cost estimation | No | No | Yes | Yes |
| SSO | No | No | Yes | Yes |
| Self-hosted agents | No | No | Yes | Yes |
| Audit logging | No | No | No | **Yes** |

**Key splits:**
- Sentinel + Cost estimation = Plus or Enterprise
- Audit logging = Enterprise ONLY
- Self-hosted install = Enterprise ONLY

---

## Module Data Flow (Critical!)

### Root → Child (passing IN)
- Root: `module "x" { var_name = value }`
- Child: must declare `variable "var_name" {}`
- Variables are SCOPED to the module — NO automatic inheritance

### Child → Root (getting OUT)
- Child: `output "name" { value = ... }`
- Root: access via `module.x.name`
- WITHOUT an output block in child, value is invisible to root

### Child → Child (peer modules)
- Cannot communicate directly
- Root acts as bridge: `module "b" { in = module.a.out }`

### Nested modules (grandchild)
- Each layer must explicitly bubble outputs up
- Cannot skip levels

---

## Function Cheatsheet (Memorize These)

### Strings
- `length(s)` / `upper(s)` / `lower(s)` / `title(s)`
- `format("Hi %s", name)` — printf-style
- `replace(s, old, new)` / `split(",", "a,b,c")` / `join(",", list)`
- `substr(s, start, len)` / `trimspace(s)`

### Numbers
- `max(...)` / `min(...)` / `abs(n)` / `ceil(n)` / `floor(n)`

### Collections
- `length(coll)` — works on lists/maps/strings
- `element(list, i)` — by index (handles wraparound)
- `lookup(map, key, default)` — safe map access
- `merge(map1, map2)` — for MAPS (later overrides earlier)
- `concat(list1, list2)` — for LISTS
- `flatten([[a,b],[c]])` — flatten one level
- `distinct(list)` / `sort(list)` / `reverse(list)`
- `keys(map)` / `values(map)` / `contains(list, val)`
- `toset(list)` / `tolist(set)` / `tomap(obj)`

### File / Template
- `file("path")` / `filebase64("path")`
- `templatefile("path", { vars })`

### Encoding
- `jsonencode/jsondecode` / `yamlencode/yamldecode` / `base64encode/base64decode`

### Network
- `cidrhost("10.0.0.0/16", 5)` → `"10.0.0.5"`
- `cidrsubnet("10.0.0.0/16", 8, 1)` → `"10.0.1.0/24"`

### Type Helpers
- `can(expr)` — boolean: did it succeed?
- `try(expr, fallback)` — returns fallback on error
- `coalesce(a, b, c)` — first non-null/non-empty
- `nonsensitive(value)` — strip sensitive marking

**Common confusion:**
- Combine MAPS → `merge()`
- Combine LISTS → `concat()`
- Convert list → set for `for_each` → `toset()`

---

## Variable Types Quick Reference

| Type | Syntax | Use |
|---|---|---|
| string | `"value"` | Single string |
| number | `42` | Numeric |
| bool | `true` | Boolean |
| list | `["a", "b"]` (square brackets) | Ordered collection |
| set | `toset([...])` | Unique unordered |
| map | `{ "k" = "v" }` (curly braces) | Key-value, dynamic keys |
| object | `{ name = "x", age = 5 }` | Fixed structure, mixed types |
| tuple | `[string, number, bool]` | Fixed-size, mixed types |

**Reference:**
- `var.list_name[1]` (zero-indexed)
- `var.map_name["key"]` (always brackets for maps with hyphens in keys)
- `var.map_name.key` (dot notation if key is valid identifier)

---

## Backend Cheatsheet

### S3 Backend (most common)
```
bucket         = required
key            = required (state file path)
region         = required
dynamodb_table = optional (for locking)
encrypt        = recommended
```

### Backends That Support Locking
s3 (with DynamoDB), azurerm, gcs, consul, HCP Terraform/TFE, pg

### Backends That DON'T Support Locking
local (no concurrent protection), http (depends)

### Backend Block Rules
- Only literals (no variables, no expressions, no resource refs)
- Use `-backend-config` flag or file for dynamic values
- Sensitive values: pass via env vars or `-backend-config` (NOT in code)
- Changing backend → run `terraform init` again (`-migrate-state` or `-reconfigure`)

---

## Provider Install Methods
1. **Terraform Public Registry** (default) — `registry.terraform.io`
2. **HCP Terraform Private Registry** — for org-internal providers
3. **Filesystem mirror** — local directory (air-gapped)
4. **Network mirror** — private HTTPS server

NOT install methods: GitHub releases, manual binary placement, source compilation.

---

## Common Workflows You Should Know

### Save plan, review, apply later
```
terraform plan -out=tfplan
# review with stakeholders
terraform apply tfplan
```

### Import existing resource (declarative)
```
1. Write resource block matching real settings
2. Add import block: import { to = ..., id = "..." }
3. terraform plan → terraform apply
```

### Migrate to remote backend
```
1. Add backend or cloud block
2. terraform init -migrate-state
3. Confirm migration prompt
```

### Detect drift
```
terraform plan -refresh-only         # see drift (no state change)
terraform apply -refresh-only        # accept drift into state
```

### Force resource recreation
```
terraform apply -replace=aws_instance.web
```

### Selective destroy (emergency only)
```
terraform destroy -target=aws_instance.web
```

### Connect CLI to HCP Terraform
```
1. Add cloud {} block to terraform block
2. terraform login (browser auth, stores token)
3. terraform init
4. Run plan/apply normally — runs go remote
```

---

## Resource Address Formats

| Pattern | Example |
|---|---|
| Resource | `aws_instance.web` |
| With count | `aws_instance.web[0]` |
| With for_each | `aws_instance.web["us-east-1"]` |
| In module | `module.network.aws_vpc.main` |
| Nested module | `module.app.module.db.aws_rds_instance.main` |
| Splat | `aws_instance.web[*].id` |
| Data source | `data.aws_ami.al2.id` |

---

## Lifecycle Block Quick Reference

```
create_before_destroy = true   # zero-downtime replacement
prevent_destroy       = true   # safety guard against destroy
ignore_changes        = [tags] # don't track drift on these attrs
replace_triggered_by  = [...]  # replace when another resource changes
precondition  { ... }          # check before applying
postcondition { ... }          # check after applying
```

---

## CLI Flag Quick Reference

| Flag | Purpose |
|---|---|
| `-auto-approve` | Skip confirmation |
| `-target=<addr>` | Specific resource only (use sparingly) |
| `-replace=<addr>` | Force replacement |
| `-refresh-only` | Update state without changing infra |
| `-refresh=false` | Skip state refresh during plan |
| `-input=false` | Disable interactive prompts (CI) |
| `-out=<file>` | Save plan to file |
| `-var "k=v"` | Set variable inline |
| `-var-file=<f>` | Load variables from file |
| `-parallelism=N` | Set concurrency (default 10) |
| `-upgrade` | Upgrade providers/modules to latest allowed |
| `-migrate-state` | Migrate state to new backend |
| `-reconfigure` | Reset backend, ignore existing state |
| `-backend=false` | Skip backend init |
| `-recursive` | (`fmt` only) include subdirs |
| `-check` | (`fmt` only) check without modifying |
| `-lock=false` | Disable state lock (dangerous) |

---

## Logging Levels (TF_LOG)

```
TRACE  → most verbose (every API call)
DEBUG  → detailed but readable
INFO   → high-level events
WARN   → warnings only
ERROR  → errors only
```

- `TF_LOG=TRACE` — set level
- `TF_LOG_PATH=/tmp/tf.log` — write to file (separate variable)
- Unset `TF_LOG` to disable (no `OFF` value exists)

---

## Sensitive Data — The Hard Truth

**Nothing prevents sensitive data from being in state EXCEPT ephemeral resources.**

| Approach | Hides from CLI | Out of State |
|---|---|---|
| `sensitive = true` on variable | Yes | NO |
| `sensitive = true` on output | Yes | NO |
| Vault data source | No | NO (still written to state) |
| `terraform.tfvars` (gitignored) | No | NO |
| **Ephemeral resource** | Yes | **YES** |
| **`password_wo` write-only attribute** | Yes | **YES** |

**For real secret protection:** ephemeral resources (TF 1.10+) + Vault + write-only attributes.

---

## IaC vs Configuration Management

| Concept | Tool Examples | Focus |
|---|---|---|
| IaC (provisioning) | Terraform, CloudFormation | Create/update/destroy infrastructure |
| Configuration management | Chef, Puppet, Ansible | Configure software on existing servers |

**Often used together:** Terraform provisions VMs, Ansible configures them.

---

## Top 10 Pre-Exam Anxieties (Reality Checks)

1. **"I don't remember every function"** — Know the common ones: length, lookup, merge, file, templatefile, cidrsubnet, toset.
2. **"I'll forget syntax"** — Multiple choice; recognition is enough.
3. **"HCP Terraform features confuse me"** — Focus on: cloud{} block, execution modes, run triggers vs run tasks, Sentinel, tier table.
4. **"What about provisioners?"** — Last resort feature. Know: local-exec, remote-exec, file, when=destroy, on_failure.
5. **"State management is complex"** — Three keys: plaintext (protect), use locking, never edit manually.
6. **"Modules trip me up"** — Two rules: (1) `module.<NAME>.<OUTPUT>`, (2) child must declare outputs.
7. **"Variables vs locals vs outputs?"** — Variables = inputs, Locals = computed, Outputs = exposed.
8. **"What if I don't know terminology?"** — Eliminate clearly wrong; remaining is usually right.
9. **"I haven't tried every command"** — Knowing what each does is enough.
10. **"What if I fail?"** — You can retake.

---

## Exam Day Checklist

### Night Before
- [ ] Skim this document
- [ ] Take one timed practice exam
- [ ] Review topics you missed
- [ ] Set out ID and computer
- [ ] Sleep 7-8 hours

### Morning Of
- [ ] Light breakfast
- [ ] Skim Top 40 Facts and Common Traps
- [ ] Quick check on HCP Terraform tier table
- [ ] No new topics — consolidate
- [ ] Arrive 15 min early (or boot 15 min before remote start)

### During Exam
- [ ] Read each question TWICE
- [ ] Eliminate obviously wrong answers first
- [ ] Watch for "always" / "never" / "all" — usually wrong
- [ ] If stuck, mark and move on
- [ ] Trust your prep — first instinct is often right

### Time Management
- 57 questions, 60 minutes = ~1 minute per question
- Don't dwell — flag and move on
- Save 5-10 minutes for flagged questions

---

## One-Page Summary Card

```
┌──────────────────────────────────────────────────────────┐
│ TERRAFORM EXAM - QUICK REFERENCE                         │
├──────────────────────────────────────────────────────────┤
│ Workflow:    Write → Init → Validate → Plan → Apply      │
│ State:       plaintext, lockable, NEVER commit           │
│ Lock file:   .terraform.lock.hcl — DO commit             │
│ Modules:     module.<name>.<output>, version optional    │
│ Backends:    S3+DynamoDB, HCP Cloud, local               │
│ Variables:   default<env<tfvars<auto<-var-file<-var      │
│ Sensitive:   redacts CLI only, NOT state                 │
│ Drift:       terraform plan -refresh-only                │
│ Import:      import block + plan + apply                 │
│ Move/rename: moved block (same state)                    │
│ Stop mgmt:   removed block (keeps infra)                 │
│ HCP exec:    Remote (default) / Local / Agent            │
│ HCP connect: cloud {} block + terraform login            │
│ Cross-WS:    tfe_outputs data source                     │
│ Sentinel:    Plus+ tier; advisory/soft/hard mandatory    │
│ Audit:       Enterprise only                             │
│ Run triggers: workspace B runs after A applies           │
│ Run tasks:   external tools between plan and apply       │
│ for_each:    needs map or set (use toset())              │
│ Plan symbols: +create -destroy ~update -/+replace        │
│ Provider wins: explicit config beats env vars            │
│ Validation:  precondition halts, check only warns        │
└──────────────────────────────────────────────────────────┘
```

---

## Final 20 Reminders

1. `cloud {}` block, NOT `backend "hcp"`
2. Variables are scoped to module — child cannot see parent's
3. `merge()` is for maps; `concat()` is for lists
4. `for_each` requires map/set — wrap lists with `toset()`
5. Plan acquires a lock even though it's "read-only"
6. `terraform fmt` defaults to current dir — use `-recursive` for subdirs
7. `validate` doesn't catch missing provider arguments — `plan` does
8. State file persists after destroy — file remains, just empty
9. Empty config + populated state = destroy ALL on next apply
10. `removed` block keeps infrastructure alive; delete block + apply destroys it
11. `moved` is for same state; cross-state migration uses `removed` + `import`
12. HCP Terraform Explorer = search resources across workspaces
13. Run triggers ≠ Run tasks (auto-queue vs external integration)
14. Each platform needs its own provider plugin
15. Import block needs ONE entry per `for_each` instance
16. `~> 5.0.0` allows only 5.0.x patches; `~> 5.0` allows 5.x minor releases
17. Module `version` works only with registry sources
18. Drift on plan = revert changes shown, no error thrown
19. Sentinel is Plus/Enterprise only; audit logging is Enterprise only
20. Provider block region wins over `AWS_REGION` env variable

---

## You're Ready

If you've worked through:
- The detailed cheatsheet
- Module data flow guide
- All 3 sets of practice questions (150 total)
- 6 mock exams (last two scored 79% and 86%)
- This last-minute prep

…you have **more than enough** preparation.

Trust yourself, breathe, and go ace it.

**Good luck!**
