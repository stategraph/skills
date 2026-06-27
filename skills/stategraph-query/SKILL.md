---
name: stategraph-query
description: |
  Query and read-only analysis skill for Stategraph.

  Use this skill for:
  - SQL queries
  - state summaries
  - module, resource, provider, and output listings
  - infrastructure inventory
  - security and compliance inspection using read-only queries
  - blast radius and dependency analysis
  - gap analysis for unmanaged resources

  Do not use this skill for:
  - plan/apply/delete workflows
  - importing state or HCL
  - refactor sessions

tags:
  - stategraph
  - sql
  - inventory
  - compliance
  - security
metadata:
  author: Stategraph
  version: "2.0"
---

# Stategraph query skill

## Purpose

This skill handles all read-only Stategraph workflows.

## Read-only commands

These commands may run without confirmation:

```bash
stategraph info
stategraph sql schema
stategraph sql query "..."
stategraph states list --tenant TENANT_ID
stategraph states summary --state STATE_ID
stategraph states resources summary --state STATE_ID
stategraph states modules list --state STATE_ID
stategraph states instances blast-radius ADDRESS --state STATE_ID
stategraph states instances query QUERY --state STATE_ID
stategraph tenant gaps analyze --tenant TENANT_ID --provider aws
stategraph tenant gaps config --tenant TENANT_ID --provider aws
```

## Required inputs

Resolve only the inputs required for the query:

* `TENANT_ID` for tenant-scoped operations
* `STATE_ID` for state-scoped operations
* SQL text for `sql query`
* resource address for blast radius
* provider for gap analysis

Do not gather extra context the command does not need.

## Canonical command ladder

### Orientation

Start here only when IDs are missing.

```bash
stategraph info
```

### Schema discovery

Use when table or column names are uncertain.

```bash
stategraph sql schema
stategraph sql schema --format json
```

Do not run schema discovery for every query by default. Use it when there is uncertainty.

### Inventory and summaries

```bash
stategraph states list --tenant TENANT_ID
stategraph states summary --state STATE_ID
stategraph states resources summary --state STATE_ID
stategraph states modules list --state STATE_ID
```

### SQL

Canonical form:

```bash
stategraph sql query "SELECT * FROM resources" --state STATE_ID
```

Notes:

* `resources` is the usual table for filtering by `type` or `module`
* `instances` is where instance attributes live
* `modules` is not a queryable table
* `LIKE` and `ILIKE` (and `NOT LIKE` / `NOT ILIKE`) are supported for pattern matching
* use `=` for exact matches

### Blast radius

Canonical form:

```bash
stategraph states instances blast-radius ADDRESS --state STATE_ID
```

### Gap analysis

Canonical form:

```bash
stategraph tenant gaps analyze --tenant TENANT_ID --provider aws
```

## Common patterns

### Inventory

```bash
stategraph states summary --state STATE_ID
stategraph states resources summary --state STATE_ID
stategraph sql query "SELECT * FROM resources" --state STATE_ID
```

### Resource type filtering

```bash
stategraph sql query "SELECT * FROM resources WHERE type = 'aws_s3_bucket'" --state STATE_ID
stategraph sql query "SELECT * FROM resources WHERE type = 'aws_iam_role'" --state STATE_ID
stategraph sql query "SELECT * FROM resources WHERE type = 'aws_security_group'" --state STATE_ID
```

### Module inspection

```bash
stategraph states modules list --state STATE_ID
stategraph sql query "SELECT * FROM resources WHERE module = 'module.vpc'" --state STATE_ID
```

### Instance attribute drill-down

```bash
stategraph sql query "SELECT * FROM instances WHERE address = 'aws_s3_bucket.my_bucket'" --state STATE_ID
```

### Blast radius

```bash
stategraph states instances blast-radius aws_vpc.main --state STATE_ID
```

### Gap analysis

```bash
stategraph tenant gaps analyze --tenant TENANT_ID --provider aws
```

## Security/compliance inspection guidance

This skill is read-only. It can identify likely issues, not certify compliance.

Useful entry-point queries:

```bash
stategraph sql query "SELECT * FROM resources WHERE type = 'aws_s3_bucket'" --state STATE_ID
stategraph sql query "SELECT * FROM resources WHERE type = 'aws_security_group'" --state STATE_ID
stategraph sql query "SELECT * FROM resources WHERE type = 'aws_iam_role'" --state STATE_ID
stategraph sql query "SELECT * FROM resources WHERE type = 'aws_iam_policy'" --state STATE_ID
stategraph sql query "SELECT * FROM resources WHERE type = 'aws_ebs_volume'" --state STATE_ID
stategraph sql query "SELECT * FROM resources WHERE type = 'aws_rds_instance'" --state STATE_ID
```

When the user needs attribute-level inspection, query `instances` by address.

## Response contract

For every result, report:

1. the command run
2. the scope used: tenant or state
3. the key result
4. any caveat such as "resource attributes require querying instances"
5. the next useful read-only command

## Failure handling

### Missing tenant or state

Use `stategraph info`, then list tenants or states only as needed.

### Query fails due to schema uncertainty

Run:

```bash
stategraph sql schema --format json
```

Then rewrite the query using exact table and column names.

### Query returns too much data

Narrow by:

* `--state STATE_ID`
* resource `type`
* `module`
* exact `address`

### User asks for mutation from inside this skill

Do not execute plan/apply/delete/import here. Route to `stategraph-change` or `stategraph-import`.
