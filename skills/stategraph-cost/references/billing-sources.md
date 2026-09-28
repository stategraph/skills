# Billing sources (FOCUS)

Read this when you add or repair a billing source, or explain why actuals are empty.

A billing source loads actual cloud spend from a FOCUS export (Parquet or CSV, globs allowed) that AWS, GCP, or Azure writes to a bucket. Stategraph syncs it on a schedule, attributes billed lines to managed resources, and serves the result through `cost attribution`, `cost unmanaged`, and the per-state actuals endpoint. Credentials resolve from the server environment (instance role, workload identity, `az login`, or the standard provider variables). No secrets go to Stategraph.

Tenant admins manage sources. A non-admin call prints `Forbidden: admin privileges required` and exits 1.

## Commands

All take `--tenant` or `STATEGRAPH_TENANT_ID`. All except `add` and `list` take `SOURCE_ID` as a positional argument. Only `list` has `--format`.

| Command | Flags | Prints |
|---------|-------|--------|
| `billing-source add` | `--provider aws\|gcp\|azure` (required), `--source-uri URI` (required), `--region R`, `--window-months N` (default 2), `--disabled` | the source as JSON |
| `billing-source list` | `--format=table\|json\|simple` | `results[]` |
| `billing-source update SOURCE_ID` | `--enabled=true\|false`, `--source-uri URI`, `--region R`, `--window-months N`; omitted fields keep their value | the source as JSON |
| `billing-source enable SOURCE_ID` | | the source as JSON |
| `billing-source disable SOURCE_ID` | | the source as JSON |
| `billing-source sync SOURCE_ID` | `--from YYYY-MM-DD` backfills from that date instead of the trailing window | `Sync queued for billing source SOURCE_ID` |
| `billing-source remove SOURCE_ID` | `--auto-approve` skips the confirmation prompt | `Billing source SOURCE_ID deleted` |

- `--window-months` is the trailing reload window per sync. `2` reloads the current and previous month.
- `--disabled` creates the source with scheduled sync off. `enable` or `update --enabled=true` turns it on.
- `remove` deletes the source and the billing rows it loaded.
- `sync` and `enable` exit 0 when queued. Confirm the result later with `list`.

## Source JSON

```json
{
  "id": "adc9a385-…", "tenant_id": "0e9354f5-…", "provider": "aws",
  "source_uri": "s3://bucket/prefix/data/**/*.parquet", "region": "us-east-1",
  "window_months": 2, "enabled": false,
  "created_at": "2026-09-28T18:45:43Z", "updated_at": "2026-09-28T18:45:43Z"
}
```

After a sync starts the entry also carries `last_status` (`running` while a sync runs, `ok` after one lands), `last_synced_at`, and `last_row_count`. `list` shows them as columns: `id provider source_uri region window_months enabled last_status last_synced_at last_row_count`.

## Why actuals are empty

Check in this order:

1. `billing-source list --format=json` returns `{ "results": [] }`: no source. Add one.
2. `enabled` is `false`: scheduled sync is off. `update SOURCE_ID --enabled=true`, then `sync SOURCE_ID`.
3. `last_status` is empty or `running`: no sync has finished. Wait, then `list` again.
4. `last_status` is not `ok` after the sync finished: the URI, region, or credentials are wrong. Fix with `update`, then `sync`.
5. `last_row_count` is 0: the export has not produced files yet. Provider exports take about a day to first land, then refresh daily.

## Provider export setup and URI shapes

| Provider | `--source-uri` | Export |
|----------|----------------|--------|
| AWS | `s3://bucket/prefix/<export>/data/**/*.parquet` | Billing and Cost Management > Data Exports > FOCUS 1.0 standard export (Parquet) to an S3 bucket. `--region` is the bucket region; omit it to resolve from the environment. |
| GCP | `gs://bucket/prefix/**/*.parquet` | Detailed billing export to BigQuery, then export the FOCUS view to a GCS bucket (Parquet or CSV). |
| Azure | `abfss://container@account.dfs.core.windows.net/path/**/*.parquet` | Cost Management > Exports > "Cost and usage details (FOCUS)" dataset (Parquet) to a storage account. |

Links from `stategraph cost billing-source add --help=plain`:

- AWS: https://docs.aws.amazon.com/cur/latest/userguide/dataexports-create-standard.html and https://docs.aws.amazon.com/cur/latest/userguide/table-dictionary-focus-1-0-aws.html
- GCP: https://cloud.google.com/billing/docs/how-to/export-data-bigquery-setup and https://cloud.google.com/blog/topics/cost-management/new-bigquery-cloud-billing-view-based-on-focus
- Azure: https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-improved-exports and https://learn.microsoft.com/en-us/azure/cost-management-billing/dataset-schema/cost-usage-details-focus

## API

| Method | Path |
|--------|------|
| `GET` | `/api/v1/tenants/{tenant_id}/billing-sources` |
| `POST` | `/api/v1/tenants/{tenant_id}/billing-sources` |
| `PUT` | `/api/v1/tenants/{tenant_id}/billing-sources/{id}` |
| `POST` | `/api/v1/tenants/{tenant_id}/billing-sources/{id}/sync` |
| `DELETE` | `/api/v1/tenants/{tenant_id}/billing-sources/{id}` |

A non-admin caller gets `403`. Read endpoints for any tenant member: `/api/v1/tenants/{tenant_id}/costs/attribution`, `/api/v1/tenants/{tenant_id}/costs/unmanaged`, `/api/v1/states/{state_id}/costs/actuals`.
