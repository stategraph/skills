# Command outputs in a refactor session

Every block below is the exact output of the command on a root module with resources `random_pet.app_name`, `random_integer.port`, `null_resource.web` (count 2), `local_file.app_config`, and a child module `module.db`. The refactor moves the two `random_*` resources into a new child module `./modules/naming`.

## `stategraph hcl addresses`

Runs locally, exit 0. One address per line: resources, module calls, outputs, variables, and `terraform` blocks, with module-local addresses prefixed by the module call.

Before the move:

```text
module.db.output.db_name
module.db.null_resource.db
module.db.random_id.suffix
module.db.random_password.admin
module.db.var.env
module.db.var.name
output.db_name
output.port
output.app_name
local_file.app_config
module.db
null_resource.web
random_integer.port
random_pet.app_name
var.environment
terraform
```

After the move, the two resources appear as `module.naming.random_integer.port` and `module.naming.random_pet.app_name`, and `module.naming`, `module.naming.output.app_name`, `module.naming.output.port`, and `module.naming.terraform` are new. Compare the two lists: every removed resource address must have a counterpart under the new module prefix.

## `stategraph refactor start`

```bash
mkdir -p .stategraph && stategraph refactor start
```

Silent, exit 0. Writes `.stategraph/refactor.json`. A second `start` in an open session also exits 0.

Without the directory:

```text
Error: `Start_write_err ("Sys_error(\".stategraph/refactor.json: No such file or directory\")")
```

Exit 1.

## `stategraph refactor step`

Move into a new module (two resources moved). Exit 0. The mappings are recorded but not printed. The new blocks are listed:

```text
New entries:
  added: module.naming.terraform
  added: module.naming.output.port
  added: module.naming.output.app_name
  added: module.naming
```

Rename detected (`null_resource.web` to `null_resource.frontend`, with a `count` change in the same edit). Exit 0:

```text
null_resource.web -> null_resource.frontend
```

Nothing changed since the last step. Exit 0:

```text
No changes detected.
```

Ambiguous: `random_pet.app_name` removed, `random_pet.a` and `random_pet.b` added with the same attributes. Exit 1. stdout:

```text
New entries:
  added: random_pet.b
  added: random_pet.a
```

stderr:

```text
Unable to determine mapping for removed entries:
  removed: random_pet.app_name
```

Explicit mapping. Exit 0. The mapping is recorded, the remaining new block is listed:

```bash
stategraph refactor step random_pet.app_name=random_pet.a
```

```text
New entries:
  added: random_pet.b
```

Several mappings go in one call: `stategraph refactor step OLD1=NEW1 OLD2=NEW2`.

No session open (after `complete` or `abort`, or `start` never ran). Exit 1:

```text
Error: `Step_read_err ("Sys_error(\".stategraph/refactor.json: No such file or directory\")")
```

## `stategraph tf plan --detailed-exitcode --skip-costs --skip-security --out plan.json`

Moves only. Exit 2. stderr:

```text
WARNING: Running in refactor mode.
```

stdout:

```text
No changes. Your infrastructure matches the configuration.

Stategraph has compared your real infrastructure against your configuration and found no differences, so no changes are needed.
```

`plan.json` is written. The saved plan carries the moves, which is why `--detailed-exitcode` returns 2.

Ambiguous move. Exit 1, no `plan.json`. stderr:

```text
WARNING: Running in refactor mode.
Refactor step has unmatched entries.
  added: module.naming.random_pet.b
  added: module.naming.random_pet.a
  removed: module.naming.random_pet.app_name
Run 'stategraph refactor step' to resolve.
Error: command failed with an unhandled error (Refactor_step_unmatched_err).
```

Resolve with `stategraph refactor step module.naming.random_pet.app_name=module.naming.random_pet.a` and plan again.

Real diff (a resource with no counterpart in the state). Exit 2 with the plan file written. stdout ends with:

```text
Plan: 1 to add, 0 to change, 0 to destroy.
```

Do not apply. Fix the HCL and plan again.

HCL evaluates the same as the stored configuration (nothing moved). Exit 0:

```text
No changes detected.
```

Without `--out`, the plan is a preview. It still runs a step and prints the diff, then:

```text
Note: You didn't use the --out option to save this plan, so Stategraph can't apply it. Re-run with --out FILE to save a plan you can apply with "stategraph apply".
```

## `stategraph tf show --json plan.json`

Exit 0. The moves are in `resource_changes[]`: a moved resource has `previous_address` set and `change.actions` equal to `["no-op"]`.

```bash
stategraph tf show --json plan.json | jq -r '.resource_changes[] | select(.previous_address != null) | "\(.previous_address) -> \(.address) \(.change.actions)"'
```

```text
random_integer.port -> module.naming.random_integer.port ["no-op"]
random_pet.app_name -> module.naming.random_pet.app_name ["no-op"]
```

```bash
stategraph tf show --json plan.json | jq '[.resource_changes[] | select(.change.actions != ["no-op"])] | length'
```

```text
0
```

Any other number: a real diff. Do not apply.

`stategraph tf show plan.json` without `--json` prints the same text as the plan, without the refactor warning.

## `stategraph tf apply plan.json`

Exit 0. stderr:

```text
WARNING: Running in refactor mode.
WARNING: Running in refactor mode. Finalizing refactor session.
```

stdout:

```text
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:

port = 54310
db_name = faithful-kitten-staging-56af
app_name = faithful-kitten
```

Afterwards `moved_stategraph.tf` exists in the module directory, `.stategraph/refactor.json` is gone (the empty directory stays), and the state holds the new addresses:

```hcl
moved {
  from = random_pet.app_name
  to   = module.naming.random_pet.app_name
}

moved {
  from = random_integer.port
  to   = module.naming.random_integer.port
}
```

## `stategraph refactor complete`

`tf apply` runs this for you. Run by hand only when the user wants the moved blocks file without an apply. Exit 0:

```text
Wrote moved_stategraph.tf
```

It ends the session. With no recorded mapping it writes an empty file. The state addresses do not change until a plan with the moved blocks is applied.

## `stategraph refactor abort`

Silent, exit 0. Deletes `.stategraph/refactor.json`. With no session open, exit 1:

```text
Error: `Abort_delete_err ("Sys_error(\".stategraph/refactor.json: No such file or directory\")")
```

## After apply

```bash
stategraph tf plan --detailed-exitcode --skip-costs --skip-security
```

```text
No changes detected.
```

Exit 0. An empty `.stategraph/` directory does not put the plan in refactor mode.

```bash
STATE_ID=$(stategraph states resolve)
stategraph sql query "SELECT address, module FROM resources WHERE state_id = '$STATE_ID' ORDER BY address"
```

```text
address                            module
---------------------------------  -------------
local_file.app_config
module.db.null_resource.db         module.db
module.db.random_id.suffix         module.db
module.db.random_password.admin    module.db
module.naming.random_integer.port  module.naming
module.naming.random_pet.app_name  module.naming
null_resource.web
```

Root resources have `module` null. To list what stayed at the root:

```bash
stategraph sql query "SELECT address FROM resources WHERE module IS NULL AND state_id = '$STATE_ID' ORDER BY address" --format=json
```

`sql query` returns one page of 20 rows unless the query has `LIMIT` (maximum 1000).
