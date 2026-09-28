# Cost command output

Read this when you parse `--format=json` output or must explain a field. The numbers are one tenant's example shape, not reference values.

## Conventions

- Money fields (`monthly_cost`, `hourly_cost`, `billed_cost`, `*_cost`) are decimal strings. Parse as decimals.
- A missing money field means nothing in that scope is priced. `"0.000000"` means priced and free.
- `coverage_percent` is `priced_count / resource_count`. Only priced resources count in a total.
- Table output prints a `(total)` row first. Blank cells are absent fields.

## `cost tenant --format=json`

Top-level keys: `by_provider`, `by_tag`, `by_type`, `coverage_percent`, `currency`, `hourly_cost`, `monthly_cost`, `priced_count`, `resource_count`, `states`, `supported_count`.

```json
{
  "monthly_cost": "1156.612000", "hourly_cost": "1.584400", "currency": "USD",
  "coverage_percent": 41.8, "resource_count": 244, "priced_count": 102, "supported_count": 244,
  "by_provider": [ { "name": "aws", "monthly_cost": "1156.612000", "hourly_cost": "1.584400", "resource_count": 244 },
                   { "name": "unknown", "resource_count": 8 } ],
  "by_type": [ { "name": "aws_db_instance", "monthly_cost": "1149.020000", "hourly_cost": "1.574000", "resource_count": 3 } ],
  "by_tag": [],
  "states": [ { "state_id": "eafdebee-…", "state_name": "acme-data-stores", "monthly_cost": "1149.020000",
                "hourly_cost": "1.574000", "coverage_percent": 61.8, "resource_count": 34, "supported_count": 34,
                "calculated_at": "2026-09-28T18:39:55Z" } ]
}
```

- `by_tag` is empty without `--tag-key`. With `--tag-key Team` it holds one row per tag value plus `untagged`. A row with `resource_count` and no money means none of those resources is priced.
- A state that was never priced appears in `states[]` with `state_id`, `state_name`, and `resource_count` only. Soft-deleted states are excluded.

## `cost state --state ID --format=json`

```json
{
  "snapshot_id": "…", "calculated_at": "2026-09-28T18:39:55Z", "source": "estimate", "triggered_by": "manual",
  "currency": "USD", "monthly_cost": "1149.020000", "hourly_cost": "1.574000",
  "resource_count": 34, "supported_count": 34, "priced_count": 21, "coverage_percent": 61.76,
  "instance_costs": [
    { "address": "aws_db_instance.primary", "type": "aws_db_instance", "provider": "aws",
      "monthly_cost": "656.270000", "hourly_cost": "0.899000", "supported": true, "no_price": false,
      "cloud_resource_id": "arn:aws:rds:us-east-1:…:db:acme-primary",
      "tags": { "Team": "data", "Environment": "production" },
      "components": [
        { "name": "Database instance (on-demand, Multi-AZ, db.r6g.xlarge)", "unit": "hours",
          "price": "0.8990000000", "hourly_quantity": "1", "monthly_quantity": "730",
          "hourly_cost": "0.899000", "monthly_cost": "656.270000" } ] },
    { "address": "aws_db_subnet_group.main", "type": "aws_db_subnet_group", "provider": "aws",
      "supported": true, "no_price": true }
  ]
}
```

- `triggered_by` is one of `state_import`, `tx_apply`, `actuator_commit`, `scheduled`, `manual`, `preview`.
- `supported: true, no_price: true` is a recognized type with nothing billable. `supported: false` is a type with no pricing model. Both are outside the totals and are what `cost unsupported` lists.
- Table columns: `address type provider region monthly hourly supported no_price`. The `(total)` row shows `priced/total priced` in the supported column.

## `cost unsupported --state ID --format=json`

```json
{ "calculated_at": "2026-09-28T18:39:55Z",
  "resources": [ { "address": "aws_db_parameter_group.postgres16", "type": "aws_db_parameter_group",
                   "provider": "aws", "supported": true, "no_price": true } ] }
```

## `cost tag-keys --format=json`

```json
{ "tag_keys": [ "Environment", "Project", "Team" ] }
```

## `cost history --format=json`

```json
{ "points": [
    { "date": "2026-09-27", "groups": [] },
    { "date": "2026-09-28", "monthly_cost": "1156.612000", "hourly_cost": "1.584400",
      "groups": [ { "name": "aws", "monthly_cost": "1156.612000", "resource_count": 244 } ] } ] }
```

- One point per day, oldest first. A day before the first snapshot has `date` and `groups` only.
- `groups[]` is empty without `--group-by`. With `--group-by provider|type|tag` each point carries one group per value.
- Table output has `date monthly hourly` columns only. Use JSON to see groups.

## `cost attribution --format=json`

```json
{ "total_cost": "0", "attributed_cost": "0", "unallocated_cost": "0", "unmanaged_cost": "0",
  "coverage_percent": 0.0, "attributed_count": 0, "unmanaged_count": 0, "unallocated_count": 0,
  "uncomputed_count": 0, "line_count": 0, "by_provider": [] }
```

| Field | Meaning |
|-------|---------|
| `total_cost` | Billed spend in the loaded window |
| `attributed_cost` | Spend matched to a managed resource |
| `unallocated_cost` | Spend that matched no managed resource |
| `unmanaged_cost` | Spend for resources that no state manages |
| `coverage_percent` | Share of spend that was attributed |

After a sync it also carries `computed_at`, `matcher_version`, `window_start`, `window_end`, and `currency` (absent when the billing lines mix currencies). All zeros with `line_count: 0` means no billing source has loaded rows.

## `cost unmanaged --format=json`

```json
{ "results": [] }
```

Each entry has `provider`, `service_name`, `resource_id`, `resource_type`, `region_id`, `billed_cost`, `line_count`, `currency`.

## Per-state actuals (API only)

```bash
curl -s -H "Authorization: Bearer $STATEGRAPH_API_KEY" "$STATEGRAPH_API_BASE/api/v1/states/STATE_ID/costs/actuals"
```

```json
{ "billed_cost": "0", "line_count": 0, "resources": [] }
```

Each resource has `address`, `billed_cost`, `line_count`. With attribution run it adds `currency`, `computed_at`, `window_start`, `window_end`.

## `tx costs --tx ID` (text only)

```text
Costs:
  Totals:
    Monthly:  — → —   (— USD)
    Hourly:   — → —   (— USD)
    Resources: 9 → 4 (-5)
    Coverage: 0.0% (unchanged)
  Per state:
    acme-app: —/mo  (resources -5)
      - null_resource.web[2]                                +0.00
```

- Each `Totals` line is `current → planned (delta)`. A dash means that side has nothing priced. A change to priced infrastructure reads like `130.82 USD → 140.16 USD   (+9.34 USD)`.
- Per state, `+` added, `-` removed, `~` changed. `+0.00` means no price effect or no price. A touched state with no cost change shows `no cost change`.
- The same block prints under `stategraph tf plan`.
- The API form `GET /api/v1/tx/TX_ID/costs` returns `totals` (`current_*`, `planned_*`, `delta_*`), `states[]` with `resources[]` (`change_kind` of `added`, `removed`, `changed`), and `by_provider`, `by_type`, `by_tag`. `202 {"status":"computing"}` means not ready.

## `cost calculate --state ID` (text only)

```text
Cost calculation queued (task 55daf068-…). Re-run `stategraph cost state --state de783642-…` once it completes.
```

`GET /api/v1/tasks/TASK_ID` returns `{"id":"…","state":"pending"}` then `"completed"`, `"failed"`, or `"aborted"`.
