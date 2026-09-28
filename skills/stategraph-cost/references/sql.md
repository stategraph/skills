# Cost data in SQL

Read this when a cost question needs a ranking, a filter, or a slice that the `stategraph cost` commands do not give. Run queries with `stategraph sql query "QUERY"` (`--format=json` to parse, `--paginate` to fetch every page). There is no `--state` flag on `sql query`. Filter with `WHERE state_id = '…'`.

## Tables

| Table | One row per | Columns |
|-------|-------------|---------|
| `cost_snapshots` | snapshot of one state | `id`, `state_id`, `tenant_id`, `kind`, `source`, `triggered_by`, `monthly_cost`, `hourly_cost`, `currency`, `resource_count`, `supported_count`, `priced_count`, `calculated_at`, `tx_id`, `pricing_service_version` |
| `cost_snapshot_resources` | resource in a snapshot | `snapshot_id`, `address`, `type`, `provider`, `region`, `monthly_cost`, `hourly_cost`, `supported`, `no_price`, `tags` (jsonb), `components` (jsonb), `cloud_resource_id` |
| `focus_billing_sources` | billing source | `id`, `tenant_id`, `provider`, `source_uri`, `region`, `window_months`, `enabled`, `last_status`, `last_error`, `last_synced_at`, `last_row_count`, `created_at`, `updated_at` |

## Rules

- `monthly_cost` and `hourly_cost` are text, `NULL` when the resource has no billable cost. Cast with `::real` for math and filter `WHERE monthly_cost IS NOT NULL` before you rank or sum.
- Snapshots are append-only. Each recompute adds a `kind = 'current'` row. Scope a resource query to one `snapshot_id` to avoid double counts.
- `kind` is `current` for estimates or `planned` for plan-time predictions tied to a `tx_id`. Filter `WHERE kind = 'current'`.
- A `JOIN` across the two tables and a subquery in `WHERE` fail to parse. Run two queries: the snapshot id, then the resources.
- `count` is a reserved word. Alias a count as `cnt`. Sort by the aggregate expression, not the alias.
- Extract a tag with `tags#>>'{Key}'`, in both `SELECT` and `GROUP BY`.

## Queries

Per-state totals across the tenant, latest first among priced snapshots:

```sql
SELECT state_id, monthly_cost, priced_count, resource_count
FROM cost_snapshots
WHERE kind = 'current' AND monthly_cost IS NOT NULL
ORDER BY monthly_cost::real DESC
LIMIT 10
```

Latest snapshot id of one state (use `--format=simple` to get the bare id):

```sql
SELECT id FROM cost_snapshots
WHERE state_id = 'STATE_ID' AND kind = 'current'
ORDER BY calculated_at DESC LIMIT 1
```

Most expensive resources in that snapshot:

```sql
SELECT address, type, monthly_cost
FROM cost_snapshot_resources
WHERE snapshot_id = 'SNAPSHOT_ID' AND monthly_cost IS NOT NULL
ORDER BY monthly_cost::real DESC
LIMIT 10
```

Monthly cost by resource type:

```sql
SELECT type, sum(monthly_cost::real) AS monthly, count(*) AS cnt
FROM cost_snapshot_resources
WHERE snapshot_id = 'SNAPSHOT_ID' AND monthly_cost IS NOT NULL
GROUP BY type
ORDER BY sum(monthly_cost::real) DESC
```

Cost by tag value. The empty group holds resources without the tag:

```sql
SELECT tags#>>'{Team}' AS team, sum(monthly_cost::real) AS monthly
FROM cost_snapshot_resources
WHERE snapshot_id = 'SNAPSHOT_ID' AND monthly_cost IS NOT NULL
GROUP BY tags#>>'{Team}'
ORDER BY sum(monthly_cost::real) DESC
```

Coverage gaps in a snapshot:

```sql
SELECT type, count(*) AS cnt
FROM cost_snapshot_resources
WHERE snapshot_id = 'SNAPSHOT_ID' AND no_price
GROUP BY type
ORDER BY count(*) DESC
```

For a tenant-wide tag split prefer `stategraph cost tenant --tag-key Team`. It counts each state once and groups untagged resources.
