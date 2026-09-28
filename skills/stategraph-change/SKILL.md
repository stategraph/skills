---
name: stategraph-change
description: |
  Plan, review, and apply Terraform changes through the Stategraph CLI, delete states, and manage transactions.

  Use this skill when the user asks to:
  - "plan this change", "what will this change do", "preview the diff", "run stategraph plan"
  - "apply the plan", "apply plan.json", "run stategraph apply", "ship it"
  - "plan networking and compute together", "one atomic plan across directories" (`stategraph tf mtx`)
  - "show me the saved plan", "summarize plan.json" (`stategraph tf show`)
  - "delete the staging state", "remove state X", "what is the state id for X"
  - "list transactions", "is a transaction stuck", "abort transaction X", "show the transaction logs", "create a transaction or session token"
  - "attach these template files to module X in stategraph.json"

  Do not use this skill for:
  - read-only SQL, inventory, blast radius, or security queries (stategraph-query)
  - importing a Terraform state file or HCL into Stategraph (stategraph-import)
  - restructuring HCL with moved blocks or a refactor session (stategraph-refactor)
  - cost questions with no plan or apply involved (stategraph-cost)
tags:
  - stategraph
  - plan
  - apply
  - transactions
metadata:
  author: Stategraph
  version: "3.0"
---

# Stategraph change skill

## Before any command

- Run `tf plan`, `tf apply`, `tf mtx`, and `states resolve` from the root module directory that holds `stategraph.json`. It names the state, so `--state` is not needed.
- Environment variables replace flags: `STATEGRAPH_API_BASE` for `--api-base`, `STATEGRAPH_TENANT_ID` for `--tenant`, `STATEGRAPH_WORKSPACE` for `--workspace` (default `default`), `STATEGRAPH_TX_ID` for `--tx`. `STATEGRAPH_API_KEY` authenticates. When `STATEGRAPH_TENANT_ID` is unset, add `--tenant UUID`; `stategraph info` prints the tenant id.
- There is no tty. Never run a command that prompts. Plan with `--out`, then apply the file. Delete with `--auto-approve`.
- Exit codes: 0 ok. 1 runtime error, message on stderr. 2 plan has changes (only with `--detailed-exitcode`). 124 unknown flag or bad flag combination. Do not retry the same command after 1 or 124.
- `tf plan` is not read-only. Every form opens a transaction and refreshes state. With `--out` the transaction stays open for apply. Without `--out` it is a transient preview, aborted at exit.

## Authorization

Plan, show, resolve, list, logs, and costs run without asking. Apply, delete, and abort need explicit user authorization in the current conversation, unless another skill's pre-authorization flow applies (stategraph-refactor defines one for finalization). Ask with the exact command:

> The plan is ready. Applying it changes infrastructure and state. Proceed with `stategraph tf apply plan.json`?

## Plan, review, apply

```bash
test -d .stategraph && echo "refactor session active"      # see Refactor side effect
stategraph tf plan --out plan.json --detailed-exitcode      # exit 2 = changes, 0 = none
stategraph tf show plan.json                                 # diff, human-readable
stategraph tf show --json plan.json                          # keys: resource_changes, planned_values, prior_state
stategraph tf apply plan.json                                # after authorization
TX=$(sed -n 's/.*"tx_id":"\([^"]*\)".*/\1/p' plan.json)
stategraph tx logs list --tx "$TX"                           # what the apply recorded
```

- `plan.json` is small JSON: `tx_id`, `task_id`, `revision_hash`, `dirs`. When the plan prints `No changes detected.`, the file has no `tx_id` and there is nothing to apply. Report that and stop.
- The plan output ends with `Plan: N to add, N to change, N to destroy.`, then a `Costs:` block and a `Security:` line. `--skip-costs --skip-security` drops both and is the fastest form.
- `--var KEY=VALUE` and `--var-file PATH` work as in `terraform plan`. `--skip-refresh` passes `-refresh=false`.
- Apply prints `Apply complete! Resources: N added, N changed, N destroyed.` and the transaction becomes `committed`.

### Preview without a plan file

`stategraph tf plan` with no `--out` prints the diff and writes nothing. Its transaction is aborted at exit, so `tx costs` and `security impact` cannot read it later. Use it only when the user wants to see the diff and nothing else.

### Apply without a plan file

`stategraph tf apply` with no PLAN plans and then prompts. Without a tty it fails with `the approval prompt for stategraph apply requires an interactive terminal, but stdin is not a tty`. Do not run it. Only when the user explicitly asks for one-step apply: `stategraph tf apply --auto-approve`.

### Apply errors

- `Local revision hash does not match the planned revision hash.` The HCL or tfvars changed after the plan. Plan again. `--skip-revision-check` applies the old plan anyway; use it only when the user asks for that.
- `Error: Transaction conflict` with `Conflicting transactions:` and an id. Another transaction committed overlapping resources first. This plan's transaction is aborted. Plan again.
- `A state must exist before you can run 'plan' or 'apply'.` No `stategraph.json` here. Find the right directory, or route to stategraph-import.

### Refactor side effect

When `.stategraph/` exists in the module directory, a refactor session is open. `tf plan` then runs a refactor step and prints `WARNING: Running in refactor mode.` `tf apply` runs a step, commits, prints `WARNING: Running in refactor mode. Finalizing refactor session.`, writes the `moved` blocks file, and ends the session. Tell the user before applying. Route session work to stategraph-refactor.

## Several directories in one transaction

```bash
stategraph tf mtx --out plan.json --skip-costs --skip-security ./networking ./compute
stategraph tf show plan.json
stategraph tf apply plan.json
```

Each directory needs `stategraph.json`. All states apply or none do. `--out` is required. `mtx` does not drive a refactor session.

## Cross-state references

A `data "terraform_remote_state"` block with `backend = "local"` fails to plan when its file is missing: `Error: Unable to find remote state` and `No stored state was found for the given workspace in the given backend.` Report the missing path and stop. Stategraph pulls a referenced state into the same transaction only when the data source uses `backend = "http"` with the address `https://stategraph/<state-id>`. Do not rewrite backends unless the user asks.

## Delete a state

```bash
stategraph states resolve --name NAME                                    # prints the id
stategraph states summary --state ID                                     # show the user what goes
stategraph states delete --state ID --workspace default --auto-approve   # after authorization
```

Delete removes resources, instances, HCL, transaction logs, cost and security data. It cannot be undone. Without `--auto-approve` it prints `Aborted` and exits 1, even with `STATEGRAPH_NON_INTERACTIVE=1`. It needs tenant admin. An unknown name gives `Error: No state with workspace 'NAME' found for NAME`.

## Transactions

```bash
stategraph tx list                                    # newest first: id, state, state_names, plan_summary, tags
stategraph tx logs list --tx TX_ID                    # actions hcl_set and state_set, with object_type and state_id
stategraph tx costs --tx TX_ID                        # cost delta of a transaction that a plan with --out opened
stategraph tx abort --tx TX_ID                        # after authorization; prints the tx with state aborted
stategraph tx create --tag key=value                  # empty open transaction, prints it as JSON
stategraph tx create-with-session --tag key=value     # prints a session token, usable as STATEGRAPH_API_KEY
```

Output is always JSON. `--format` is rejected with exit 124. States: `open`, `previewing`, `previewed`, `committing`, `committed`, `failed`, `failed-committed`, `aborted`. A stuck plan shows `previewing` or `previewed`. `tx costs` on an aborted or hand-created transaction prints `Transaction not found, or aborted`.

## Cost and security in plan output

- `Costs:` shows current, planned, and delta per total and per resource. A dash means nothing priced on that side. When every value is a dash and `Coverage: 0.0%`, the server has no pricing. Report that once; do not retry.
- `Costs: preview not yet ready. Run stategraph tx costs --tx <id>`: run that after the plan.
- `Security: impact not yet ready. Run stategraph security impact summary --tx <id>`: run that. `Security is disabled on the server (STATEGRAPH_SECURITY)` means scanning is off. Report once.
- `--costs-wait N` and `--security-wait N` change the 3 second wait. Each is rejected (exit 124) together with its `--skip-*` flag.

## stategraph.json file attachments

```bash
stategraph config files attach --workspace default --address 'module.app.*' './templates/*.tpl'
stategraph config files remove --workspace default --address 'module.app.*'
```

Local only, no server call. Use when HCL reads files with `file()`, `templatefile()`, or `fileset()`. Commit `stategraph.json`.

## Report

Plan: command, directory and state, the `Plan:` line, notable resources, the exact apply command, an approval request. Apply: command, the `Apply complete!` line, transaction id. Delete or abort: target, that it is irreversible, result. Failure: error lines verbatim, cause, next step. Never ask to apply a failed or empty plan.

## Routing

Import: stategraph-import. Moving resources between modules: stategraph-refactor. Read-only questions: stategraph-query. Cost with no change involved: stategraph-cost.

Read `references/commands.md` for every flag, output samples, and the error catalogue.
