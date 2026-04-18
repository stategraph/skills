---
name: stategraph-import
description: |
  Import and onboarding skill for Stategraph.

  Use this skill for:
  - importing Terraform state into a new Stategraph state
  - importing HCL into an existing Stategraph state
  - wiring a Terraform repo to Stategraph
  - preflight checks for importable sources

  Do not use this skill for:
  - read-only MQL queries and summaries
  - standard plan/apply workflows
  - interactive refactor sessions after the repo is already wired

tags:
  - stategraph
  - import
  - onboarding
  - terraform
metadata:
  author: Stategraph
  version: "2.0"
---

# Stategraph import skill

## Purpose

This skill handles onboarding an existing Terraform project into Stategraph.

## Mutability rule

Import changes Stategraph state and repo wiring. Treat it as a write operation.

Before running the final import command, make clear what source file will be imported, what tenant will be used, and what state name will be created.

## Allowed command set

Primary commands:

```bash
stategraph info
stategraph user tenants list
stategraph states list --tenant TENANT_ID
stategraph import tf FILE --tenant TENANT_ID --name NAME -y
stategraph hcl import --tenant TENANT_ID --state STATE_ID -y
```

Allowed external helper commands only when necessary:

```bash
terraform state pull > /tmp/terraform.tfstate.json
```

## Import source resolution

Resolve import sources in this order.

### For full Terraform state import

1. local `terraform.tfstate`
2. remote backend pulled via `terraform state pull`
3. an explicit user-provided state file path

### For HCL import into an existing state

Use:

```bash
stategraph hcl import --tenant TENANT_ID --state STATE_ID -y
```

## Tenant resolution

Resolve tenant in this order:

1. `STATEGRAPH_TENANT_ID` environment variable
2. `stategraph info` if it yields exactly one tenant
3. `stategraph user tenants list` when multiple tenants exist and the user must choose

Do not ask for tenant if it is already unambiguous.

## State naming

When creating a new state, derive a sensible default from the current working directory basename.

Ask only for the state name if it is not explicitly provided. Accept the default if the user approves it.

## Canonical workflows

### Workflow A: import local state file into a new Stategraph state

```bash
stategraph import tf terraform.tfstate --tenant TENANT_ID --name STATE_NAME -y
```

Then verify with:

```bash
stategraph states summary --state STATE_ID
stategraph states resources summary --state STATE_ID
```

### Workflow B: import remote backend state into a new Stategraph state

```bash
terraform state pull > /tmp/terraform.tfstate.json
stategraph import tf /tmp/terraform.tfstate.json --tenant TENANT_ID --name STATE_NAME -y
```

Warn that `terraform state pull` needs the backend credentials configured in the environment.

### Workflow C: import HCL into an existing state

```bash
stategraph hcl import --tenant TENANT_ID --state STATE_ID -y
```

## Verification contract

After import, verify at least one of these:

```bash
stategraph states summary --state STATE_ID
stategraph states resources summary --state STATE_ID
stategraph mql query "SELECT * FROM resources" --state STATE_ID
```

If repo wiring is expected, confirm that `stategraph.json` now exists in the current working directory.

## Response contract

Before import, report:

* source file being imported
* target tenant
* target state name or state ID
* that the operation is mutating

After import, report:

* the command run
* whether it succeeded
* the created or updated state
* the first verification result
* the next recommended step, usually query or plan

## Failure handling

### No importable source found

Stop and explain which expected files were missing:

* no `terraform.tfstate`
* no usable backend for `terraform state pull`
* no explicit user-provided source

### Multiple tenants and no clear default

Present the tenant choices explicitly. Do not guess.

### Import succeeded but repo is not wired

If `stategraph.json` was expected and is missing, report that clearly. Do not claim the repo is wired.

### User asks to start module carve-up after import

Route to `stategraph-refactor`.
