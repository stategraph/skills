# Security tables and output shapes

## security_scans

| column | type |
|---|---|
| id | uuid |
| state_id | uuid |
| tenant_id | uuid |
| tx_id | uuid |
| kind | text (`current` or `planned`) |
| status | text (`running`, `completed`, `failed`) |
| scanner | text |
| scanner_version | text |
| scanned_at | timestamptz |
| finding_count | integer |
| severity_breakdown | jsonb (for example `{"unknown":29}`) |
| triggered_by | text (`manual`, `preview`, `tx_apply`) |
| error_message | text |

## security_scan_findings

| column | type |
|---|---|
| scan_id | uuid |
| state_id | uuid |
| workspace | text |
| check_id | text |
| resource_fq_address | text |
| fingerprint | text |
| severity_base | text |
| severity_effective | text |
| severity_reason | text |
| is_internet_reachable | bool |
| internet_reachability_evidence | jsonb |
| blast_radius_resource_count | integer |
| blast_radius_modules | text[] |
| cross_state_refs | uuid[] |
| source_file | text |
| source_start_line | integer |
| source_end_line | integer |
| first_seen_scan_id | uuid |
| resolved_scan_id | uuid (NULL while the finding is open) |

## Queries

Scan history of a state:

```bash
stategraph sql query "SELECT scanned_at, kind, status, finding_count, severity_breakdown, triggered_by, error_message FROM security_scans WHERE state_id = 'STATE_ID' ORDER BY scanned_at DESC LIMIT 20"
```

Open findings of a state grouped by check:

```bash
stategraph sql query "SELECT check_id, severity_effective, count(*) AS n FROM security_scan_findings WHERE state_id = 'STATE_ID' AND resolved_scan_id IS NULL GROUP BY check_id, severity_effective ORDER BY n DESC, check_id LIMIT 50"
```

Rules: no `--state` flag on `sql query`; one page of 20 rows without `LIMIT`, with a note on stderr when more rows match; `--paginate` reads every page and needs an `ORDER BY`; subqueries are rejected with `QUERY_ERR`.

## JSON shapes

`security findings summary --format=json`:

```json
{
  "internet_reachable_count": 0,
  "scan": { "finding_count": 29, "id": "...", "kind": "current", "scanned_at": "2026-09-28T18:40:26+00", "scanner": "checkov", "scanner_version": "3.3.16", "severity_breakdown": { "unknown": 29 }, "state_id": "...", "status": "completed", "triggered_by": "manual" },
  "severity_breakdown": { "critical": 0, "high": 0, "info": 0, "low": 0, "medium": 0, "unknown": 29 },
  "top_checks": [ { "check_id": "CKV2_AWS_34", "count": 7 } ],
  "total": 29
}
```

`security findings list --format=json`: `{ "findings": [ ... ], "limit": N, "scan": { ... }, "total_count": N }`. `total_count` is the full count.

`security scans list --format=json`: `{ "limit": N, "scans": [ ... ], "total_count": N }`. A `planned` scan row also carries `tx_id`.

`security history --format=json`: `{ "last_scanned_at": "...", "points": [ { "date": "2026-09-28", "findings_total_count": 177, "severity_breakdown": { ... }, "blast_breakdown": [ { "bucket": "high", "count": 2 }, ... ] } ], "states_scanned": 7, "states_total": 7 }`.

`security impact summary --format=json`:

```json
{
  "added_by_severity": {},
  "computed_at": "2026-09-28T18:44:08+00",
  "cross_boundary_finding_count": 0,
  "resolved_by_severity": {},
  "scan_ids": [ "..." ],
  "source": "planned",
  "states_affected": [ "STATE_ID" ],
  "tx_id": "TX_ID"
}
```

`security impact findings` prints the same envelope with `findings_added` and `findings_resolved` arrays in place of the severity maps.

`stategraph tf plan --out plan.json` writes `{ "tx_id": "...", "task_id": "...", "revision_hash": "...", "dirs": [ ... ] }`.
