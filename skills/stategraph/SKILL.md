---
name: stategraph
description: |
  Router skill for the Stategraph CLI.

  Use this skill only when the task is specifically about Stategraph, including:
  - querying infrastructure stored in Stategraph
  - summarizing states, modules, resources, or inventory
  - blast radius or dependency analysis
  - gap analysis for unmanaged resources
  - importing Terraform state or HCL into Stategraph
  - planning or applying changes with the Stategraph CLI
  - running a Stategraph refactor session
  - cost intelligence: state/tenant cost, attribution, history, or plan-time cost preview

  Do not use this skill for:
  - generic Terraform syntax questions
  - generic cloud questions with no Stategraph context
  - general architecture advice unrelated to the Stategraph CLI
  - Terraform Cloud, Atlantis, or OpenTofu questions unless the user is explicitly using or migrating to Stategraph

tags:
  - stategraph
  - terraform
  - infrastructure
  - iac
  - devops
metadata:
  author: Stategraph
  version: "2.0"
---

# Stategraph router

## Purpose

This skill does not carry full operational detail for every workflow. Its job is to:

1. detect whether the task belongs in Stategraph
2. select exactly one subskill
3. apply global safety rules
4. hand off execution to the correct subskill

## Global rules

### Prefer canonical Stategraph commands

Prefer these canonical command families:

- `stategraph info`
- `stategraph sql schema`
- `stategraph sql query`
- `stategraph states ...`
- `stategraph tenant gaps ...`
- `stategraph tf plan`
- `stategraph tf apply`
- `stategraph import tf`
- `stategraph hcl import`
- `stategraph refactor ...`

Do not switch between aliases unless the alias is required for compatibility. Use one canonical command form in output and execution.

### Allowed non-Stategraph exceptions

The following commands are allowed only where explicitly required by a subskill:

- `terraform state pull` for import or refactor preflight when remote state must be fetched
- `tofu init -backend=false -reconfigure` and `tofu validate` for refactor verification

Outside those cases, prefer Stategraph commands over Terraform or OpenTofu commands.

### Authorization policy

Read-only commands may run without confirmation.

Commands that mutate state, delete state, or abort transactions require explicit user authorization, except where a subskill defines a valid upfront pre-authorization flow.

### Output contract

Every response should make these items clear:

- what mode was selected
- what command was run or should be run
- the key result or expected result
- whether the action is read-only or mutating
- the next step

For write operations, show the planned effect before execution whenever possible.

## Subskill routing

This skill has five subskills. **When the user's request matches one, invoke it using the Skill tool as your FIRST action.** Do NOT answer directly, do NOT run stategraph commands first. The subskill owns the workflow.

The subskills live on disk at:

- `~/.claude/skills/stategraph-query/SKILL.md`
- `~/.claude/skills/stategraph-change/SKILL.md`
- `~/.claude/skills/stategraph-import/SKILL.md`
- `~/.claude/skills/stategraph-refactor/SKILL.md`
- `~/.claude/skills/stategraph-cost/SKILL.md`

Invoke via the Skill tool using the subskill's `name` field (e.g. `stategraph-query`). Users may also type these directly as slash commands (`/stategraph-query`, `/stategraph-change`, `/stategraph-import`, `/stategraph-refactor`, `/stategraph-cost`) — when they do, invoke that skill immediately without re-routing.

### Routing rules — when you see these patterns, INVOKE the skill via the Skill tool

- User wants to query, list, summarize, inventory, or inspect Stategraph data (SQL, state summaries, modules, resources, blast radius, gap analysis, security/compliance inspection) → invoke `stategraph-query`
  - Examples: "what S3 buckets do we have?", "show all resources in this state", "what depends on aws_vpc.main?", "list modules in this state", "gap analysis for AWS"

- User wants to plan, apply, delete, or control transactions (`stategraph tf plan`, `stategraph tf apply`, state deletion, `tx create/list/abort`) → invoke `stategraph-change`
  - Examples: "plan these changes", "apply the plan", "delete this state", "abort this transaction"

- User wants to onboard a Terraform repo or state file into Stategraph (import `.tfstate`, import HCL, wire the repo) → invoke `stategraph-import`
  - Examples: "import this terraform.tfstate", "wire this repo to Stategraph", "import HCL into the existing state"

- User wants the interactive Stategraph address-rewrite workflow (carve root into child modules, restructure a repo while preserving state addresses) → invoke `stategraph-refactor`
  - Examples: "refactor this Terraform repo into modules", "move resources into child modules without losing state", "restructure the repo and preserve addresses"

- User wants cost intelligence — spend/cost of a state or tenant, cost attribution by tag/owner/provider, cost over time, the plan-time cost delta of a change, coverage gaps, or managing billing sources → invoke `stategraph-cost`
  - Examples: "what does this tenant cost?", "cost of this state", "break down spend by Owner tag", "what will this change cost?", "which resources can't be priced?"

**Do NOT answer the user's question directly when a matching subskill exists.** Each subskill has structured preflight, authorization, verification, and failure rules that produce better results than ad-hoc stategraph commands.

### Orient inline (no subskill)

Only when tenant, state, or environment context is missing and the user has not yet asked for any of the four workflows above, answer inline with an orientation command:

```bash
stategraph info
```

Use `stategraph user tenants list` only if `stategraph info` is insufficient. Use `stategraph states list --tenant TENANT_ID` only when a specific state must be selected. Do not run broad orientation if the user has already provided the required tenant or state — route straight to the matching subskill.

