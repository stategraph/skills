---
name: stategraph-query
description: |
  Read-only queries and analysis of the infrastructure stored in Stategraph, with the `stategraph` CLI: SQL over states, resources, instances, outputs, providers, modules, and HCL references; state summaries and inventory; blast radius (what a change to one resource reaches); gap analysis of cloud resources that no state manages.

  Use this skill for requests like:
  - "what states do we have?", "summarize acme-networking", "how many resources per type?", "give me an overview of the infrastructure"
  - "list all S3 buckets and whether versioning is on", "which instances are t3.micro?", "which resources are tagged Environment=production?", "which security groups allow 0.0.0.0/0?"
  - "what modules do we use and where?", "what outputs does the platform state expose?", "which states read the networking state?"
  - "what depends on aws_vpc.main?", "what would a change to the database subnet group affect?", "blast radius of X"
  - "which AWS resources are not managed by Terraform?", "run gap analysis"

  Do not use it for plan, apply, delete, or transactions (stategraph-change), importing state or HCL (stategraph-import), refactor sessions (stategraph-refactor), cost questions (stategraph-cost), or security findings and scans (stategraph-security).
tags:
  - stategraph
  - sql
  - inventory
  - blast-radius
  - gap-analysis
metadata:
  author: Stategraph
  version: "3.0"
---

# Stategraph query skill

Every command here is read-only. Run them without confirmation.

## Connection

`STATEGRAPH_API_BASE` replaces `--api-base`, `STATEGRAPH_API_KEY` is the bearer token, and `STATEGRAPH_TENANT_ID` replaces `--tenant`. When an env var is set, omit its flag. `--state` has no env var and takes the state id (UUID), never the name. Flags go after the subcommand. Add `--format=json` to every command whose output you parse.

Exit codes: 0 ok, 1 runtime error (API error, bad query, unknown state), 123 error on stderr, 124 command-line parse error (a flag that does not exist), 125 internal error. Read stderr before any second attempt.

## Get ids

```bash
stategraph info --format=json                       # tenants in .results[]; skip when STATEGRAPH_TENANT_ID is set
stategraph states list --format=json                # .results[]: id, name, workspace, group_id, created_at
stategraph states resolve --name acme-networking    # prints the state id
```

Unknown name: `Error: No state with workspace 'NAME' found for NAME`, exit 1. Unknown id on any `--state` command: `State not found`, exit 1.

## State summaries

```bash
stategraph states summary --state STATE_ID --format=json            # edges, instances, modules, providers, resources
stategraph states resources summary --state STATE_ID --format=json  # {"aws_subnet": {"instances": 6}, ...}
stategraph states modules list --state STATE_ID --format=json       # .results[]: name, resource_count, instance_count
```

For a tenant overview, run `states list`, then `states resources summary` for each state.

## SQL

```bash
stategraph sql query "SELECT ..." --format=json
```

- There is no `--state` and no `--tenant` flag. A query covers every state in the tenant. Scope with `WHERE state_id = 'STATE_ID'`.
- Without `LIMIT`, the result is the first 20 rows plus a stderr note `Note: these are the first 20 rows ...`. That page is not the full result. Add `LIMIT 1000` (the maximum) or `--paginate` with an `ORDER BY`. `--paginate` without `ORDER BY` fails, exit 1.
- No comments in the query. `count(*)` needs an alias other than `count`: use `AS total`. Qualify every column in a join (`r.type`), or the query fails with `Ambiguous column`.
- `type`, `module`, `provider`, `mode` are on `resources`; `attributes`, `dependencies`, `index_key` are on `instances`. Join with `i.resource_address = r.address AND i.state_id = r.state_id`.
- Root-module resources have `module IS NULL`.
- JSON: `attributes->>'key'`, `attributes->'tags'->>'Name'`, nested and array paths with `attributes#>>'{versioning_configuration,0,status}'`. `->0` fails.
- Unsupported: `DISTINCT` (use `GROUP BY`), `ANY`, `ARRAY[...]`. Search arrays or whole attributes with `dependencies::text LIKE '%x%'` and `attributes::text LIKE '%x%'`.

Table and column names: `references/sql-schema.md`. Read it before writing a query on a table not used below.

Queries for common questions (replace `STATE_ID`):

```bash
# states and ids
stategraph sql query "SELECT id, name, workspace FROM states ORDER BY name" --format=json
# resource count by type, whole tenant
stategraph sql query "SELECT type, count(*) AS total FROM resources GROUP BY type ORDER BY count(*) DESC LIMIT 1000" --format=json
# resource count per state
stategraph sql query "SELECT s.name AS state, count(*) AS total FROM resources AS r INNER JOIN states AS s ON r.state_id = s.id GROUP BY s.name ORDER BY s.name" --format=json
# every resource in one state
stategraph sql query "SELECT address, type, module FROM resources WHERE state_id = 'STATE_ID' ORDER BY address LIMIT 1000" --format=json
# which state holds an address
stategraph sql query "SELECT s.name AS state, r.address, r.type FROM resources AS r INNER JOIN states AS s ON r.state_id = s.id WHERE r.address = 'aws_vpc.main'" --format=json
# instances of a type with an attribute: S3 buckets
stategraph sql query "SELECT i.state_id, i.address, i.attributes->>'bucket' AS bucket FROM instances AS i INNER JOIN resources AS r ON i.resource_address = r.address AND i.state_id = r.state_id WHERE r.type = 'aws_s3_bucket' ORDER BY i.address LIMIT 1000" --format=json
# versioning status per bucket, from the aws_s3_bucket_versioning resource
stategraph sql query "SELECT i.address, i.attributes->>'bucket' AS bucket, i.attributes#>>'{versioning_configuration,0,status}' AS versioning FROM instances AS i INNER JOIN resources AS r ON i.resource_address = r.address AND i.state_id = r.state_id WHERE r.type = 'aws_s3_bucket_versioning' ORDER BY i.address LIMIT 1000" --format=json
# instances with a tag value
stategraph sql query "SELECT s.name AS state, i.address FROM instances AS i INNER JOIN states AS s ON i.state_id = s.id WHERE i.attributes->'tags'->>'Environment' = 'production' ORDER BY s.name, i.address LIMIT 1000" --format=json
# module inventory: modules per state with resource counts
stategraph sql query "SELECT s.name AS state, r.module, count(*) AS total FROM resources AS r INNER JOIN states AS s ON r.state_id = s.id WHERE r.module IS NOT NULL GROUP BY s.name, r.module ORDER BY s.name, r.module LIMIT 1000" --format=json
# module sources
stategraph sql query "SELECT s.name AS state, h.module_address, h.module_source FROM hcl AS h INNER JOIN states AS s ON h.state_id = s.id WHERE h.module_source <> '' GROUP BY s.name, h.module_address, h.module_source ORDER BY s.name, h.module_address LIMIT 1000" --format=json
# blocks that read an address, for example a remote state (cross-state references)
stategraph sql query "SELECT s.name AS state, h.id AS block, h.attr_path FROM hcl_refs AS h INNER JOIN states AS s ON h.state_id = s.id WHERE h.ref = 'data.terraform_remote_state.networking' GROUP BY s.name, h.id, h.attr_path ORDER BY s.name, h.id LIMIT 1000" --format=json
# outputs of every state
stategraph sql query "SELECT s.name AS state, o.name, o.value FROM outputs AS o INNER JOIN states AS s ON o.state_id = s.id ORDER BY s.name, o.name LIMIT 1000" --format=json
```

More queries, with their output shapes (direct dependents, cross-state references, open ingress rules, providers, data sources, attribute text search): `references/queries.md`.

## Instances query (filter terms, no SQL)

```bash
stategraph states instances query "type:aws_s3_bucket" --state STATE_ID --format=json
stategraph states instances query -i "type:aws_subnet and not module:module.public_subnets" --state STATE_ID --format=json
```

Terms are `key:value` with keys `type`, `module`, `address`, `resource_address`, and `attr:NAME:VALUE` for a top-level string attribute. Combine with `and`, `or`, `not`, and parentheses. `-i` reads all pages. SQL syntax such as `type = 'x'` fails with `Error: UNKNOWN_TAG`. Output: `.results[]` with `address`, `type`, `attributes`, `dependencies`, `provider`, `sensitive_attributes`.

## Blast radius

```bash
stategraph states instances blast-radius INSTANCE_ADDRESS --state STATE_ID --format=json
```

- `INSTANCE_ADDRESS` is an instance address. For a resource with `count` or `for_each`, use the indexed form (`aws_route_table.private[0]`, `module.private_subnets.aws_subnet.this[0]`). The unindexed address of such a resource, or a missing address, returns `total_nodes: 0` with exit 0 and an empty table. Get the instance addresses first: `stategraph sql query "SELECT address FROM instances WHERE state_id = 'STATE_ID' AND resource_address = 'aws_route_table.private' ORDER BY address" --format=json`.
- Table rows: `address`, `kind`, `state`, `instances`, `depends_on`, nearest first. `*` marks the queried block. `state` can differ from the queried state: the graph follows references into linked states.
- JSON: `nodes` (`address`, `kind`, `resource_type`, `module_address`, `state_name`, `instance_addresses`, `is_seed`), `edges` (`from_id`, `to_id`, `kind`, `attr_path`), `root_id`, `total_nodes`, `truncated`.
- `--max-depth N` (default and maximum 100) and `--limit N` (default and maximum 1000). When they cut the graph, stderr says `warning: N of M blocks are shown` and `truncated` is true.
- Direct dependents only, without the graph: `stategraph sql query "SELECT address FROM instances WHERE state_id = 'STATE_ID' AND dependencies::text LIKE '%aws_vpc.main%' ORDER BY address LIMIT 1000" --format=json`.

## Gap analysis (unmanaged cloud resources)

```bash
stategraph tenant gaps analyze --provider aws --format=json
```

The scan runs on the server in the background. The first call returns `{"started_at": "...", "status": "running"}` and no resources. Wait about 30 seconds and run the same command again, until the response has `summary` (`total_cloud_resources`, `managed_by_stategraph`, `unmanaged`) and `unmanaged_resources[]` (`id`, `arn`, `service`, `resource_type`, `region`). Later calls read a cache. `--source no-cache` starts a fresh scan and returns `running` again; run the command without it for the result.

When the response is still `running` after three tries, or has a non-empty `error`, run `stategraph tenant gaps config --provider aws`. On a server where the provider is not set up, it fails with `Error: API call failed: Conversion_err (... "status":"not_configured" ... "warnings":[{"code":"NO_REGION", "message": ..., "fix": ...`. Report that `message` and `fix` once and stop. Credentials are configured on the server, not with the CLI.

## Security

Findings, scans, posture, and change impact belong to `stategraph-security`. Attribute-level SQL with a security flavor (open ingress rules, public buckets) stays here.

## Report

State the command, the scope (tenant, or state name and id), the key result, any caveat (a truncated page, a cut graph, a running scan), and the next useful read-only command. Do not run plan, apply, delete, or import from this skill; route to the owning skill.
