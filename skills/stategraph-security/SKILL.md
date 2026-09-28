---
name: stategraph-security
description: |
  Security scanning results in Stategraph: the findings of a state, its scan history, the tenant's posture over time, and the security impact of a planned change.

  Use this skill when the question is about Stategraph or a Stategraph state and asks:
  - "any security findings", "what did the last scan find", "which checks fail in state X"
  - "critical findings in state X", "high severity findings", "internet-reachable findings"
  - "is this change making us less secure", "security impact of this plan or transaction"
  - "run a security scan", "rescan state X", "when was X last scanned"
  - "security posture over time", "security trend for the tenant"
  - checkov results stored by Stategraph

  Do not use this skill for:
  - security questions with no Stategraph context (code review, cloud hardening, CVEs, secrets)
  - inventory queries about security groups, IAM, or buckets with no scan involved (stategraph-query)
  - planning or applying changes (stategraph-change); this skill only reads the security impact of a plan
  - the checkov step of a Stategraph Orchestration workflow
tags:
  - stategraph
  - security
  - checkov
  - findings
metadata:
  author: Stategraph
  version: "1.0"
---

# Stategraph security skill

## Setup

Environment variables replace required flags: `STATEGRAPH_API_BASE` for `--api-base`, `STATEGRAPH_TENANT_ID` for `--tenant`, `STATEGRAPH_TX_ID` for `--tx`. `STATEGRAPH_API_KEY` authenticates. `--state` has no environment variable. Add `--format=json` when you parse the output.

## Disabled server

When scanning is off, every `stategraph security` command prints this line and exits 0:

```text
Security is disabled on the server (STATEGRAPH_SECURITY).
```

Report it once and stop. Do not retry, do not run the other security commands, and do not cross-check with SQL.

## Findings of a state

Run these in order. Stop as soon as the question is answered.

### 1. State id

```bash
stategraph states resolve --name NAME                                    # prints the UUID
stategraph states resolve                                                # in a directory with stategraph.json
stategraph states list --tenant "$STATEGRAPH_TENANT_ID" --format=json    # every state with its id
```

An unknown name exits 1 with `Error: No state with workspace 'NAME' found for NAME`. `stategraph info` prints the tenant id.

### 2. Summary

```bash
stategraph security findings summary --state STATE_ID --format=json
```

JSON keys: `total`, `internet_reachable_count`, `severity_breakdown` (critical, high, medium, low, info, unknown), `top_checks` (check_id, count), and `scan` (id, scanned_at, scanner, scanner_version, status, finding_count). The table has one row per metric plus `top:CHECK_ID` rows.

When every finding is `unknown`, the scanner reported no severity. Rank by `total` and `top_checks` instead, and say so.

A state without a completed scan prints `No completed current scan for this state yet.` and exits 1. Go to step 4.

### 3. List, filtered by severity

```bash
stategraph security findings list --state STATE_ID --severity critical --format=json
stategraph security findings list --state STATE_ID --severity high --limit 20
stategraph security findings list --state STATE_ID --format=json
```

`--severity` takes one value: `critical`, `high`, `medium`, `low`, or `unknown`. Omit it for all severities. `--limit N` caps the rows; without it every finding is returned. Table columns: `check_id`, `severity_effective`, `resource_fq_address`, `blast`, `fingerprint`. An empty result prints nothing in table form; JSON gives `"findings": []` and `"total_count": 0`. Each JSON finding has `check_id`, `resource_fq_address`, `severity_base`, `severity_effective`, `blast_radius_resource_count`, `blast_radius_modules`, `cross_state_refs`, `fingerprint`, `first_seen_scan_id`, `state_id`, `workspace`.

### 4. Scan history

```bash
stategraph security scans list --state STATE_ID --limit 5
```

Newest first. Columns: `scanned_at`, `kind` (`current` for a scan of the state's HCL, `planned` for a transaction's change), `status` (`running`, `completed`, `failed`), `scanner`, `findings`, `scan_id`, `triggered_by` (`manual`, `preview`, `tx_apply`). No scans: the table prints nothing, JSON gives `"scans": []`. A state with no `current` scan has never been scanned. Offer to trigger one.

## Trigger a scan

```bash
stategraph security scan --state STATE_ID
```

Prints `Security scan queued (task UUID). Re-run ... once it completes.` and exits 0. The scan runs in the background and takes about 15 seconds. Wait, then check:

```bash
stategraph security scans list --state STATE_ID --limit 5 --format=json
```

Find the newest row with `"kind": "current"`. When its `"status"` is `"completed"` and its `scanned_at` is after the trigger, re-run the summary (step 2). Otherwise wait 15 seconds and check again. Rows with `"kind": "planned"` belong to transactions and can be newer.

## Posture over time

```bash
stategraph security history --tenant "$STATEGRAPH_TENANT_ID" --format=json
stategraph security history --tenant "$STATEGRAPH_TENANT_ID" --from 2026-09-01T00:00:00Z --to 2026-09-28T00:00:00Z
```

One row per day, oldest first. Default window: 30 days ago to now. Table columns: `date`, `total`, `critical`, `high`, `medium`, `low`, `info`, `unknown`. JSON adds `states_total`, `states_scanned`, `last_scanned_at`, and a per-day `blast_breakdown`.

## Security impact of a change

The transaction id comes from the plan. `stategraph tf plan --out plan.json` writes it as `tx_id` in `plan.json`, and the plan output ends with a `Security:` block that names it when the impact was not ready. Otherwise `stategraph tx list --tenant "$STATEGRAPH_TENANT_ID"` prints JSON with `id` and `state_names` per transaction.

```bash
stategraph security impact summary --tx TX_ID --format=json
stategraph security impact findings --tx TX_ID
```

`summary` JSON: `added_by_severity`, `resolved_by_severity`, `states_affected`, `cross_boundary_finding_count`, `source` (`planned` at plan time, `commit` after apply), `computed_at`. The table has `added:SEVERITY` and `resolved:SEVERITY` rows. `findings` is always JSON, with `findings_added` and `findings_resolved` arrays.

While the server computes it, both commands print `Security impact not yet ready for this transaction; try again shortly.` and exit 0. Wait 10 seconds and re-run, up to three times. An unknown or aborted transaction prints `No security impact for this transaction (unknown or aborted).` and exits 1.

Plan-time flags (the stategraph-change skill owns `tf plan`):

- `--security-wait SECONDS` waits up to that long for the impact (default 3). When ready, the plan prints a `Security:` block with `Added findings`, `Resolved findings`, and `States affected`. When not ready it prints `Security: impact not yet ready. Run stategraph security impact summary --tx TX_ID to view it later.`
- `--skip-security` skips the preview. Passing both exits 124 with `Error: --skip-security is mutually exclusive with --security-wait.`

## SQL cross-check

`sql query` has no `--state` flag: filter on `state_id` in the query. Without `LIMIT` it returns one page of 20 rows. Subqueries are rejected.

```bash
stategraph sql query "SELECT check_id, resource_fq_address, severity_effective, blast_radius_resource_count FROM security_scan_findings WHERE state_id = 'STATE_ID' AND resolved_scan_id IS NULL ORDER BY check_id, resource_fq_address LIMIT 100" --format=json
```

`resolved_scan_id IS NULL` keeps the open findings. Read `references/tables.md` for the columns of `security_scans` and `security_scan_findings` and two more queries.

## Reporting

- Give the total and the count per severity, then the top checks with the resources they hit.
- `unknown` is a severity bucket, not an absence of findings. Report its count.
- Name the scan behind the numbers: `scanned_at`, `scanner`, `scanner_version` from the summary's `scan` object.
- Exit codes: 0 ok; 1 no completed scan, unknown transaction, or unknown state name; 123 error on stderr; 124 wrong flag; 125 internal bug.
