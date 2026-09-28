# Carve-up guidance

Read this before proposing the module split for a repo with several environment roots, or when the user names a target layout.

## Target layouts

Pick one and state it in the proposal.

- **Shared modules only** (default for a single root): child modules under `./modules/<name>/`, called from the existing root.
- **Shared modules + env roots**: environment roots stay where they are (`./envs/prod/`, `./live/staging/`); child modules go under `./modules/<name>/`; each environment root is rewritten to call them with `source = "../../modules/<name>"`. Each environment root is its own Stategraph module directory with its own `stategraph.json`, so run a separate session in each root.
- **Custom path**: the user gives the prefix or topology. It is a real target layout, not a scratch tree. All steps still run in the real working tree.

Do not put child module definitions under `envs/<env>/<name>/` unless the user asks for that.

Ask the user only when the repo has several environment roots and the user did not name a layout. Otherwise use the default and say so.

## Environment root detection

Strong signals:

- directories named `env`, `envs`, `environment`, or `environments`
- sibling directories named `prod`, `production`, `staging`, `stage`, `dev`, `development`, `qa`, `test`, or `sandbox`
- repeated Terraform roots whose differences are backend config, tfvars, provider config, or values, not different infrastructure

Weak signals, not environments on their own: regions (`us-east-1`), accounts, teams, services, application names, Terraform workspaces.

Name the inferred roots and the directory signal in the proposal. When detection is ambiguous, keep the existing boundaries.

## Writing the child module

For each module in one step:

1. Create `./modules/<name>/main.tf` with the moved resource blocks copied byte for byte, except references that now enter through variables.
2. Add `terraform { required_providers { <provider> = { source = "<namespace>/<provider>" } } }` matching the root. No `provider "x" {}` blocks.
3. Declare a `variable` for every value the moved resources take from the root (`var.X`, `local.X`, another root resource), and pass it in the module block.
4. Declare an `output` for every attribute the root still reads (`random_pet.app_name.id` becomes `output "app_name" { value = random_pet.app_name.id }`).
5. Replace the resource blocks in the root with one `module "<name>" { source = "./modules/<name>" ... }` block.
6. Rewrite every remaining reference in the root, other modules, and root outputs to `module.<name>.<output>`. Check `count`, `for_each`, `depends_on`, `triggers`, and `jsonencode` arguments.

Address rule: the resource keeps its type and name. `random_pet.app_name` becomes `module.naming.random_pet.app_name`. Do not rename in the same step as the move.

Moving an existing child `module` call into a new parent module is its own step, after the resource moves.

## Variables and locals

A moved resource that reads `var.X` or `local.X` needs the value passed into the child module. Do not inline secrets. If a value looks secret-like (`password`, `secret`, `token`, `key`), stop and ask whether to pass it through a variable or leave that resource in place.

When inlining is needed, resolve `var.X` in this order and keep the literal type:

1. `terraform.tfvars`
2. `*.auto.tfvars`, later files win
3. `TF_VAR_X`
4. `variable "X" { default = ... }`

Values inside `jsonencode`, `merge`, or other computed expressions must stay byte-identical. A byte change there is a real diff, not noise.

## Root layout after the carve-up

When every resource has moved, the root holds:

- `main.tf` for module calls
- `providers.tf` for provider and `terraform` blocks
- `variables.tf` for root variable declarations
- `locals.tf` only if module calls still depend on locals
- `moved_stategraph.tf`, written by the apply

Before deleting a root file, confirm that every surviving `var.X` and `local.X` reference still has a declaration.
