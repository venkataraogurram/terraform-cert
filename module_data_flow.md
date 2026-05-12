# Module Data Flow — All Scenarios

---

## Scenario 1: Root → Child (Passing data INTO a child module)

**Mechanism: Input Variables**

```
ROOT MODULE                         CHILD MODULE (./modules/web)
┌──────────────────────┐           ┌──────────────────────────┐
│                      │           │ variables.tf:            │
│ module "web" {       │  passes   │                          │
│   source = "./web"   │─────────►│ variable "vpc_id" {      │
│   vpc_id = "vpc-123" │           │   type = string          │
│   env    = "prod"    │           │ }                        │
│ }                    │           │ variable "env" {         │
│                      │           │   type = string          │
└──────────────────────┘           │ }                        │
                                   │                          │
                                   │ main.tf:                 │
                                   │ resource "aws_instance" {│
                                   │   subnet = var.vpc_id    │
                                   │   tags = { Env = var.env}│
                                   │ }                        │
                                   └──────────────────────────┘
```

**Rule**: Child module MUST declare a `variable` block for every input it expects. Root passes values as arguments in the `module` block.

---

## Scenario 2: Child → Root (Getting data OUT of a child module)

**Mechanism: Output + `module.<NAME>.<OUTPUT>`**

```
CHILD MODULE (./modules/web)        ROOT MODULE
┌──────────────────────────┐       ┌─────────────────────────────┐
│ resource "aws_instance" {│       │                             │
│   ami = "ami-123"        │       │ module "web" {              │
│ }                        │       │   source = "./modules/web"  │
│                          │       │ }                           │
│ outputs.tf:              │       │                             │
│                          │output │ # Access child's output:    │
│ output "instance_id" {   │──────►│ resource "aws_eip" "web" {  │
│   value = aws_instance   │       │   instance =                │
│          .web.id         │       │     module.web.instance_id  │
│ }                        │       │ }                           │
└──────────────────────────┘       └─────────────────────────────┘
```

**Rule**: Child MUST define an `output` block. Without it, root has **zero visibility** into child's resources.

```hcl
# This FAILS — root cannot reach into child's resources directly
resource "aws_eip" "web" {
  instance = aws_instance.web.id          # ERROR! resource is inside child
  instance = module.web.aws_instance.web  # ERROR! wrong syntax
}

# This WORKS
resource "aws_eip" "web" {
  instance = module.web.instance_id       # CORRECT — uses output
}
```

---

## Scenario 3: Child Module → Child Module (Module to Module)

**Mechanism: Root acts as the BRIDGE — output from one, input to another**

```
MODULE A (./modules/network)     ROOT MODULE                MODULE B (./modules/app)
┌─────────────────────┐        ┌───────────────────┐      ┌─────────────────────┐
│                     │        │                   │      │                     │
│ resource "aws_vpc"{ │        │ module "network" {│      │ variable "vpc_id" { │
│   cidr = "10.0.0/16"│        │   source = "..."  │      │   type = string     │
│ }                   │        │ }                 │      │ }                   │
│                     │        │                   │      │                     │
│ output "vpc_id" {   │──OUT──►│ module "app" {    │─IN──►│ resource "aws_sub" {│
│   value = aws_vpc   │        │   source = "..."  │      │   vpc_id = var.vpc_id│
│     .main.id        │        │   vpc_id =        │      │ }                   │
│ }                   │        │   module.network  │      │                     │
│                     │        │     .vpc_id       │      │                     │
└─────────────────────┘        │ }                 │      └─────────────────────┘
                               └───────────────────┘
```

**Full code example:**

```hcl
# ===== modules/network/main.tf =====
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# ===== modules/network/outputs.tf =====
output "vpc_id" {
  value = aws_vpc.main.id
}
output "subnet_id" {
  value = aws_subnet.main.id
}

# ===== modules/app/variables.tf =====
variable "vpc_id" {
  type = string
}
variable "subnet_id" {
  type = string
}

# ===== modules/app/main.tf =====
resource "aws_instance" "web" {
  subnet_id = var.subnet_id
  # ...
}

# ===== Root module main.tf (THE BRIDGE) =====
module "network" {
  source = "./modules/network"
}

module "app" {
  source    = "./modules/app"
  vpc_id    = module.network.vpc_id       # network output → app input
  subnet_id = module.network.subnet_id    # network output → app input
}
```

**Key rule**: Child modules **CANNOT talk to each other directly**. The root module always acts as the wiring layer.

---

## Scenario 4: Child → External Consumer (via Root Outputs)

**Mechanism: Root module re-exports child output as its own output**

```
CHILD MODULE              ROOT MODULE                    EXTERNAL
┌──────────────┐         ┌──────────────────┐          ┌──────────────┐
│ output "ip" {│──OUT───►│ output "web_ip" {│──OUT────►│ terraform    │
│  value = ... │         │  value =         │          │  output      │
│ }            │         │   module.web.ip  │          │              │
└──────────────┘         │ }                │          │ remote_state │
                         └──────────────────┘          │  data source │
                                                       └──────────────┘
```

```hcl
# Root outputs.tf — re-exports child output
output "web_ip" {
  value = module.web.ip
}
```

This is how other Terraform configurations can read your outputs using `terraform_remote_state`.

---

## Scenario 5: Nested Modules (Grandchild)

```
ROOT                    CHILD (Module A)              GRANDCHILD (Module B)
┌──────────┐           ┌──────────────────┐          ┌─────────────────┐
│module "a"{│──input──►│ module "b" {     │──input──►│ variable "x" {} │
│ x = "hi" │           │   source = "..."  │          │ resource "..." { │
│}          │           │   x = var.x      │          │   name = var.x  │
│           │           │ }                │          │ }               │
│           │◄──output─│ output "result" {│◄─output──│ output "id" {   │
│           │           │  value =         │          │  value = ...    │
│module.a   │           │   module.b.id   │          │ }               │
│ .result   │           │ }                │          │                 │
└──────────┘           └──────────────────┘          └─────────────────┘
```

Data must pass **through every layer** — it cannot skip levels.

---

## Summary Table

| Direction | Mechanism | Syntax |
|---|---|---|
| **Root → Child** | Input variables | `module "x" { var_name = value }` |
| **Child → Root** | Output blocks | `module.x.output_name` |
| **Child → Child** | Root bridges output→input | `module "b" { input = module.a.output }` |
| **Child → External** | Root re-exports as output | `output "y" { value = module.x.out }` |
| **Nested (grandchild)** | Each layer passes through | Cannot skip levels |

---

## Golden Rules

1. **Modules are black boxes** — you can only interact through inputs (variables) and outputs
2. **Root is always the wiring layer** — it connects modules to each other
3. **No direct cross-module references** — `module.a` cannot reference `module.b` inside its own code
4. **No skipping levels** — grandchild data must flow through the child first
5. **No output = no access** — if the child doesn't declare an output, that data is invisible to everyone outside
