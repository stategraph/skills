---
name: stategraph-change
description: |
  Stateful change-management skill for Stategraph.

  Use this skill for:
  - `stategraph tf plan`
  - `stategraph tf apply`
  - deleting states
  - transaction creation, listing, and abort
  - change review and approval flow

  Do not use this skill for:
  - read-only MQL and inventory workflows
  - importing Terraform state or HCL
  - refactor sessions

tags:
  - stategraph
  - plan
  - apply
  - transactions
metadata:
  author: Stategraph
  version: "2.0"
---

# Stategraph change skill

## Purpose

This skill handles Stategraph workflows that may change state or infrastructure.

## Authorization rules

### Safe without confirmation
These are read-only and may run freely:

```bash
stategraph info
stategraph states list --tenant TENANT_ID
stategraph states summary --state STATE_ID
stategraph tx list --tenant TENANT_ID
stategraph tx logs list --tx TX_ID
stategraph tf plan --tenant TENANT_ID --out plan.json
```

### Require explicit user authorization

Do not run these until the user has explicitly authorized them:

```bash
stategraph tf apply plan.json
stategraph states delete --state STATE_ID
stategraph tx abort --tx TX_ID
```

### Important exception

A valid upfront pre-authorization flow defined by another subskill overrides the default re-confirmation rule. The refactor skill defines such an exception for finalization.

## Canonical workflow

### 1. Orient only if needed

```bash
stategraph info
```

### 2. Plan before apply

Always prefer this sequence:

```bash
stategraph tf plan --tenant TENANT_ID --out plan.json
```

Then summarize the plan before any apply.

### 3. Ask for approval before apply

Use direct language:

> The plan is ready. Applying it may change infrastructure and state. Proceed with `stategraph tf apply plan.json`?

Do not auto-apply even if the user says things like "just do it" earlier in the conversation, unless an explicit pre-authorization rule applies.

### 4. Apply only after approval

```bash
stategraph tf apply plan.json
```

## Transaction commands

Canonical forms:

```bash
stategraph tx create --tenant TENANT_ID
stategraph tx list --tenant TENANT_ID
stategraph tx logs list --tx TX_ID
stategraph tx abort --tx TX_ID
```

`tx abort` is mutating and requires explicit approval.

## State deletion

Canonical form:

```bash
stategraph states delete --state STATE_ID
```

This always requires explicit approval.

Before deletion, summarize the target state so the user can confirm the correct object.

## Output contract

### For `plan`

Report:

* the command run
* the target tenant or working directory
* whether the plan succeeded
* the important summary of changes
* the exact apply command that would run next
* a direct approval request

### For `apply`

Report:

* the command run
* whether it succeeded
* the apply summary
* whether follow-up verification is recommended

### For delete or abort

Report:

* the exact target object
* that the action is mutating
* the result

## Failure handling

### Plan fails

Return:

* the relevant error lines verbatim
* the likely cause
* the next repair step

Do not ask for approval to apply a failed or missing plan.

### Apply requested without a plan file

Generate a fresh plan first unless the user already has a valid plan artifact from the current workflow.

### Ambiguous state target for delete

Resolve the state first using:

```bash
stategraph states list --tenant TENANT_ID
```

Do not guess.

### User asks for import from here

Route to `stategraph-import`.

### User asks for repo restructure or module carve-up

Route to `stategraph-refactor`.
