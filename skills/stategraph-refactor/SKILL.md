---
name: stategraph-refactor
description: |
  Interactive Stategraph refactor skill for restructuring a Terraform repo
  without losing state by recording address rewrites.

  Use this skill for:
  - carving a root module into child modules
  - reorganizing resources across directories within one Stategraph-backed repo
  - moving resources while preserving state addresses
  - refactoring messy multi-environment repos into shared modules plus environment roots

  Do not use this skill for:
  - net-new resources
  - deleting resources
  - changing resource behavior or attributes as part of the refactor
  - cross-state operations

tags:
  - stategraph
  - refactor
  - terraform
  - modules
metadata:
  author: Stategraph
  version: "2.1"
---

# Stategraph refactor skill

## Purpose

This skill runs the interactive Stategraph address-rewrite workflow.

The goal is structural change only. The final plan must be a no-op or pure refactor-mode noise. Real infrastructure diffs mean the refactor is not complete.

For multi-environment repos, the default target topology is:

- shared child modules under `./modules/<name>/`
- environment roots kept in place under paths like `./envs/prod/` or `./live/staging/`
- each environment root rewritten to call the shared child modules

Do not default to putting child modules under `envs/<env>/<name>/` unless the user explicitly asks for that via Custom path.

## Core rules

1. one new module per `stategraph refactor step`
2. never rename and move in the same step
3. never move an existing child `module` call into a new parent in the same step as resource moves
4. verify mapping count after every step
5. do not finalize until `stategraph tf plan` reaches an allowed success category
6. layout selection is a real topology choice, not a cosmetic path choice
7. when the user selects environment-aware refactor mode, detect environment roots before the first carve-up
8. in environment-aware mode, shared child modules normally live under `./modules/<name>/` and are instantiated from the environment roots

## Allowed command set

Primary commands:

```bash
mkdir -p .stategraph && stategraph refactor start
stategraph refactor step
stategraph refactor complete
stategraph refactor abort
stategraph tf plan --tenant "$STATEGRAPH_TENANT_ID" --out plan.json
stategraph tf apply plan.json
```

Allowed helper commands only where explicitly needed:

```bash
terraform state pull > /tmp/terraform.tfstate.json
tofu init -backend=false -reconfigure >/dev/null && tofu validate
python3 -c "import json; m=json.load(open('.stategraph/refactor.json'))['map']; print(len(m),'mappings')"
```

## Upfront decisions

Before `stategraph refactor start`, gather exactly two decisions from the user.

Use the `AskUserQuestion` tool for both decisions. Send them in a single `AskUserQuestion` call so the user can answer both at once. Do not ask these in free-form prose.

### Decision 1: target refactor topology

Ask via `AskUserQuestion` which topology should be used for the refactor.

Use exactly these options:

* `Shared modules + env roots`
* `Shared modules only`
* `Custom path`

Interpretation:

* `Shared modules + env roots` means detect environment roots, keep them as roots, create reusable child modules under `./modules/<name>/`, and rewrite the environment roots to call those shared modules
* `Shared modules only` means create reusable child modules under `./modules/<name>/` from a single root-oriented repo layout with no environment-root detection
* `Custom path` means the user will provide the exact target layout prefix or topology in a follow-up answer

Do not default this silently.

Important:

* Do not add a separate `./refactor` mode.
* If the user wants output under `./refactor/...`, that is handled by `Custom path`.
* Treat `Custom path` as a real target layout choice, not as a shadow copy or staging tree.
* For messy multi-environment repos, prefer `Shared modules + env roots` over embedding child modules directly under environment directories.

### Decision 2: finalization mode

Ask via `AskUserQuestion` for exactly one of these modes:

* `PR flow (emit moved_stategraph.tf)`
* `Direct apply (rewrite state now)`
* `Print commands for manual finalization`

This upfront choice is the authorization model for finalization. Do not ask again at the end.

## Preflight

### Step 1: check whether the repo is wired

The repo is considered wired when `stategraph.json` exists in the current working directory.

* if present, continue
* if missing, run import preflight

### Step 2: import preflight when not wired

Resolve the source in this order:

1. `terraform.tfstate` in the current working directory
2. remote backend via `terraform state pull`
3. otherwise stop and report that the directory is not yet importable

### Step 3: tenant resolution

Resolve tenant in this order:

1. `STATEGRAPH_TENANT_ID`
2. exactly one tenant from `stategraph info`
3. explicit user choice if multiple tenants exist — ask via `AskUserQuestion`, with each tenant as an option

### Step 4: state name

Use the current directory basename as the default state name unless the user provides another.

### Step 5: import when approved

```bash
stategraph import tf SOURCE_FILE --tenant TENANT_ID --name STATE_NAME -y
```

Proceed only if the command succeeds and `stategraph.json` exists afterward.

## Environment-aware mode

This section applies only when the user selected `Shared modules + env roots`.

Before the first carve-up proposal, detect whether the repo has multiple environment roots.

Claude may infer environment roots only from strong signals such as:

* directories named `env`, `envs`, `environment`, or `environments`
* sibling directories named `prod`, `production`, `staging`, `stage`, `dev`, `development`, `qa`, `test`, or `sandbox`
* repeated Terraform roots whose differences are mostly backend config, tfvars, provider config, or environment-specific values rather than different infrastructure intent
* root modules with substantially similar infrastructure shape but different input values, backends, providers, or overlays

Do not infer environments from weak naming alone.

Do not confuse these with environments unless the structure strongly supports it:

* regions
* accounts
* teams
* services
* application names
* Terraform workspaces by themselves

When environment roots are detected, Claude should:

1. name the inferred environments explicitly in the carve-up proposal
2. explain which directory signal led to that inference
3. keep those directories as root modules
4. create reusable child modules under `./modules/<name>/` unless the user explicitly chose a different custom layout
5. rewrite each environment root to call the shared child modules

When environment detection is ambiguous:

* mention the ambiguity in the carve-up proposal
* choose the most likely interpretation only when the directory structure strongly supports it
* otherwise prefer the safer non-destructive interpretation and keep the existing boundaries intact

When no environment directories are detected at all, do not ask the user to clarify. Default to wrapping envs as nested child modules under "envs/" and proceed. The user already committed to `Shared modules + env roots` in Decision 1; a missing env-dir signal is not grounds for re-prompting.

Examples:

* `envs/production`, `envs/staging` -> environment roots are `envs/production` and `envs/staging`
* `live/prod`, `live/staging` -> likely environment roots if each contains a Terraform root with similar structure
* `us-east-1`, `eu-west-1` -> do not treat as environment roots unless the repo clearly uses region directories as its root-level environment structure

## Do not ask the user to define the carve-up

Once refactor mode starts, the skill owns the initiative.

Do not ask:

* what to carve out
* which directory to inspect
* whether to propose a split
* whether to read the repo first

The current working directory is the scope.

## Immediate post-start behavior

After:

```bash
mkdir -p .stategraph && stategraph refactor start
```

immediately do all of the following without pausing for permission:

1. read the repo's `.tf` files
2. if the selected topology is `Shared modules + env roots`, detect environment roots first
3. propose a concrete module carve-up
4. begin module 1 in the same turn
5. run `stategraph refactor step`
6. verify mapping count increased as expected

The correct shape is:

* short carve-up proposal
* if environment-aware mode is active, explicitly name the inferred environment roots first
* state clearly that shared child modules will be written under `./modules/<name>/`
* "Starting with <module-name> now"
* then execute

## Proactive variable/local mitigation

Before the first refactor step, inspect root-level module call arguments.

Where safe, inline root-level `var.X` and `local.X` references that would otherwise break refactor-mode planning.

Resolve `var.X` in this order:

1. `terraform.tfvars`
2. `*.auto.tfvars` with later files winning
3. `TF_VAR_X`
4. `variable "X" { default = ... }`

Never guess a missing value. If the value appears secret-like, stop and ask via `AskUserQuestion` whether to inline it or leave it for manual handling.

Preserve literal type when inlining.

Be careful with values embedded inside `jsonencode`, `merge`, or other computed expressions. Byte-level drift there is a real diff, not benign noise.

## Canonical loop

### 1. Start session

```bash
mkdir -p .stategraph && stategraph refactor start
```

### 2. Read repo and propose carve-up

Name specific modules and specific resources in each.

If the selected topology is `Shared modules + env roots`, detect environment roots first and propose the carve-up in this shape:

* environment roots remain where they are
* shared child modules are created under `./modules/<name>/`
* each environment root is rewritten to instantiate those shared modules

Examples of valid proposal shapes:

* shared modules only:
  * `modules/network`
  * `modules/data`
  * `modules/compute`

* shared modules + env roots:
  * `modules/network`
  * `modules/data`
  * `envs/production` calls `../../modules/network` and `../../modules/data`
  * `envs/staging` calls `../../modules/network` and `../../modules/data`

### 3. Move one module only

Create one new child module directory under the chosen layout, move all resources that belong to that module, and add or update the calling root `module` block.

In environment-aware mode:

* shared child module definitions normally go under `./modules/<name>/`
* environment roots remain roots
* each step should preserve the distinction between module definition and module instantiation
* keep changes scoped so the address rewrite remains attributable and verifiable

Do not default to placing child module definitions under `envs/<env>/<name>/` unless the user explicitly requested that via `Custom path`.

### 4. Record the move

```bash
stategraph refactor step
```

### 5. Verify mapping count

```bash
python3 -c "import json; m=json.load(open('.stategraph/refactor.json'))['map']; print(len(m),'mappings')"
```

If the count did not rise by the expected amount, stop immediately and diagnose before editing anything else.

### 6. Repeat

Repeat steps 3 to 5 one module at a time.

## Structural rules during carve-up

### The repo remains one state

All moved resources must remain reachable from one logical root graph. Use child modules called from the existing root or from the existing environment roots.

### Respect existing environment roots

When the selected topology is `Shared modules + env roots`, do not collapse separate environment roots into one flattened layout.

Keep each environment root in place unless the repo structure itself clearly requires a different boundary.

### Keep module definitions and module instantiations separate

In environment-aware mode, distinguish clearly between:

* module definitions under `./modules/<name>/`
* environment roots such as `./envs/prod/` or `./live/staging/`
* module instantiations inside those environment roots

Do not conflate these into a single `envs/<env>/<name>/` directory pattern unless the user explicitly chose that topology.

### Move dependencies before dependents

When a moved resource is referenced from outside its new module, add the required child output and rewrite the external reference in the same edit.

### Preserve root declarations

After carve-out, the root should contain only:

* `main.tf` for module calls
* `providers.tf` for provider and `terraform` blocks
* `variables.tf` for root variable declarations
* `locals.tf` only if root module calls still depend on locals

For environment-aware repos, apply the same rule within each environment root that continues to act as a root module.

Before deleting a root file, ensure any surviving `var.X` or `local.X` references still have declarations in the canonical root files.

### Child module provider declarations

Do not add empty provider proxy blocks such as:

```hcl
provider "aws" {}
```

Use `required_providers` in the child module's `terraform` block instead.

## Pre-finalization verification

When all modules are moved, run these gates from the repo root in order.

### Gate A: mapping coverage

```bash
python3 -c "import json; m=json.load(open('.stategraph/refactor.json'))['map']; print(len(m),'mappings')"
```

Cross-check the mapping total against the number of resources moved.

### Gate B: cheap syntax check

```bash
tofu init -backend=false -reconfigure >/dev/null && tofu validate
```

This is useful but not authoritative.

### Gate C: authoritative plan gate

```bash
stategraph tf plan --tenant "$STATEGRAPH_TENANT_ID" --out plan.json 2>&1 | tee /tmp/plan.stdout
echo "exit: ${PIPESTATUS[0]}"
```

Classify the result into exactly one category.

#### Category A: ideal no-op

Proceed when exit code is 0 and output shows either:

* `No changes`
* `0 to add, 0 to change, 0 to destroy`

#### Category B: refactor-mode noise only

Proceed only when all of the following are true:

* exit code is 0
* plan summary contains `0 to change`
* every flagged address appears in `.stategraph/refactor.json["map"]` as a key or value
* no literal attribute drift exists

Use this verification snippet:

```bash
python3 <<'PY'
import json, re
plan_out = open('/tmp/plan.stdout').read()
mapping = json.load(open('.stategraph/refactor.json'))['map']
addresses = set(re.findall(r'^\s*#\s+(\S+)\s+will be', plan_out, re.M))
mapped = set(mapping.keys()) | set(mapping.values())
unexplained = [a for a in addresses if a not in mapped]
print('flagged:', len(addresses), 'unexplained:', len(unexplained))
for a in unexplained:
    print(' UNEXPLAINED:', a)
PY
```

If the summary shows any non-zero `to change`, this is not Category B.

#### Category C: hard failure

This includes non-zero exit or errors such as:

* `Reference to undeclared local value`
* `Reference to undeclared input variable`
* `Reference to undeclared output value`
* `Output refers to sensitive values`
* `Invalid provider configuration`
* `Redundant empty provider block`
* generic `Error:`

Stop. Do not finalize.

#### Category D: real diffs

The plan succeeded but shows real changes not explained by address rewrites.

Stop. Do not finalize.

## Finalization behavior

### Layout note

`Custom path` is a target layout choice only.

It is not a separate shadow tree, scratch tree, or alternate working directory mode.

All refactor steps, verification, and finalization still operate against the real repo working tree and the active Stategraph session in the current working directory.

Only Categories A and B may proceed.

### Mode: PR flow

Run:

```bash
stategraph refactor complete
```

Report the emitted `moved_stategraph.tf` path.

### Mode: Direct apply

Reuse the verified `plan.json` from the final gate. Do not re-plan.

Run:

```bash
stategraph tf apply plan.json
```

### Mode: Print commands for manual finalization

Do not run a finalization command. Print exactly:

```text
Refactor session is ready to finalize. Mappings accumulated in .stategraph/refactor.json.

Run ONE of the following from: <REPO_ROOT>

  # Option A - PR flow (writes moved_stategraph.tf for review)
  stategraph refactor complete

  # Option B - Direct apply (rewrites state addresses in Stategraph)
  stategraph tf plan --tenant "$STATEGRAPH_TENANT_ID" --out plan.json
  stategraph tf apply plan.json
```

## Post-finalization verification

For direct apply, or later after PR merge and apply, verify:

```bash
stategraph tf plan --tenant "$STATEGRAPH_TENANT_ID" --out /tmp/verify-plan.json
stategraph mql query "SELECT address FROM resources WHERE module = ''" --state STATE_ID
```

Expected result:

* no changes in the plan
* no resources left at the root module when all resources were meant to move into child modules

## Failure summaries

When finalization is blocked, report:

* the exact error lines
* the classification category
* the most likely cause
* the next fix step

Do not say the refactor is ready until the final plan gate is Category A or Category B.
