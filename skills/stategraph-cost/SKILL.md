---
name: stategraph-cost
description: |
  Cost-intelligence skill for Stategraph.

  Use this skill for:
  - the monthly/hourly cost of a state or a whole tenant
  - cost attribution by tag, owner, environment, team, provider, or resource type
  - cost over time (history / trend) for a tenant
  - "what will this change cost?" — the plan-time current-vs-planned delta for a transaction
  - cost coverage gaps (which resources are unsupported or unpriced)
  - triggering a fresh cost calculation for a state
  - managing FOCUS billing sources (actual cloud spend ingestion)

  Do not use this skill for:
  - planning or applying changes (use stategraph-change; this skill only *reads* the cost of a plan)
  - importing state or HCL (use stategraph-import)
  - refactor sessions (use stategraph-refactor)
  - general SQL / inventory queries with no cost dimension (use stategraph-query)

tags:
  - stategraph
  - cost
  - finops
  - billing
  - focus
metadata:
  author: Stategraph
  version: "1.0"
---

# Stategraph cost skill

## Purpose

This skill handles all Stategraph cost-intelligence workflows: estimated spend for a
state or tenant, actual (FOCUS) spend attribution and unmanaged-spend detection, cost
history, the plan-time cost delta of a transaction, and management of the FOCUS billing
sources those actuals come from.

## Authorization rules

### Read-only — may run without confirmation

```bash
stategraph cost tenant --tenant TENANT_ID
stategraph cost state --state STATE_ID
stategraph cost history --tenant TENANT_ID
stategraph cost attribution --tenant TENANT_ID
stategraph cost unmanaged --tenant TENANT_ID
stategraph cost unsupported --state STATE_ID
stategraph cost tag-keys --tenant TENANT_ID
stategraph cost billing-source list --tenant TENANT_ID
stategraph tx costs --tx TX_ID
```

### Require explicit user authorization — these MUTATE

```bash
stategraph cost calculate --state STATE_ID            # enqueues a recompute
stategraph cost billing-source add --tenant TENANT_ID --provider aws --source-uri URI
stategraph cost billing-source update SOURCE_ID --tenant TENANT_ID
stategraph cost billing-source remove SOURCE_ID --tenant TENANT_ID
stategraph cost billing-source enable SOURCE_ID --tenant TENANT_ID
stategraph cost billing-source disable SOURCE_ID --tenant TENANT_ID
stategraph cost billing-source sync SOURCE_ID --tenant TENANT_ID
```

`cost calculate` only enqueues a recompute job; it does not touch infrastructure. The
`billing-source add/update/remove/enable/disable/sync` commands change tenant billing
configuration and what spend gets ingested. `remove` is destructive and prompts unless
`--auto-approve` is passed.

## Required inputs

Every command requires `--api-base` (env `STATEGRAPH_API_BASE`). Resolve only the scope
the command needs:

* tenant-scoped commands take `--tenant UUID` (env `STATEGRAPH_TENANT_ID`):
  `cost tenant`, `cost history`, `cost attribution`, `cost unmanaged`, `cost tag-keys`,
  and all `cost billing-source` subcommands
* state-scoped commands take `--state UUID` (no env fallback — pass it explicitly):
  `cost state`, `cost unsupported`, `cost calculate`
* `tx costs` takes `--tx UUID` (env `STATEGRAPH_TX_ID`)
* `billing-source update/remove/enable/disable/sync` also take a positional `SOURCE_ID`

Do not gather context a command does not need. If only a state name or workspace is
known, resolve the state ID first (see stategraph-change), then pass `--state`.

## Estimates vs actuals

This distinction drives which command to reach for:

* **Estimates (pricing engine):** `cost state`, `cost tenant`, `cost history`, and the
  `tx costs` delta are computed from priced cost snapshots. They answer "what should this
  cost?" and exist even with no billing data wired up.
* **Actuals (FOCUS billing):** `cost attribution` and `cost unmanaged` read actual cloud
  spend ingested from FOCUS billing sources. They answer "what did we actually pay, and to
  which managed/unmanaged resources?" and require at least one enabled billing source.

If actuals come back empty, check `cost billing-source list` — there may be no source, or
it may be disabled or not yet synced.

## Canonical command ladder

### Tenant-level rollups (estimates + actuals)

```bash
stategraph cost tenant --tenant TENANT_ID                    # totals, coverage, per-state
stategraph cost tenant --tenant TENANT_ID --tag-key Env      # break down by a tag's values
stategraph cost history --tenant TENANT_ID                   # daily series, last 30d default
stategraph cost history --tenant TENANT_ID --from 2026-01-01 --to 2026-03-31 \
  --group-by tag --tag-key Owner                             # grouped trend
stategraph cost attribution --tenant TENANT_ID               # actual spend -> managed resources
stategraph cost unmanaged --tenant TENANT_ID --limit 20      # billed but unmanaged spend
stategraph cost tag-keys --tenant TENANT_ID                  # tag keys usable for attribution
```

`--group-by` accepts `provider`, `type`, or `tag` (the last requires `--tag-key`).
`--from`/`--to` are ISO 8601; default window is the last 30 days through now.

### State-level (estimates)

```bash
stategraph cost state --state STATE_ID                       # totals, coverage, per-resource
stategraph cost unsupported --state STATE_ID                 # resources the engine can't price
stategraph cost calculate --state STATE_ID                   # MUTATING: enqueue a recompute
```

Use `cost unsupported` to explain low coverage from `cost state`/`cost tenant`. Use
`cost calculate` when a snapshot is stale; it returns once the recompute is queued, so
re-run `cost state` afterward to read the fresh numbers.

### Plan-time delta (current vs planned)

```bash
stategraph tx costs --tx TX_ID
```

Shows the current-vs-planned cost delta for a pending transaction, per state and per
resource (added / removed / changed). `stategraph tf plan` already attempts a one-shot
cost-delta preview after the diff and waits up to `--costs-wait` seconds (default 3,
env `STATEGRAPH_COSTS_WAIT_SECONDS`); `--skip-costs` disables that fetch entirely.
Combining `--skip-costs` with `--costs-wait` is a usage error. When plan's inline preview
times out, fall back to `tx costs --tx TX_ID` for the full delta.

### Billing-source management (FOCUS actual-spend ingestion)

```bash
stategraph cost billing-source list --tenant TENANT_ID
stategraph cost billing-source add --tenant TENANT_ID --provider aws \
  --source-uri 's3://bucket/prefix/data/**/*.parquet'        # also gcp / azure URIs
stategraph cost billing-source update SOURCE_ID --tenant TENANT_ID --source-uri NEW_URI
stategraph cost billing-source enable  SOURCE_ID --tenant TENANT_ID
stategraph cost billing-source disable SOURCE_ID --tenant TENANT_ID
stategraph cost billing-source sync    SOURCE_ID --tenant TENANT_ID   # optional --from YYYY-MM-DD
stategraph cost billing-source remove  SOURCE_ID --tenant TENANT_ID   # optional --auto-approve
```

`add` requires `--provider` (`aws`, `gcp`, `azure`) and `--source-uri`; optional
`--region`, `--window-months` (default 2), and `--disabled` to create it dormant.
Credentials resolve ambiently (instance role / workload identity / az login / standard
env vars). All of these except `list` mutate billing config — confirm before running.

## Common patterns

### What does this tenant cost, and where is coverage weak?

```bash
stategraph cost tenant --tenant TENANT_ID
stategraph cost unsupported --state STATE_ID    # for any state with low coverage
```

### Who/what is driving spend?

```bash
stategraph cost tag-keys --tenant TENANT_ID                 # discover usable tag keys first
stategraph cost attribution --tenant TENANT_ID              # actuals by managed resource
stategraph cost tenant --tenant TENANT_ID --tag-key Team    # estimate split by a tag
```

### Find waste (paid for, not managed)

```bash
stategraph cost unmanaged --tenant TENANT_ID --limit 50
```

### Cost a pending change before applying

```bash
stategraph tx costs --tx TX_ID
```

## Output contract

Most read commands accept `--format table|json|simple` (table is the default; `simple`
prints one value per line with no headers). `cost calculate`, `tx costs`, and the
`billing-source` mutating subcommands do not take `--format`.

For every result, report:

1. the command run
2. the scope used: tenant, state, or transaction
3. whether the figures are **estimates** (pricing engine) or **actuals** (FOCUS billing)
4. the key numbers (monthly/hourly totals, coverage, or the signed delta)
5. any caveat — partial coverage, an empty/disabled billing source, or a stale snapshot
6. the next useful command (e.g. `cost unsupported` to explain low coverage)

For mutating commands (`calculate`, `billing-source` writes) also state plainly that the
action changes server-side state and report the result.

## Failure handling

### Missing tenant, state, or tx id

Resolve it before guessing. Use stategraph-query (`stategraph states list --tenant ...`)
to find a state ID and stategraph-change (`stategraph tx list --tenant ...`) to find a
transaction ID. Do not invent UUIDs.

### Cost numbers are empty or zero

* estimates empty → no priced snapshot yet; run `cost calculate --state STATE_ID`, then
  re-read `cost state`
* actuals empty → check `cost billing-source list`; the source may be missing, disabled,
  or not yet synced (`cost billing-source sync SOURCE_ID --tenant TENANT_ID`)

### Coverage is low

Run `cost unsupported --state STATE_ID` to list the resources the pricing engine cannot
price; report them rather than implying the total is complete.

### User asks to plan or apply from here

This skill only *reads* cost. Route plan/apply, transaction creation, and state deletion
to stategraph-change.
