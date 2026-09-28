---
name: stategraph
description: |
  Router for the Stategraph CLI. It picks one of seven Stategraph subskills and hands the task to it through the Skill tool. It also carries the connection, output, and exit-code rules every subskill relies on.

  Use this skill when the task names Stategraph or its CLI, for example:
  - "what S3 buckets do we have in Stategraph?", "summarize the acme-networking state", "what depends on aws_vpc.main?", "which AWS resources are unmanaged?"
  - "plan this change", "apply plan.json", "is there an open transaction?", "delete the staging state"
  - "import terraform.tfstate", "wire this repo to Stategraph", "push the HCL into the existing state"
  - "split main.tf into modules without losing state", "move the RDS resources into module.db"
  - "what does this tenant cost?", "spend by Owner tag", "what will this change cost?"
  - "any critical security findings in acme-platform?", "did this plan add findings?", "rescan the state"
  - "create a read-only CI token", "what can this token do?", "grant the platform group plan rights"

  Do not use this skill for generic Terraform or OpenTofu syntax, cloud questions with no Stategraph context, or Terraform Cloud and Atlantis questions unless the user is using or migrating to Stategraph.
tags:
  - stategraph
  - terraform
  - infrastructure
  - iac
  - devops
metadata:
  author: Stategraph
  version: "3.0"
---

# Stategraph router

Select exactly one subskill and invoke it with the Skill tool as the first action. The subskill has the exact commands. Do not answer from memory and do not run Stategraph commands first, except the orientation call below when the tenant or state is unknown.

## Routing

| Request | Subskill |
|---|---|
| SQL queries, inventory, state summaries, modules, resources, providers, outputs, blast radius, dependency analysis, gap analysis | `stategraph-query` |
| `tf plan`, `tf apply`, `tf show`, `tf mtx`, delete a state, resolve a state ID, transactions (create, list, abort, logs), `config files attach` and `remove`, change review and approval | `stategraph-change` |
| Import a `.tfstate` into a new state, import HCL into an existing state, wire a repo to Stategraph, import preflight | `stategraph-import` |
| Address rewrites: carve a root module into child modules, move resources and keep state addresses, `moved` blocks | `stategraph-refactor` |
| Cost of a state or tenant, attribution by tag, owner, provider, or type, cost history, plan-time cost delta, unpriced resources, recalculate, billing sources | `stategraph-cost` |
| Security findings (list, summary), scan history, trigger a scan, posture over time, security impact of a transaction | `stategraph-security` |
| Access tokens (create, list, delete), default capabilities, capability group rules, `whoami`, what a token or user can do | `stategraph-capabilities` |

When the user types a slash command (`/stategraph-query`, `/stategraph-change`, `/stategraph-import`, `/stategraph-refactor`, `/stategraph-cost`, `/stategraph-security`, `/stategraph-capabilities`), invoke that skill without re-routing.

Tie-breakers:

- "What will this change cost?" reads a transaction: `stategraph-cost`. Producing the plan itself: `stategraph-change`.
- "Did this plan add findings?" is `stategraph-security`. Running the plan is `stategraph-change`.
- HCL into an existing state is `stategraph-import` (`stategraph import tf --hcl`), not `stategraph-change`.
- A refactor that adds, removes, or changes resources is `stategraph-change`. `stategraph-refactor` only rewrites addresses.
- An SQL question with a security flavor ("which security groups allow 0.0.0.0/0") is `stategraph-query`. Findings and scans are `stategraph-security`.
- `version`, `diagnostics`, and `hcl addresses|eval|json` have no subskill. Run `stategraph <command> --help=plain` once, then the exact command it shows.

## Global rules

### Connection

| Env var | Replaces | Note |
|---|---|---|
| `STATEGRAPH_API_BASE` | `--api-base` | Required on every API command. |
| `STATEGRAPH_API_KEY` | (no flag) | Bearer token. Missing or wrong prints `Unauthorized`, exit 1. |
| `STATEGRAPH_TENANT_ID` | `--tenant` | Tenant UUID. |
| `STATEGRAPH_WORKSPACE` | `--workspace` | Default `default`. |
| `STATEGRAPH_NON_INTERACTIVE` | `-y` | Answers yes on `import tf` and `states import`. `tf apply` without a plan file still needs `--auto-approve`. |

Flags go after the subcommand. When an env var is set, omit its flag.

### Orientation

Run `stategraph info` once when the user, tenant, or state is unknown. It prints the user, capabilities, server version, and tenant IDs. Add `--format=json` to parse it.

```bash
stategraph info --format=json
stategraph states list --format=json
```

`states list` gives `id`, `name`, `workspace`, `group_id`, `created_at`. Every `--state` flag takes the `id`, never the name. Use `stategraph user tenants list` only when `info` is not enough. Skip orientation when the user already gave the tenant or state ID.

### Output and exit codes

Add `--format=json` to read commands when parsing the output (`info`, `whoami`, `states list|summary`, `sql query`, `gaps`, `blast-radius`, `cost`, `security`). `tx` commands always print JSON and reject `--format`.

JSON list output is wrapped: `states list`, `states modules list`, `tx list`, and the tenants in `info` use `.results[]`; `user access-tokens list` uses `.tokens[]`; `security scans list` uses `.scans[]`; `security findings list` uses `.findings[]`; `capabilities group list` uses `.rules[]`. `sql query` prints a bare array.

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | Runtime error: authentication, connection, or a failed command |
| 2 | `tf plan` or `tf mtx` with `--detailed-exitcode`: changes present |
| 123 | Other error, text on stderr |
| 124 | Command-line parse error: a wrong or missing flag |
| 125 | Internal error |

Exit 124 means the flag does not exist. Never guess a flag; the subskill has the exact command. `sql query` has no `--state`: filter with `WHERE state_id = '...'`. Read the stderr text before any second attempt.

### Disabled features

Report the message once and stop. Do not retry.

- Security off: `Security is disabled on the server (STATEGRAPH_SECURITY).`
- Cost off: `Pricing service is not configured on this server.`

### Canonical commands and aliases

Run and print the canonical form. Accept the alias when the user or a doc uses it.

| Alias | Canonical |
|---|---|
| `stategraph plan` | `stategraph tf plan` |
| `stategraph apply` | `stategraph tf apply` |
| `stategraph query` | `stategraph sql query` |
| `stategraph gaps` | `stategraph tenant gaps analyze` |
| `stategraph blast-radius` | `stategraph states instances blast-radius` |
| `stategraph whoami` | `stategraph user whoami` |
| `stategraph caps` | `stategraph capabilities` |

`stategraph hcl import` is deprecated. Its replacement is `stategraph import tf --hcl`, owned by `stategraph-import`.

Non-Stategraph commands are allowed only where a subskill requires them: `terraform state pull` for import or refactor preflight, `tofu init -backend=false -reconfigure` and `tofu validate` for refactor verification.

### Authorization

Read-only commands run without confirmation. `tf plan` opens a transaction, refreshes state, and with `--out` writes a plan file; run it without confirmation but say so. Commands that apply, delete a state, abort a transaction, overwrite an import, trigger a scan or cost calculation, or create, change, or delete tokens, capabilities, or group rules need explicit user authorization, unless the subskill defines an upfront pre-authorization flow.

### Output contract

State the subskill selected, the command run, the key result, whether it was read-only or mutating, and the next step. For a mutating command, show the planned effect before running it.
