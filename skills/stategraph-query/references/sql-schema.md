# Stategraph SQL schema (condensed)

`stategraph sql schema --format=json` prints every table with column types and the row limits (`default_limit` 20, `max_limit` 1000). The tables below are the ones a query usually needs. Most tables carry `state_id`; join it to `states.id` for the state name.

## states

| Column | Type | Note |
|---|---|---|
| `id` | uuid | The value every `--state` flag takes |
| `name` | text | |
| `workspace` | text | Default `default` |
| `group_id` | uuid | |
| `tenant_id` | uuid | |
| `created_at`, `updated_at`, `deleted_at` | timestamptz | |
| `deleted_by` | uuid | |
| `schema_version` | integer | |

## resources

One row per resource block (`aws_vpc.main`, `module.x.aws_subnet.this`), across all instances.

| Column | Type | Note |
|---|---|---|
| `address` | text | `aws_vpc.main`, `module.private_subnets.aws_subnet.this` |
| `fq_address` | text | |
| `type` | text | `aws_s3_bucket` |
| `name` | text | The block name, `main` |
| `provider` | text | `provider["registry.opentofu.org/hashicorp/aws"]` |
| `module` | text | `module.private_subnets`; `NULL` in the root module |
| `mode` | text | `managed` or `data` |
| `state_id` | uuid | |

## instances

One row per instance (`aws_route_table.private[0]`). `resource_address` matches `resources.address` in the same `state_id`.

| Column | Type | Note |
|---|---|---|
| `address` | text | `aws_route_table.private[0]`; the value `blast-radius` takes |
| `resource_address` | text | `aws_route_table.private` |
| `fq_address`, `fq_resource_address` | text | |
| `attributes` | jsonb | `attributes->>'bucket'`, `attributes->'tags'->>'Name'`, `attributes#>>'{path,0,key}'`, `attributes::text LIKE '%x%'` |
| `dependencies` | text[] | Addresses this instance depends on; search with `dependencies::text LIKE '%x%'` |
| `index_key` | jsonb | `0` or `"key"` for `count` and `for_each`; `NULL` otherwise |
| `status` | text | |
| `deposed` | text | |
| `sensitive_attributes` | jsonb | |
| `create_before_destroy` | bool | |
| `schema_version`, `identity_schema_version` | integer | |
| `identity`, `private` | jsonb, text | |
| `state_id` | uuid | |

## outputs

| Column | Type | Note |
|---|---|---|
| `address` | text | |
| `name` | text | |
| `value` | jsonb | |
| `type` | jsonb | |
| `sensitive` | bool | |
| `state_id` | uuid | |

## providers

| Column | Type | Note |
|---|---|---|
| `name` | text | `provider["registry.opentofu.org/hashicorp/aws"]` |
| `state_id` | uuid | |

## hcl

One row per HCL block of the configuration.

| Column | Type | Note |
|---|---|---|
| `id` | text | Block id: `locals.common_tags`, `module.private_subnets`, `aws_vpc.main` |
| `fq_address` | text | |
| `module_address` | text | Module that holds the block; `''` in the root module |
| `module_source` | text | Source of a `module` block, `modules/s3_bucket`; `''` otherwise |
| `data` | jsonb | Block body |
| `refs` | text[] | Addresses the block references |
| `file_refs`, `hints`, `path_attrs` | jsonb | |
| `state_id` | uuid | |
| `created_at`, `updated_at` | timestamptz | |

## hcl_refs

One row per reference from a block to an address.

| Column | Type | Note |
|---|---|---|
| `id` | text | Referencing block, `module.private_subnets` |
| `ref` | text | Referenced address, `aws_vpc.main`, `data.terraform_remote_state.networking` |
| `attr_path` | text[] | Attribute read, `{id}` |
| `from_depends_on` | bool | |
| `index_kind` | text | |
| `index_val` | jsonb | |
| `is_bare` | bool | |
| `resolvable` | bool | |
| `state_id` | uuid | |

## files

| Column | Type | Note |
|---|---|---|
| `id` | text | |
| `filepath` | text | `.terraform.lock.hcl` |
| `content_hash` | text | |
| `mode` | integer | |
| `module_address` | text | |
| `template_vars` | text[] | |
| `state_id` | uuid | |

## security_scan_findings

Read them with `stategraph security findings list` (skill `stategraph-security`); the table is here for joins.

| Column | Type |
|---|---|
| `check_id` | text |
| `resource_fq_address` | text |
| `severity_base`, `severity_effective`, `severity_reason` | text |
| `is_internet_reachable` | bool |
| `internet_reachability_evidence` | jsonb |
| `blast_radius_resource_count` | integer |
| `blast_radius_modules` | text[] |
| `cross_state_refs` | uuid[] |
| `fingerprint` | text |
| `scan_id`, `first_seen_scan_id`, `resolved_scan_id` | uuid |
| `source_file` | text |
| `source_start_line`, `source_end_line` | integer |
| `workspace` | text |
| `state_id` | uuid |

## cost_snapshot_resources

Read costs with `stategraph cost` (skill `stategraph-cost`); the table is here for joins.

| Column | Type |
|---|---|
| `snapshot_id` | uuid |
| `address` | text |
| `type`, `provider`, `region` | text |
| `cloud_resource_id` | text |
| `monthly_cost`, `hourly_cost` | text |
| `supported`, `no_price` | bool |
| `components`, `tags` | jsonb |

## transactions

| Column | Type | Note |
|---|---|---|
| `id` | uuid | |
| `state` | text | `previewed`, `committed`, ... |
| `tags`, `params` | jsonb | |
| `tenant_id` | uuid | |
| `created_at`, `completed_at` | timestamptz | |
| `created_by`, `completed_by` | uuid | |

## Other tables

`check_entries`, `check_results`, `cost_snapshots`, `focus_billing_sources`, `revision_hashes`, `revision_tx_hashes`, `security_scans`, `tenants`, `tfvars`, `transaction_logs`, `transaction_output_chunks`, `transaction_plan_operations`, `transaction_subgraphs`, `users`. Run `stategraph sql schema --format=json` for their columns.

## Syntax that works

- `INNER`, `LEFT`, `RIGHT` joins with `ON`; table aliases (`FROM resources AS r`); column aliases; CTEs (`WITH x AS (...)`); `GROUP BY`, `HAVING`, `ORDER BY`, `LIMIT`.
- `=`, `<>`, `<`, `>`, `IN (...)`, `IS NULL`, `IS NOT NULL`, `LIKE`, `ILIKE`, `NOT LIKE`, `NOT ILIKE` with `%` and `_`.
- JSON: `->`, `->>`, `#>>'{a,0,b}'`, `@> '{"k":"v"}'::jsonb`. Casts: `dependencies::text`, `attributes::text`.

## Syntax that fails

- Comments (`--`, `/* */`): `QUERY_ERR`.
- `count(*) AS count`: `QUERY_ERR`. Use `AS total`.
- `SELECT DISTINCT`: `QUERY_ERR`. Use `GROUP BY`.
- `ANY(...)`: `FUNC_ACCESS_ERR`. `ARRAY[...]`: `Unknown column 'ARRAY'`.
- `attributes->'x'->0`: `operator does not exist: jsonb -> bigint`. Use `#>>'{x,0,key}'`.
- An unqualified column in a join: `Ambiguous column 'address'. Qualify with table name.`
- `--paginate` without `ORDER BY`: `Error: --paginate cannot page this query.`
