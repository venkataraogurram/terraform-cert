# Terraform Associate — Practice Questions

100 exam-style questions covering all objectives. Answers and explanations at the bottom of each section.

---

## Section 1: Core Workflow & CLI (Q1–Q15)

**Q1.** Which command initializes a working directory and downloads required providers?
- A) `terraform plan`
- B) `terraform init`
- C) `terraform validate`
- D) `terraform setup`

**Q2.** What does `terraform validate` do?
- A) Validates syntax against the cloud API
- B) Validates HCL syntax and internal consistency without API calls
- C) Runs a plan and validates the diff
- D) Checks state file integrity

**Q3.** Which command formats `.tf` files to canonical style?
- A) `terraform format`
- B) `terraform style`
- C) `terraform fmt`
- D) `terraform lint`

**Q4.** What is the modern replacement for `terraform taint`?
- A) `terraform untaint`
- B) `terraform apply -replace=ADDR`
- C) `terraform state rm`
- D) `terraform refresh`

**Q5.** What does `terraform plan -out=plan.tfplan` do?
- A) Outputs the plan to stdout in JSON
- B) Saves the plan to a file for later apply
- C) Skips the refresh step
- D) Validates the plan against policy

**Q6.** Which flag skips the refresh step during plan?
- A) `-no-refresh`
- B) `-refresh=false`
- C) `-skip-refresh`
- D) `-static`

**Q7.** What does `-auto-approve` do?
- A) Auto-creates a plan file
- B) Skips interactive approval prompts
- C) Auto-formats files
- D) Approves any plan changes regardless of risk

**Q8.** What is the default parallelism for `terraform apply`?
- A) 1
- B) 5
- C) 10
- D) 20

**Q9.** Which command shows a human-readable summary of state?
- A) `terraform state show`
- B) `terraform show`
- C) `terraform output`
- D) `terraform inspect`

**Q10.** Which command produces a DOT-format dependency graph?
- A) `terraform plan -graph`
- B) `terraform graph`
- C) `terraform show -graph`
- D) `terraform dag`

**Q11.** What does `terraform destroy` do at its core?
- A) Deletes the state file
- B) Runs `terraform apply -destroy`
- C) Removes all `.tf` files
- D) Removes resources from state without affecting infrastructure

**Q12.** Which command logs you in to Terraform Cloud?
- A) `terraform auth`
- B) `terraform login`
- C) `terraform connect`
- D) `tfc login`

**Q13.** When is `terraform init` required?
- A) On first use of a configuration
- B) After changing backend configuration
- C) After changing provider versions
- D) All of the above

**Q14.** What does `-target` do, and how should you use it?
- A) Routine deployments
- B) Targets a specific resource for operations; intended for exceptional/recovery use
- C) Sets the target environment
- D) Selects a workspace

**Q15.** Which file should be committed to version control?
- A) `terraform.tfstate`
- B) `.terraform/`
- C) `.terraform.lock.hcl`
- D) `terraform.tfvars` containing secrets

### Answers 1–15
1. **B** — `terraform init` initializes the directory.
2. **B** — Static syntax/consistency check, no API calls.
3. **C** — `terraform fmt`.
4. **B** — `apply -replace` is the modern way; `taint` is deprecated.
5. **B** — Saves plan for later apply.
6. **B** — `-refresh=false`.
7. **B** — Skips interactive approval.
8. **C** — Default is 10.
9. **B** — `terraform show`.
10. **B** — `terraform graph`.
11. **B** — `destroy` is an alias of `apply -destroy`.
12. **B** — `terraform login`.
13. **D** — All of the above require `init`.
14. **B** — Exceptional use only.
15. **C** — Lock file should be committed; state and secrets should not.

---

## Section 2: Variables & Outputs (Q16–Q30)

**Q16.** Which has the highest precedence for variable values?
- A) `terraform.tfvars`
- B) `TF_VAR_` env vars
- C) `-var` CLI flag
- D) `*.auto.tfvars`

**Q17.** Which file is auto-loaded?
- A) `prod.tfvars`
- B) `variables.tfvars`
- C) `dev.auto.tfvars`
- D) `env.json`

**Q18.** How do you set a variable named `region` via env var?
- A) `TERRAFORM_region`
- B) `TF_region`
- C) `TF_VAR_region`
- D) `TF_VARIABLE_region`

**Q19.** When multiple `*.auto.tfvars` files exist, in what order are they loaded?
- A) Order in directory listing
- B) Alphabetical
- C) Reverse alphabetical
- D) Random

**Q20.** What does `sensitive = true` on an output do?
- A) Encrypts the value in state
- B) Hides the value from CLI output
- C) Removes it from state
- D) Prevents reading from other modules

**Q21.** What happens if a variable has no default and no value is provided?
- A) Terraform uses an empty string
- B) Terraform fails immediately
- C) Terraform prompts interactively (unless `-input=false`)
- D) Terraform uses null

**Q22.** Which is a valid variable type constraint?
- A) `string`
- B) `list(number)`
- C) `map(string)`
- D) `object({name=string, port=number})`
- E) All of the above

**Q23.** What does the `validation` block do in a variable?
- A) Validates against cloud API
- B) Validates input value with custom condition
- C) Validates type only
- D) Performs runtime checks at apply

**Q24.** Where can outputs be referenced?
- A) Only inside the same module
- B) Only by parent modules calling this module
- C) Inside any resource's `provisioner`
- D) From any module via `module.NAME.OUTPUT`

**Q25.** What is `nullable = false` for in a variable?
- A) Disallows the variable from being unset
- B) Disallows null values
- C) Sets default to empty string
- D) Same as `sensitive = true`

**Q26.** Which file is loaded second in the precedence chain (after defaults and env vars)?
- A) `*.auto.tfvars`
- B) `terraform.tfvars`
- C) `-var-file`
- D) `-var`

**Q27.** What's the result of running `terraform plan -var "count=3"` with `terraform.tfvars` setting `count=5`?
- A) `count = 5`
- B) `count = 3`
- C) Error
- D) `count = 8`

**Q28.** Can `TF_VAR_` env vars set complex types like maps or lists?
- A) No, only strings
- B) Yes, using HCL or JSON syntax
- C) Only with a special flag
- D) Only via `-var-file`

**Q29.** What does `marked` mean for a sensitive value at runtime?
- A) It is encrypted
- B) Terraform tracks it through expressions to redact display
- C) It is moved to a vault
- D) It cannot be used in resources

**Q30.** Which is true about outputs in modules?
- A) Outputs are private by default
- B) Outputs must be explicitly exposed via `output` blocks
- C) Outputs are inherited automatically
- D) Outputs can only be primitives

### Answers 16–30
16. **C** — `-var` CLI flag.
17. **C** — `*.auto.tfvars` is auto-loaded; `prod.tfvars` is not.
18. **C** — `TF_VAR_<name>`.
19. **B** — Alphabetical.
20. **B** — Hides from CLI output (state still has plaintext).
21. **C** — Interactive prompt unless `-input=false`.
22. **E** — All listed are valid.
23. **B** — Custom validation condition.
24. **D** — Via `module.NAME.OUTPUT` (when output is declared).
25. **B** — Disallows null.
26. **B** — `terraform.tfvars` after env vars.
27. **B** — CLI `-var` wins over `terraform.tfvars`.
28. **B** — HCL/JSON syntax in the env var value.
29. **B** — Tracking through expressions for redaction.
30. **B** — Must be declared as `output` blocks.

---

## Section 3: State & Drift (Q31–Q45)

**Q31.** Which file contains the previous state version?
- A) `terraform.tfstate.old`
- B) `terraform.tfstate.backup`
- C) `terraform.tfstate.prev`
- D) `.terraform/state.bak`

**Q32.** What does `terraform apply -refresh-only` do?
- A) Refreshes config only
- B) Updates state to match real infrastructure, no infra changes
- C) Plans without refresh
- D) Recreates the state file

**Q33.** What is **drift** in Terraform?
- A) Terraform version mismatch
- B) Real infrastructure differs from state
- C) Lock file outdated
- D) Provider plugin issue

**Q34.** Which command brings an existing AWS resource into Terraform state?
- A) `terraform adopt`
- B) `terraform import`
- C) `terraform state add`
- D) `terraform state push`

**Q35.** What's the recommended way to handle out-of-band changes you want to KEEP?
- A) Run `terraform apply` to revert them
- B) Run `terraform apply -refresh-only` and update config (or use `ignore_changes`)
- C) Delete the state file
- D) Run `terraform destroy`

**Q36.** Is `terraform refresh` (standalone) recommended?
- A) Yes, primary way to handle drift
- B) Yes, when state is corrupt
- C) No, deprecated — use `apply -refresh-only`
- D) Yes, but only with `-target`

**Q37.** What exit code does `terraform plan -detailed-exitcode` return when changes are detected?
- A) 0
- B) 1
- C) 2
- D) 3

**Q38.** Which command moves a resource within state (e.g., for a refactor)?
- A) `terraform state move`
- B) `terraform state mv`
- C) `terraform state rename`
- D) `terraform mv`

**Q39.** What does `terraform state rm aws_instance.web` do?
- A) Destroys the resource
- B) Removes the resource from state without affecting actual infrastructure
- C) Moves it to trash
- D) Marks it as tainted

**Q40.** What recovers from `errored.tfstate`?
- A) `terraform state push errored.tfstate`
- B) Rename it to `terraform.tfstate`
- C) Re-run `terraform apply`
- D) `terraform recover`

**Q41.** When is `terraform.tfstate.backup` updated?
- A) Every plan
- B) Before every state write (apply)
- C) Manually only
- D) Once per workspace

**Q42.** Which is a benefit of S3 versioning for state files?
- A) Concurrent writes
- B) Recovery to any prior version
- C) Encryption
- D) State locking

**Q43.** Which AWS service is paired with S3 for **state locking**?
- A) ELB
- B) DynamoDB
- C) Lambda
- D) SNS

**Q44.** What does `-lock=false` do?
- A) Disables state encryption
- B) Disables state locking (risky)
- C) Marks state as read-only
- D) Removes existing locks

**Q45.** What's true about `terraform.tfstate` and Git?
- A) Always commit it for traceability
- B) Never commit it; use remote backends
- C) Only commit `.tfstate.backup`
- D) Commit only after redacting secrets

### Answers 31–45
31. **B** — `terraform.tfstate.backup`.
32. **B** — State to reality, no infra changes.
33. **B** — Real differs from state.
34. **B** — `terraform import`.
35. **B** — `apply -refresh-only` and update config.
36. **C** — Deprecated standalone.
37. **C** — Exit 2 = changes.
38. **B** — `terraform state mv`.
39. **B** — Removes from state, infra untouched.
40. **A** — `state push errored.tfstate`.
41. **B** — Before each state write.
42. **B** — Recover any prior version.
43. **B** — DynamoDB.
44. **B** — Disables locking.
45. **B** — Never commit; use remote backend.

---

## Section 4: Providers & Backends (Q46–Q60)

**Q46.** What does `~> 3.5` mean as a version constraint?
- A) `>= 3.5, < 3.6`
- B) `>= 3.5, < 4.0`
- C) Exactly 3.5
- D) `> 3.5`

**Q47.** What does `~> 3.5.2` allow?
- A) `>= 3.5.2, < 3.6.0`
- B) `>= 3.5.2, < 4.0.0`
- C) Exactly 3.5.2
- D) Any 3.x

**Q48.** What's the purpose of `.terraform.lock.hcl`?
- A) Locks state file
- B) Locks workspace
- C) Locks provider versions and checksums
- D) Locks module versions

**Q49.** How do you upgrade providers within constraints?
- A) `terraform init -upgrade`
- B) `terraform upgrade`
- C) `terraform providers update`
- D) Delete the lock file

**Q50.** Which is a valid provider source address format?
- A) `aws`
- B) `hashicorp/aws`
- C) `registry.terraform.io/hashicorp/aws`
- D) Both B and C

**Q51.** Which backend is **enhanced**?
- A) `s3`
- B) `local`
- C) `remote` (Terraform Cloud)
- D) `gcs`

**Q52.** How do you configure a second AWS provider for `us-west-2`?
- A) Two `provider` blocks, one with `alias = "west"`
- B) Two separate root modules
- C) `provider.aws.west`
- D) You can't

**Q53.** What does `terraform init -migrate-state` do?
- A) Move state to a new backend
- B) Encrypt state
- C) Generate a migration script
- D) Reset workspace state

**Q54.** Which is required to switch backend without copying existing state?
- A) `terraform init -reconfigure`
- B) `terraform init -migrate-state`
- C) `terraform init -upgrade`
- D) Delete `.terraform/`

**Q55.** Where is provider plugin code stored after `init`?
- A) `terraform.tfstate`
- B) `.terraform/providers/`
- C) `~/.terraform/`
- D) `/usr/local/lib/terraform/`

**Q56.** Which backends support state locking?
- A) Only S3
- B) Most remote backends (s3 with DynamoDB, azurerm, gcs, consul, …)
- C) Only Terraform Cloud
- D) None

**Q57.** Why use multiple provider configurations?
- A) Multi-region or multi-account deployments
- B) Different credentials for different resources
- C) Both A and B
- D) Provider parallelism

**Q58.** Where do credentials for AWS provider come from?
- A) Only from `provider` block
- B) From the standard AWS chain (env, shared credentials, IAM role, etc.)
- C) Only Terraform Cloud
- D) Hard-coded in modules

**Q59.** What does `required_version` do?
- A) Pins required Terraform CLI version
- B) Pins required provider version
- C) Pins required module version
- D) Pins required state file version

**Q60.** Which command shows currently installed providers and versions?
- A) `terraform providers`
- B) `terraform list providers`
- C) `terraform version`
- D) `terraform plugins`

### Answers 46–60
46. **B** — `~>` with two segments allows minor.
47. **A** — Three segments allows patch only.
48. **C** — Provider lock file.
49. **A** — `terraform init -upgrade`.
50. **D** — Both B (shorthand) and C (full) work.
51. **C** — `remote` is enhanced (remote ops).
52. **A** — `alias` on a second provider block.
53. **A** — Migrate state to new backend.
54. **A** — `-reconfigure` ignores existing state.
55. **B** — `.terraform/providers/`.
56. **B** — Most remote backends support locking.
57. **C** — Multi-region/account or different creds.
58. **B** — Standard AWS provider chain.
59. **A** — Required Terraform CLI version.
60. **A** — `terraform providers`.

---

## Section 5: Resources, Modules & Expressions (Q61–Q80)

**Q61.** Which loop construct gives stable identity if list order changes?
- A) `count`
- B) `for_each`
- C) Both
- D) Neither

**Q62.** What does this expression return: `flatten([["a"],[],["b","c"]])`?
- A) `[["a"],["b","c"]]`
- B) `["a","b","c"]`
- C) `["a",null,"b","c"]`
- D) Error

**Q63.** What does `for_each = toset(["a","b"])` do?
- A) Creates a list
- B) Lets you iterate over distinct values with `each.key`/`each.value`
- C) Creates one resource
- D) Errors

**Q64.** When is `dynamic` block used?
- A) For iterating top-level resources
- B) For programmatically generating nested blocks
- C) For modules
- D) For data sources

**Q65.** What does `replace_triggered_by = [aws_security_group.web]` do?
- A) Replaces the SG when the resource changes
- B) Replaces the resource when the SG is replaced
- C) Updates the resource in place when SG changes
- D) Triggers a refresh

**Q66.** Which `lifecycle` argument prevents accidental destroy?
- A) `protect`
- B) `prevent_destroy`
- C) `no_destroy`
- D) `lock`

**Q67.** What does `create_before_destroy = true` enable?
- A) Faster destroys
- B) Zero-downtime replacement (new resource created before old destroyed)
- C) Destroy-only mode
- D) Skip the destroy phase

**Q68.** What does `ignore_changes = [tags]` do?
- A) Removes tags from the resource
- B) Ignores drift on `tags` attribute during plan
- C) Prevents tag updates from CLI
- D) Hides tags from output

**Q69.** Which is a valid module source?
- A) `./modules/vpc`
- B) `terraform-aws-modules/vpc/aws`
- C) `git::https://github.com/x/y.git//modules/vpc?ref=v1.0`
- D) All of the above

**Q70.** How do you reference an output from a module named `vpc`?
- A) `vpc.output.id`
- B) `module.vpc.id`
- C) `module.vpc.output.id`
- D) `output.vpc.id`

**Q71.** What's the result of `merge({a=1,b=2},{b=3,c=4})`?
- A) `{a=1,b=2,c=4}`
- B) `{a=1,b=3,c=4}`
- C) Error
- D) `{a=1,b=2,b=3,c=4}`

**Q72.** What does `lookup(map, "key", "default")` return?
- A) Always `"default"`
- B) The value at `"key"`, or `"default"` if missing
- C) The map keys
- D) Error if key missing

**Q73.** What does `try(local.x.y.z, "fallback")` do?
- A) Always returns `"fallback"`
- B) Returns the expression value, or `"fallback"` if it errors
- C) Tries to evaluate `z`
- D) Errors if `local.x` is null

**Q74.** What does `cidrsubnet("10.0.0.0/16", 8, 1)` return?
- A) `"10.0.1.0/24"`
- B) `"10.0.0.0/24"`
- C) `"10.1.0.0/24"`
- D) Error

**Q75.** Which expression filters a list?
- A) `[for s in list : s if s != ""]`
- B) `filter(list, lambda)`
- C) `select(list)`
- D) `list.filter()`

**Q76.** What does `splat`, e.g. `aws_instance.web[*].id`, return?
- A) The first instance's ID
- B) A list of all instances' IDs
- C) The state of all instances
- D) Error if multiple instances exist

**Q77.** Which is true about `count` and removing an item from the middle of the list?
- A) Only that resource is destroyed
- B) Resources after that index are also affected (shifted)
- C) No effect
- D) Error

**Q78.** What's the conventional structure of a module?
- A) `main.tf`, `variables.tf`, `outputs.tf`
- B) `module.tf`
- C) `index.tf`
- D) `manifest.tf`

**Q79.** Which is the Terraform Public Module Registry URL pattern?
- A) `<provider>/<module>`
- B) `<NAMESPACE>/<NAME>/<PROVIDER>`
- C) `<owner>/<repo>`
- D) `<region>/<module>`

**Q80.** Can a module call other modules?
- A) No
- B) Yes, modules can be nested
- C) Only with Terraform Cloud
- D) Only one level deep

### Answers 61–80
61. **B** — `for_each` keys by name; `count` by index.
62. **B** — Flatten removes empty lists.
63. **B** — Iterates over the set.
64. **B** — Generates nested blocks.
65. **B** — Replaces the resource when the dependency is replaced.
66. **B** — `prevent_destroy`.
67. **B** — Zero-downtime replacement.
68. **B** — Ignores drift on the attribute.
69. **D** — All are valid sources.
70. **C** — `module.vpc.output.id` (where `id` is the output name) — actually it's `module.vpc.<output_name>`. The correct syntax shown is `module.vpc.output.id`; some literature writes simply `module.vpc.id`. The conventional syntax is **`module.NAME.OUTPUT_NAME`** (no `.output`). Best answer per current docs: **B**. (Both forms appear; current docs use `module.vpc.id`.)
71. **B** — Right-hand map wins on conflicts.
72. **B** — Returns value or default.
73. **B** — Catches errors and returns fallback.
74. **A** — Subnet 1 of /24 size from /16 base.
75. **A** — `for` with `if`.
76. **B** — List of all values.
77. **B** — Index shifts cause cascading changes.
78. **A** — Conventional module layout.
79. **B** — Registry pattern.
80. **B** — Modules can call modules.

(Q70 note: documented modern syntax is `module.<NAME>.<OUTPUT>` — answer B is technically correct.)

---

## Section 6: Workspaces, Terraform Cloud, Misc (Q81–Q100)

**Q81.** What's the default workspace name?
- A) `main`
- B) `default`
- C) `prod`
- D) `root`

**Q82.** Which command creates a new workspace?
- A) `terraform workspace add`
- B) `terraform workspace new`
- C) `terraform workspace create`
- D) `terraform new workspace`

**Q83.** Are CLI workspaces recommended for prod/dev separation?
- A) Yes, primary use case
- B) Discouraged; use separate root configs / backends
- C) Only with Terraform Cloud
- D) Only with locking

**Q84.** What variable refers to the current workspace name in config?
- A) `${workspace}`
- B) `terraform.workspace`
- C) `var.workspace`
- D) `local.workspace`

**Q85.** Each workspace has its own:
- A) Source code
- B) State file
- C) Provider plugins
- D) Lock file

**Q86.** What does Sentinel provide?
- A) State encryption
- B) Policy as code
- C) Module registry
- D) State locking

**Q87.** Sentinel enforcement levels are:
- A) low / medium / high
- B) advisory / soft-mandatory / hard-mandatory
- C) info / warn / error
- D) optional / required / blocking

**Q88.** What does Terraform Cloud's "remote operations" mean?
- A) Plans/applies run on TFC infra, not your laptop
- B) Resources are remote
- C) State is remote
- D) Modules are remote

**Q89.** Which environment variable enables `TRACE` logging?
- A) `TF_DEBUG=trace`
- B) `TF_LOG=TRACE`
- C) `TF_LEVEL=trace`
- D) `TERRAFORM_LOG=TRACE`

**Q90.** How do you redirect logs to a file?
- A) `TF_LOG_FILE`
- B) `TF_LOG_PATH`
- C) `TF_OUTPUT`
- D) Pipe stdout

**Q91.** Which logging level is most verbose?
- A) DEBUG
- B) INFO
- C) WARN
- D) TRACE

**Q92.** Where does `TF_INPUT=0` apply?
- A) Disables interactive prompts
- B) Disables logging
- C) Disables refresh
- D) Disables locking

**Q93.** What's the purpose of `data "terraform_remote_state"`?
- A) Read outputs from another state file
- B) Migrate state
- C) Lock remote state
- D) Backup state

**Q94.** Which is true about provisioners?
- A) They are HashiCorp's preferred way to configure resources
- B) They are a last resort
- C) They run during plan
- D) They modify state

**Q95.** What's `self.id` in a provisioner?
- A) The Terraform process ID
- B) The ID attribute of the parent resource
- C) The state file ID
- D) The provider's ID

**Q96.** Which provisioner runs on the Terraform host?
- A) `local-exec`
- B) `remote-exec`
- C) `file`
- D) `chef`

**Q97.** What's the provisioner default `on_failure`?
- A) `continue`
- B) `fail`
- C) `retry`
- D) `skip`

**Q98.** What does `terraform_data` (1.4+) replace?
- A) `null_resource`
- B) `local_file`
- C) `terraform_remote_state`
- D) `external`

**Q99.** Where can a destroy-time provisioner reference variables from?
- A) Any local, var, or other resource
- B) Only `self`, `count.index`, `each.key`
- C) Only `var.*`
- D) None

**Q100.** Which is a recommended HashiCorp practice?
- A) Use `-target` for routine deployments
- B) Commit `.tfstate` to Git
- C) Use remote backends with locking
- D) Use provisioners over `user_data`

### Answers 81–100
81. **B** — `default`.
82. **B** — `workspace new`.
83. **B** — Discouraged for prod/dev separation.
84. **B** — `terraform.workspace`.
85. **B** — Separate state per workspace.
86. **B** — Policy as code.
87. **B** — Three Sentinel levels.
88. **A** — Plans/applies on TFC infrastructure.
89. **B** — `TF_LOG=TRACE`.
90. **B** — `TF_LOG_PATH`.
91. **D** — TRACE most verbose.
92. **A** — Disables interactive prompts.
93. **A** — Read another state's outputs.
94. **B** — Last resort per HashiCorp.
95. **B** — Parent resource's id attribute.
96. **A** — `local-exec` on local host.
97. **B** — Default `fail` (taints resource).
98. **A** — Modern replacement for `null_resource`.
99. **B** — Strict reference rules.
100. **C** — Remote backends with locking.

---

## Score Yourself

| Score | Reading |
|---|---|
| 90–100 | Strong — ready for the exam |
| 75–89 | Solid — review weak sections |
| 60–74 | Borderline — go back through the cheatsheet |
| < 60 | Keep studying — focus on state, variables, and resource lifecycle |

Good luck!
