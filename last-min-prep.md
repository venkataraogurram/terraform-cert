# Terraform Associate — Last Minute Prep

**Read this the night before and morning of the exam.**

This is a high-density review. No code examples — just facts you must recall instantly.

---

## 60-Second Mental Map

Terraform = Declarative IaC tool by HashiCorp
Workflow = Write → Init → Plan → Apply → Destroy
State = Maps config to real resources (CRITICAL — protect it)
Modules = Reusable groups of resources
Providers = Plugins that talk to APIs (AWS, GCP, Azure, etc.)
HCP Terraform = HashiCorp's managed SaaS (state + remote runs + governance)

---

## Top 30 Facts to Memorize

1. `terraform init` — required first; downloads providers + modules + sets up backend
2. `terraform validate` — offline syntax/type check (no API calls)
3. `terraform fmt` — canonical formatting
4. `terraform plan` — shows changes; `-out=file.tfplan` to save
5. `terraform apply file.tfplan` — applies a saved plan exactly
6. `terraform destroy` — destroys all managed infrastructure (prompts unless `-auto-approve`)
7. `.terraform/` — local cache (providers + modules), DO NOT commit
8. `.terraform.lock.hcl` — provider version lock, DO commit
9. `terraform.tfstate` — state file, DO NOT commit
10. State is plaintext — `sensitive = true` only redacts CLI output, not state
11. State locking prevents concurrent runs (use DynamoDB with S3 backend)
12. `terraform force-unlock <ID>` — manually release stuck lock
13. `terraform import` — bring existing resource into state (must have matching config)
14. `import` block (1.5+) — declarative import in code
15. `moved` block (1.1+) — rename/move resources without recreation
16. `removed` block (1.7+) — remove from state without destroying
17. `count` — integer index; `for_each` — string key; for_each preferred
18. `terraform.workspace` — current workspace name as string
19. OSS workspaces ≠ HCP Terraform workspaces (different concepts!)
20. `TF_VAR_<name>` — env var prefix for setting variables
21. `TF_LOG=TRACE` (or DEBUG/INFO/WARN/ERROR) — enable logging
22. `TF_LOG_PATH=/path` — write logs to file
23. Variable precedence: defaults < env < tfvars < auto.tfvars < -var-file < -var
24. Module reference: `module.<NAME>.<OUTPUT>` — only way to access child output
25. Module `version` — optional but RECOMMENDED; only works with registry sources
26. `~> 5.0` means `>= 5.0, < 6.0` (allows minor); `~> 5.0.0` means `>= 5.0.0, < 5.1.0`
27. Sentinel — Plus/Enterprise only; enforcement: advisory / soft-mandatory / hard-mandatory
28. Audit logging — Enterprise ONLY
29. Run trigger — auto-queues run in workspace B after A applies successfully
30. CLI-driven workflow with HCP Terraform — runs execute REMOTELY by default

---

## Most Common Exam Traps

### Trap 1: Module output access
**Wrong**: `output.module.x`, `var.module.x`, `module.x.resource.attr`
**Right**: `module.<NAME>.<OUTPUT_NAME>` — and only if child declares the output

### Trap 2: HCP Terraform CLI-driven mode
**Wrong**: "runs on local machine"
**Right**: runs on HCP Terraform infrastructure, output streamed back

### Trap 3: HCP Terraform Local execution mode
**Wrong**: "blocks all CLI usage"
**Right**: only stores state remotely; plans/applies run locally

### Trap 4: `sensitive = true`
**Wrong**: "encrypts in state"
**Right**: only redacts from CLI output; still plaintext in state

### Trap 5: `lifecycle.prevent_destroy`
**Wrong**: "prevents updates"
**Right**: errors if a plan would destroy; doesn't stop in-place updates

### Trap 6: `count` vs `for_each` deletion behavior
**Wrong**: "they behave the same"
**Right**: removing from middle of `count` shifts indices and may destroy/recreate; `for_each` only affects removed item

### Trap 7: OSS workspaces for prod isolation
**Wrong**: "use workspaces for dev/staging/prod"
**Right**: workspaces share config; for true isolation use separate root modules + backends

### Trap 8: `terraform refresh`
**Wrong**: "use this to detect drift"
**Right**: deprecated; use `terraform plan -refresh-only` (read) or `terraform apply -refresh-only` (write)

### Trap 9: `terraform taint`
**Wrong**: "mark for replacement"
**Right**: deprecated; use `terraform apply -replace=<address>`

### Trap 10: Module `version` argument
**Wrong**: "required for registry modules"
**Right**: optional but recommended; without it, latest version is downloaded

### Trap 11: Backend reconfigure flags
- `-migrate-state` = copy existing state to new backend
- `-reconfigure` = ignore existing state, fresh start in new backend
- `-backend=false` = skip backend init entirely

### Trap 12: terraform validate vs plan
- `validate` — offline, syntax/types only, no API calls
- `plan` — online, checks against real provider state

### Trap 13: Where state lives
- Local backend default: `terraform.tfstate` in working directory
- Workspaces (local): `terraform.tfstate.d/<workspace>/terraform.tfstate`
- Remote: in the configured backend (S3, HCP Terraform, etc.)

### Trap 14: Cannot mix-and-match
- You can't have two DIFFERENT versions of the same provider in one config
- You CAN have multiple configurations of the same provider via aliases

### Trap 15: Block order in files
- File order doesn't matter
- Resource block order in files doesn't matter
- Terraform builds dependency graph from references

---

## Exam Objectives Checklist

### Objective 1: IaC Concepts
- [ ] What is IaC? (declarative vs imperative)
- [ ] Advantages: version control, repeatability, collaboration
- [ ] Multi-cloud benefits with single tool

### Objective 2: Terraform Purpose
- [ ] Plugin architecture (Core + Providers)
- [ ] How providers are installed and versioned
- [ ] State purpose: map, metadata, performance

### Objective 3: Workflow
- [ ] init / plan / apply / destroy
- [ ] validate / fmt / show / output
- [ ] When to re-run init (new providers/modules/versions/backend)

### Objective 4: Configuration
- [ ] Resource vs data blocks
- [ ] Variable types (string, number, bool, list, set, map, object, tuple)
- [ ] Variable validation
- [ ] Implicit vs explicit dependencies
- [ ] Built-in functions (length, lookup, merge, file, etc.)
- [ ] Sensitive data handling

### Objective 5: Modules
- [ ] Sources (local, registry, git, s3, etc.)
- [ ] Inputs (variables) and outputs
- [ ] Version constraints
- [ ] Public vs private registry

### Objective 6: State Management
- [ ] Local vs remote backends
- [ ] State locking (DynamoDB for S3)
- [ ] Backend configuration block
- [ ] Drift detection (refresh-only)

### Objective 7: Maintaining Infrastructure
- [ ] Import existing resources (CLI + import blocks)
- [ ] Inspect state (state list, state show)
- [ ] Verbose logging (TF_LOG)

### Objective 8: HCP Terraform
- [ ] Workspaces (organization → project → workspace)
- [ ] Execution modes (Remote, Local, Agent)
- [ ] VCS-driven, CLI-driven, API-driven workflows
- [ ] Run triggers
- [ ] Sentinel policies
- [ ] Private registry

---

## Function Cheatsheet (Memorize These)

### Strings
- `length(s)` — string length
- `upper(s)` / `lower(s)` / `title(s)`
- `format("Hi %s", name)` — printf-style
- `replace(s, old, new)` — replace substring
- `split(",", "a,b,c")` — to list
- `join(",", list)` — to string
- `substr(s, start, length)`
- `trimspace(s)` / `trim(s, cutset)`

### Numbers
- `max(...)` / `min(...)`
- `abs(n)` / `ceil(n)` / `floor(n)`

### Collections
- `length(list)` — also works on maps
- `element(list, i)` — by index (handles wraparound)
- `lookup(map, key, default)` — safe map access
- `merge(map1, map2)` — later overrides earlier
- `concat(list1, list2)` — join lists
- `flatten([[a,b],[c]])` — flatten one level
- `distinct(list)` — remove duplicates
- `sort(list)` — sort strings
- `reverse(list)` — reverse order
- `keys(map)` / `values(map)`
- `contains(list, value)` — boolean
- `toset(list)` / `tolist(set)` / `tomap(obj)`

### File / Template
- `file("path")` — read as string
- `filebase64("path")` — read as base64
- `templatefile("path", { vars })` — render template

### Encoding
- `jsonencode(obj)` / `jsondecode(str)`
- `yamlencode(obj)` / `yamldecode(str)`
- `base64encode(s)` / `base64decode(s)`

### Network
- `cidrhost("10.0.0.0/16", 5)` → `"10.0.0.5"`
- `cidrsubnet("10.0.0.0/16", 8, 1)` → `"10.0.1.0/24"`

### Type Helpers
- `can(expr)` — boolean: did expression succeed?
- `try(expr, fallback)` — return fallback on error
- `coalesce(a, b, c)` — first non-null/non-empty
- `nonsensitive(value)` — strip sensitive marking

---

## Backend Cheatsheet

### S3 Backend (most common)
```
bucket         = required
key            = required (state file path)
region         = required
dynamodb_table = optional (for locking)
encrypt        = recommended
kms_key_id     = optional (custom KMS)
role_arn       = optional (assume role)
```

### Backends That Support Locking
- s3 (with DynamoDB)
- azurerm (built-in)
- gcs (built-in)
- consul
- HCP Terraform / TFE
- pg (PostgreSQL)

### Backends That DON'T Support Locking
- local (no concurrent protection)
- http (depends on implementation)

---

## HCP Terraform Tier Summary

| Feature | OSS | Free | Plus | Enterprise |
|---|---|---|---|---|
| State storage | Local | Cloud | Cloud | Self-hosted |
| Remote runs | No | Yes | Yes | Yes |
| VCS integration | No | Yes | Yes | Yes |
| Private registry | No | Yes | Yes | Yes |
| Run triggers | No | Yes | Yes | Yes |
| Sentinel | No | **No** | **Yes** | Yes |
| Cost estimation | No | No | Yes | Yes |
| SSO | No | No | Yes | Yes |
| Self-hosted agents | No | No | Yes | Yes |
| Audit logging | No | No | No | **Yes** |

**Key splits**:
- Sentinel + Cost estimation = Plus or Enterprise
- Audit logging = Enterprise ONLY
- Self-hosted install = Enterprise ONLY

---

## Common Workflows You Should Know

### Save plan, review, apply later
```
terraform plan -out=tfplan
# review with stakeholders
terraform apply tfplan
```

### Import existing resource
```
1. Write resource block matching real settings
2. terraform import <address> <real-id>
3. terraform plan (verify no diff)
```

### Migrate to remote backend
```
1. Add backend config block
2. terraform init -migrate-state
3. Confirm migration prompt
```

### Switch HCP Terraform workspaces
```
terraform workspace list
terraform workspace select <name>
```

### Detect drift
```
terraform plan -refresh-only         # see drift
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

---

## Lifecycle Block Quick Reference

```
create_before_destroy = true   # zero-downtime replacement
prevent_destroy       = true   # safety guard against destroy
ignore_changes        = [tags] # don't track drift on these attrs
replace_triggered_by  = [...]  # replace when another resource changes
precondition  { condition, error_message }  # check before applying
postcondition { condition, error_message }  # check after applying
```

---

## CLI Flag Quick Reference

| Flag | Purpose |
|---|---|
| `-auto-approve` | Skip confirmation prompt |
| `-target=<addr>` | Operate on specific resource (use sparingly) |
| `-replace=<addr>` | Force replacement (replaces taint) |
| `-refresh-only` | Update state without changing infra |
| `-refresh=false` | Skip state refresh during plan |
| `-input=false` | Disable interactive prompts (CI) |
| `-out=<file>` | Save plan to file |
| `-var "k=v"` | Set variable inline |
| `-var-file=<f>` | Load variables from file |
| `-parallelism=N` | Set concurrency (default 10) |
| `-upgrade` | Upgrade providers/modules to latest allowed |
| `-migrate-state` | Migrate state to new backend |
| `-reconfigure` | Reset backend config, ignore existing state |
| `-backend=false` | Skip backend initialization |

---

## Logging Levels (TF_LOG)

```
TRACE  → most verbose (every API call)
DEBUG  → detailed but readable
INFO   → high-level events
WARN   → warnings only
ERROR  → errors only
```

Set `TF_LOG_PATH=/tmp/tf.log` to file output.

---

## Top 10 Pre-Exam Anxieties (and Reality Checks)

1. **"I don't remember every function"** — You don't need to. Know the common ones: length, lookup, merge, file, templatefile, cidrsubnet.

2. **"I'll forget syntax on the exam"** — The exam is multiple choice. You'll recognize correct syntax even if you can't write it from scratch.

3. **"HCP Terraform features confuse me"** — Focus on: execution modes, run triggers, Sentinel, tier differences. That covers most questions.

4. **"What about provisioners?"** — They're a "last resort" feature. Know the basics: local-exec, remote-exec, file. Know when=destroy, on_failure=continue.

5. **"State management is so complex"** — Three keys: state contains plaintext (protect it), use locking, never edit manually.

6. **"Modules trip me up"** — Two rules: (1) `module.<NAME>.<OUTPUT>` to access child outputs, (2) child must declare output blocks.

7. **"Variables vs locals vs outputs?"** — Variables = inputs, Locals = computed, Outputs = exposed values. That's it.

8. **"What if I don't know terminology?"** — Read the question carefully; eliminate clearly wrong answers; the remaining choice is usually correct.

9. **"I haven't tried every command"** — That's fine. Knowing what each command does is sufficient for the exam.

10. **"What if I fail?"** — You can retake. Most people pass on first or second attempt with proper prep.

---

## Exam Day Checklist

### Night Before
- [ ] Skim this document
- [ ] Take 1 timed practice exam (Set 2 or 3)
- [ ] Review any topics you missed
- [ ] Set out ID and computer (if remote)
- [ ] Sleep 7-8 hours

### Morning Of
- [ ] Light breakfast
- [ ] Skim Top 30 Facts and Common Traps sections
- [ ] Quick check on HCP Terraform tier table
- [ ] No new topics — consolidate what you know
- [ ] Arrive 15 min early (or boot 15 min before remote start)

### During Exam
- [ ] Read each question TWICE
- [ ] Eliminate obviously wrong answers first
- [ ] Watch for "always" / "never" / "all" — usually wrong
- [ ] If stuck, mark and move on; come back later
- [ ] Trust your prep — first instinct is often right

### Time Management
- 57 questions, 60 minutes = ~1 minute per question
- Don't dwell — flag and move on if stuck
- Save 5-10 minutes at the end for flagged questions

---

## One-Page Summary Card

```
┌──────────────────────────────────────────────────────────┐
│ TERRAFORM EXAM - QUICK REFERENCE                          │
├──────────────────────────────────────────────────────────┤
│ Workflow:  Write → Init → Validate → Plan → Apply        │
│ State:     plaintext, lockable, never commit             │
│ Modules:   module.<name>.<output>, version optional      │
│ Backends:  S3+DynamoDB, HCP Cloud, local                 │
│ Variables: defaults < env < tfvars < auto < -var-file < -var │
│ Sensitive: redacts CLI only, NOT state                   │
│ Drift:     terraform plan -refresh-only                  │
│ Import:    write block + import block + apply            │
│ HCP exec:  Remote (default) / Local / Agent              │
│ Sentinel:  Plus+ tier; advisory/soft/hard mandatory      │
│ Audit:     Enterprise only                               │
│ Triggers:  workspace B runs after workspace A applies    │
└──────────────────────────────────────────────────────────┘
```

---

## You're Ready

If you've worked through:
- The detailed cheatsheet
- Module data flow guide
- All 3 sets of practice questions (150 total)
- This last-minute prep

…you have **more than enough** preparation. Trust yourself, breathe, and go ace it.

**Good luck!**
