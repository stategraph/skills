# Queries for common questions

Replace `STATE_ID` with a state id from `stategraph states list --format=json`. Add `--format=json` to parse the output. Each example shows the shape of its rows.

## States and ids

```bash
stategraph sql query "SELECT id, name, workspace FROM states ORDER BY name" --format=json
```

```json
[ { "id": "8f1af0b8-...", "name": "acme-networking", "workspace": "default" } ]
```

## Resource count by type, whole tenant

```bash
stategraph sql query "SELECT type, count(*) AS total FROM resources GROUP BY type ORDER BY count(*) DESC LIMIT 1000" --format=json
```

```json
[ { "type": "aws_cloudwatch_metric_alarm", "total": 30 }, { "type": "aws_sqs_queue", "total": 14 } ]
```

## Resource count by type in one state

```bash
stategraph sql query "SELECT type, count(*) AS total FROM resources WHERE state_id = 'STATE_ID' GROUP BY type ORDER BY count(*) DESC LIMIT 1000" --format=json
```

`stategraph states resources summary --state STATE_ID --format=json` gives the same counts per instance.

## Resource count per state

```bash
stategraph sql query "SELECT s.name AS state, count(*) AS total FROM resources AS r INNER JOIN states AS s ON r.state_id = s.id GROUP BY s.name ORDER BY s.name" --format=json
```

```json
[ { "state": "acme-compute", "total": 40 }, { "state": "acme-networking", "total": 16 } ]
```

## Managed resources and data sources

```bash
stategraph sql query "SELECT mode, count(*) AS total FROM resources GROUP BY mode ORDER BY mode" --format=json
```

```json
[ { "mode": "data", "total": 19 }, { "mode": "managed", "total": 223 } ]
```

## Every resource in one state

```bash
stategraph sql query "SELECT address, type, module FROM resources WHERE state_id = 'STATE_ID' ORDER BY address LIMIT 1000" --format=json
```

```json
[ { "address": "aws_vpc.main", "type": "aws_vpc", "module": null },
  { "address": "module.private_subnets.aws_subnet.this", "type": "aws_subnet", "module": "module.private_subnets" } ]
```

## Which state holds an address

```bash
stategraph sql query "SELECT s.name AS state, r.address, r.type FROM resources AS r INNER JOIN states AS s ON r.state_id = s.id WHERE r.address = 'aws_vpc.main'" --format=json
```

```json
[ { "state": "acme-networking", "address": "aws_vpc.main", "type": "aws_vpc" } ]
```

## Instance addresses of a resource (input for blast-radius)

```bash
stategraph sql query "SELECT address FROM instances WHERE state_id = 'STATE_ID' AND resource_address = 'aws_route_table.private' ORDER BY address" --format=json
```

```json
[ { "address": "aws_route_table.private[0]" }, { "address": "aws_route_table.private[1]" } ]
```

## Instances of a type with an attribute

S3 buckets and their bucket name:

```bash
stategraph sql query "SELECT i.state_id, i.address, i.attributes->>'bucket' AS bucket FROM instances AS i INNER JOIN resources AS r ON i.resource_address = r.address AND i.state_id = r.state_id WHERE r.type = 'aws_s3_bucket' ORDER BY i.address LIMIT 1000" --format=json
```

```json
[ { "state_id": "eafdebee-...", "address": "module.assets_bucket.aws_s3_bucket.this", "bucket": "acme-assets-production" } ]
```

Versioning status per bucket. The `aws_s3_bucket` row has `attributes->'versioning'` = `[{"enabled": false, "mfa_delete": false}]` on every bucket; the status set by the `aws_s3_bucket_versioning` resource is on that resource:

```bash
stategraph sql query "SELECT i.address, i.attributes->>'bucket' AS bucket, i.attributes#>>'{versioning_configuration,0,status}' AS versioning FROM instances AS i INNER JOIN resources AS r ON i.resource_address = r.address AND i.state_id = r.state_id WHERE r.type = 'aws_s3_bucket_versioning' ORDER BY i.address LIMIT 1000" --format=json
```

```json
[ { "address": "module.assets_bucket.aws_s3_bucket_versioning.this", "bucket": "acme-assets-production", "versioning": "Enabled" },
  { "address": "module.logs_bucket.aws_s3_bucket_versioning.this", "bucket": "acme-logs-production", "versioning": "Suspended" } ]
```

EC2 instances with their instance type and Name tag:

```bash
stategraph sql query "SELECT i.address, i.attributes->>'instance_type' AS instance_type, i.attributes#>>'{tags,Name}' AS name FROM instances AS i INNER JOIN resources AS r ON i.resource_address = r.address AND i.state_id = r.state_id WHERE r.type = 'aws_instance' ORDER BY i.address LIMIT 1000" --format=json
```

```json
[ { "address": "aws_instance.bastion", "instance_type": "t3.micro", "name": "acme-bastion" } ]
```

## Instances with a tag value

```bash
stategraph sql query "SELECT s.name AS state, i.address FROM instances AS i INNER JOIN states AS s ON i.state_id = s.id WHERE i.attributes->'tags'->>'Environment' = 'production' ORDER BY s.name, i.address LIMIT 1000" --format=json
```

`WHERE i.attributes->'tags' @> '{"Environment":"production"}'::jsonb` returns the same rows.

## Security groups open to the internet

Ingress rules with `cidr_ipv4` `0.0.0.0/0`:

```bash
stategraph sql query "SELECT s.name AS state, i.address, i.attributes->>'cidr_ipv4' AS cidr, i.attributes->>'from_port' AS from_port, i.attributes->>'to_port' AS to_port FROM instances AS i INNER JOIN resources AS r ON i.resource_address = r.address AND i.state_id = r.state_id INNER JOIN states AS s ON i.state_id = s.id WHERE r.type = 'aws_vpc_security_group_ingress_rule' AND i.attributes->>'cidr_ipv4' = '0.0.0.0/0' ORDER BY s.name, i.address LIMIT 1000" --format=json
```

```json
[ { "state": "acme-platform", "address": "aws_vpc_security_group_ingress_rule.bastion_ssh", "cidr": "0.0.0.0/0", "from_port": "22", "to_port": "22" } ]
```

Any instance whose attributes mention a string, whatever the type:

```bash
stategraph sql query "SELECT s.name AS state, i.address FROM instances AS i INNER JOIN states AS s ON i.state_id = s.id WHERE i.attributes::text LIKE '%0.0.0.0/0%' ORDER BY s.name, i.address LIMIT 1000" --format=json
```

## Direct dependents of an instance

`dependencies` lists the addresses an instance depends on. This finds the instances one hop away; `stategraph states instances blast-radius` follows the whole graph.

```bash
stategraph sql query "SELECT address FROM instances WHERE state_id = 'STATE_ID' AND dependencies::text LIKE '%aws_vpc.main%' ORDER BY address LIMIT 1000" --format=json
```

```json
[ { "address": "aws_internet_gateway.main" }, { "address": "aws_nat_gateway.main[0]" } ]
```

## Module inventory

Modules per state with resource counts:

```bash
stategraph sql query "SELECT s.name AS state, r.module, count(*) AS total FROM resources AS r INNER JOIN states AS s ON r.state_id = s.id WHERE r.module IS NOT NULL GROUP BY s.name, r.module ORDER BY s.name, r.module LIMIT 1000" --format=json
```

```json
[ { "state": "acme-data-stores", "module": "module.assets_bucket", "total": 4 } ]
```

Module sources (the `hcl` table has one row per block, so group the rows):

```bash
stategraph sql query "SELECT s.name AS state, h.module_address, h.module_source FROM hcl AS h INNER JOIN states AS s ON h.state_id = s.id WHERE h.module_source <> '' GROUP BY s.name, h.module_address, h.module_source ORDER BY s.name, h.module_address LIMIT 1000" --format=json
```

```json
[ { "state": "acme-data-stores", "module_address": "module.assets_bucket", "module_source": "modules/s3_bucket" } ]
```

`stategraph states modules list --state STATE_ID --format=json` lists the modules of one state with resource and instance counts.

## Outputs of every state

```bash
stategraph sql query "SELECT s.name AS state, o.name, o.value FROM outputs AS o INNER JOIN states AS s ON o.state_id = s.id ORDER BY s.name, o.name LIMIT 1000" --format=json
```

```json
[ { "state": "acme-networking", "name": "vpc_id", "value": "vpc-4061fe7110a0e1dde" } ]
```

## Providers per state

```bash
stategraph sql query "SELECT s.name AS state, p.name AS provider FROM providers AS p INNER JOIN states AS s ON p.state_id = s.id ORDER BY s.name, p.name LIMIT 1000" --format=json
```

```json
[ { "state": "acme-networking", "provider": "provider[\"registry.opentofu.org/hashicorp/aws\"]" } ]
```

## Cross-state references

Which states read another state through `terraform_remote_state`, and how many references each has:

```bash
stategraph sql query "SELECT s.name AS state, h.ref, count(*) AS total FROM hcl_refs AS h INNER JOIN states AS s ON h.state_id = s.id WHERE h.ref LIKE 'data.terraform_remote_state.%' GROUP BY s.name, h.ref ORDER BY s.name, h.ref LIMIT 1000" --format=json
```

```json
[ { "state": "acme-compute", "ref": "data.terraform_remote_state.networking", "total": 3 } ]
```

## Whole result with --paginate

`--paginate` follows every page. It needs an `ORDER BY` on a selected column; `LIMIT` then sets the page size, not a cap.

```bash
stategraph sql query --paginate "SELECT address FROM resources ORDER BY address" --format=json
```
