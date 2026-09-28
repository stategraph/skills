---
name: stategraph-cost
description: |
  Reads and manages cost data in Stategraph with the `stategraph cost` and
  `stategraph tx costs` commands: estimated monthly and hourly cost of a state
  or a whole tenant, attribution by tag, resource type, or provider, cost over
  time, coverage gaps, the cost delta of a pending plan, a fresh recompute, and
  FOCUS billing sources for actual cloud spend.

  Use this skill when the user asks things like: "what does this state cost",
  "what is our monthly cloud spend", "cost by team / environment / owner / tag",
  "cost by resource type or provider", "how has cost changed", "cost trend",
  "what will this change cost", "cost of this plan / transaction", "which
  resources are not priced", "coverage gaps", "recalculate cost", "actual spend
  vs estimate", "unmanaged spend", or "add / list / sync / remove a billing source".

  Do not use this skill for:
  - running a plan or apply (stategraph-change; this skill only reads the cost of a plan)
  - importing state or HCL (stategraph-import)
  - refactor sessions (stategraph-refactor)
  - inventory or SQL questions with no cost dimension (stategraph-query)

tags:
  - stategraph
  - cost
  - finops
  - billing
  - focus
metadata:
  author: Stategraph
  version: "2.0"
---

# Stategraph cost

## Setup

- `STATEGRAPH_API_BASE` replaces `--api-base`. `STATEGRAPH_API_KEY` authenticates. `STATEGRAPH_TENANT_ID` replaces `--tenant`. `STATEGRAPH_TX_ID` replaces `--tx`.
- `--state` has no environment variable. Pass the UUID. Resolve it from a name: `stategraph states resolve --name NAME` prints the bare UUID.
- Read commands accept `--format=json`. Use it when you parse the output. `cost calculate`, `tx costs`, `states resolve`, and the billing-source write commands have no `--format` and reject it with exit 124.
- Money fields are decimal strings. A missing money field means nothing in that scope is priced. `0.000000` means priced and free. Report the two cases differently.

## Disabled cost

The server can run with cost off. The signs, in the order you meet them:

- States that were never priced show no cost columns in `cost tenant`. `cost state` prints `State has never been priced. Run: stategraph cost calculate --state ID` and exits 1.
- `cost calculate` then prints `Pricing service is not configured on this server.` and exits 1. Report that once and stop. Do not retry any cost command.
- `tx costs` prints `Cost tracking is not enabled on this server.` and `tf plan` prints no `Costs:` block.

## Commands, in the order to run them

```bash
stategraph cost tenant --format=json                        # tenant total, coverage, by_provider, by_type, states[]
stategraph cost state --state STATE_ID --format=json        # one state: totals, coverage, instance_costs[] with components[]
stategraph cost tag-keys --format=json                      # tag keys you can attribute by
stategraph cost tenant --tag-key Team --format=json         # estimate split by tag value, in by_tag[]; untagged row included
stategraph cost history --format=json                       # one point per day, oldest first, last 30 days
stategraph cost history --from 2026-09-01 --to 2026-09-28 --group-by provider --format=json
stategraph cost history --group-by tag --tag-key Team --format=json
stategraph cost unsupported --state STATE_ID --format=json  # resources outside the totals: no_price or unsupported
stategraph cost attribution --format=json                   # actual FOCUS spend matched to managed resources
stategraph cost unmanaged --limit 20 --format=json          # billed resources no state manages
stategraph tx costs --tx TX_ID                              # current vs planned delta of a pending plan
stategraph cost calculate --state STATE_ID                  # queue a recompute; asynchronous
```

- `cost tenant`, `cost state`, and `cost history` are estimates from the price book. `cost attribution` and `cost unmanaged` are actuals from a billing source. Say which one you report.
- `--group-by` takes `provider`, `type`, or `tag`. `tag` needs `--tag-key`, or the server rejects the call with exit 1. `--from` and `--to` take ISO 8601 dates or timestamps.
- Read every total with its `coverage_percent`. Below 100, run `cost unsupported` and show the gap list with the total. Most gaps are free resources such as parameter groups, subnet groups, and IAM.
- With no billing source, `cost attribution` returns zeros and `cost unmanaged` returns an empty `results`. Check `cost billing-source list` before you call that a finding.
- Field names and example output shapes: read `references/output.md` when you must parse or explain a field.

## Plan-time delta

`stategraph tf plan --out plan.json` prints a `Costs:` block after the diff. It waits `--costs-wait` seconds (default 3, env `STATEGRAPH_COSTS_WAIT_SECONDS`). `--skip-costs` turns the fetch off, and `--skip-costs` with `--costs-wait` exits 124.

When the plan printed `Costs: preview not yet ready`, or the user asks later:

```bash
TX_ID=$(jq -r .tx_id plan.json)
stategraph tx costs --tx "$TX_ID"
```

- Only a transaction from `tf plan` or `tf mtx` has a preview. A plan without `--out` opens a transient transaction that you cannot query later. A plan that prints `No changes detected.` writes no `tx_id` and has no delta. Stop there.
- `Cost preview not yet ready.` with exit 1: wait a few seconds and run it again, once.
- `Transaction not found, or aborted` with exit 1: an abort drops the preview. Committed transactions keep it.
- Each `Totals` line reads `current → planned (delta)`. A dash means that side has nothing priced. Per state, `+` is added, `-` removed, `~` changed.

## Recompute

`cost calculate` prints `Cost calculation queued (task TASK_ID). Re-run ...` and exits 0 at once. The new snapshot lands when the task completes. Then:

```bash
curl -s -H "Authorization: Bearer $STATEGRAPH_API_KEY" "$STATEGRAPH_API_BASE/api/v1/tasks/TASK_ID"   # until "state":"completed"
stategraph cost state --state STATE_ID --format=json                                                 # calculated_at moves
```

Treat task state `failed` or `aborted` as an error. Recompute only when the snapshot is missing or stale. Snapshots also refresh on import, after apply, and on a daily schedule.

## Billing sources (actual spend)

Tenant admins only. A non-admin gets `Forbidden: admin privileges required` and exit 1. All take `--tenant` or `STATEGRAPH_TENANT_ID`. All except `add` and `list` take the source id as a positional argument.

```bash
stategraph cost billing-source list --format=json
stategraph cost billing-source add --provider aws --source-uri 's3://bucket/prefix/data/**/*.parquet'   # prints the source as JSON
stategraph cost billing-source update SOURCE_ID --enabled=false     # also --enabled=true, --source-uri, --region, --window-months
stategraph cost billing-source sync SOURCE_ID --from 2026-01-01     # sync now; --from backfills from a date
stategraph cost billing-source remove SOURCE_ID --auto-approve      # deletes the source and its loaded rows
```

`add`, `update`, `enable`, `disable`, `sync`, and `remove` change tenant billing configuration. Confirm with the user before you run them. `add` takes `--provider aws|gcp|azure`, `--source-uri`, and optional `--region`, `--window-months` (default 2), `--disabled`. Provider export setup, URI shapes, and `list` columns: read `references/billing-sources.md`.

## SQL

`stategraph sql query` reads the `cost_snapshots` and `cost_snapshot_resources` tables. Use it for a ranking or a slice the commands above do not give. Working queries and the parser limits: read `references/sql.md`.

## Report

For each result give the command, the scope (tenant, state, or transaction), estimate or actual, the key numbers, the coverage, and any gap or empty billing source. For a write command say what changed on the server.

## Exit codes

| Code | Meaning |
|------|---------|
| 0 | Success. `cost calculate` and `sync` exit 0 when the job is queued, not when it is done. |
| 1 | The server refused the call. The reason is on stderr: never priced, not found, forbidden, or a bad parameter. |
| 124 | Wrong or missing flag, `--format` on a command without it, `--skip-costs` with `--costs-wait`. |
