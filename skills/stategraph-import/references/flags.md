# Import command options

Every option below is from `stategraph <cmd> --help=plain`. `--api-base` and `--tenant` are on every server command; set `STATEGRAPH_API_BASE` and `STATEGRAPH_TENANT_ID` instead.

## stategraph import tf [OPTION]... [FILE]

Creates a state and imports a Terraform state file and the HCL of the current directory in one step, or with `--hcl` the HCL alone.

| Option | Description |
|---|---|
| `FILE` | Path to the Terraform state file. Required unless `--hcl` is given. |
| `--name VAL` | Name of the state to create. |
| `--workspace VAL` | Workspace for the state. Default `default`. Env `STATEGRAPH_WORKSPACE`. |
| `--group UUID` | Group ID for the state. |
| `--overwrite` | Overwrite an existing state, using the state ID from `stategraph.json`. With `--name` and no `stategraph.json`, creates a new state. The existing state's contents are replaced: anything committed to it that this import does not contain is deleted. Transaction history, cost and security data are kept. |
| `--hcl` | Import the HCL configuration only. FILE must be omitted. The target state's recorded resources and instances stay as they are. Its configuration, variable values, attached files, and revision hashes are replaced. |
| `--no-write-config` | Do not write `stategraph.json`. |
| `--var KEY=VALUE` | Set a variable. Repeatable. Recorded in the `tfvars` table with `file = 'tfvars'`. |
| `--var-file FILE` | Path to a variable file. |
| `--attach-files ADDRESS_GLOB=FILE_GLOB` | Attach files to resources matching an address glob, e.g. `'module.foo.*=**/*.yml'`. Repeatable. File globs for the same address glob are combined. |
| `--attach-dest ADDRESS_GLOB=DEST` | Destination directory for files attached at an address glob, e.g. `'module.foo.*=${path.module}/config'`. DEST supports `${path.module}` (the matched block's module dir) and `${path.root}` (the bundle root), each with an optional relative subdir. Without it, files go to the bundle root. Repeatable. |
| `--diagnostics[=MODE]` | Evaluate the HCL with tracing during the import. MODE: `rollup`, `on` (default for a bare flag), `detail`, `verbose`, `perf`. Env `STATEGRAPH_DIAGNOSTICS`. |
| `--batch-size INT` | Batch size for transaction log appends. Env `STATEGRAPH_TX_BATCH_SIZE`. |
| `--parallel-batches INT` | Parallel batches for transaction log appends. Env `STATEGRAPH_TX_PARALLEL_BATCHES`. |
| `-y` | Auto-confirm prompts. Env `STATEGRAPH_NON_INTERACTIVE`. |
| `--silent` | Suppress prompt output. Only with `-y`. Env `STATEGRAPH_SILENT`. |
| `-q`, `-v`, `--verbosity LEVEL` | Log verbosity. LEVEL: `quiet`, `error`, `warning`, `info`, `debug`. |

Output on success:

```text
Terraform state and HCL imported successfully for state <ID> (transaction <TX>)
```

With `--hcl`:

```text
HCL configuration imported successfully for state <ID> (transaction <TX>)
```

Every run prints a warning that Stategraph ignores the existing Terraform backend configuration. `--overwrite` adds a warning that the target's contents are replaced. `--hcl` adds a warning that only the HCL is imported. All three are informational.

## stategraph diagnostics run [OPTION]... [DIR]

Evaluates the HCL of a root module locally and traces the evaluation. No API call.

| Option | Description |
|---|---|
| `DIR` | Root module directory. Default: the current directory. |
| `--mode MODE` | `rollup`, `on`, `detail`, `verbose` (default), `perf`. |
| `--out FILE` | Write the trace to FILE. Without it, the trace goes to stderr. |
| `--workspace VAL` | Workspace to evaluate. Default `default`. Env `STATEGRAPH_WORKSPACE`. |
| `--var KEY=VALUE` | Set a variable. Repeatable. |
| `--var-file FILE` | Path to a variable file. |
| `--attach-files ADDRESS_GLOB=FILE_GLOB` | As on `import tf`. |
| `--attach-dest ADDRESS_GLOB=DEST` | As on `import tf`. |

Exits 1 with `Error loading HCL: ...` when the module does not load.

## stategraph hcl addresses

No options. Prints one address per line for the HCL in the current directory: resources, data sources, module calls, `var.*`, `local.*`, `output.*`, `provider.*`, and `terraform`. Exits 1 with `Error loading HCL: ...` when the module does not load.

## stategraph states import [OPTION]... FILE

Uploads a state file only. No HCL, no `stategraph.json`. Prints the created state as JSON.

| Option | Description |
|---|---|
| `FILE` | Path to the state file. Required. |
| `--name VAL` | Name of the state. Required. |
| `--workspace VAL` | Workspace. Default `default`. Env `STATEGRAPH_WORKSPACE`. |
| `--group UUID` | Create the state in a group. |
| `--tag KEY=VALUE` | Tag on the transaction. Repeatable. |
| `--tags JSON` | Tags as a JSON object. Takes precedence over `--tag`. |
| `-y`, `--silent` | As on `import tf`. |

## stategraph states anonymize [OPTION]... PATH

Prints an anonymized copy of a Terraform state JSON file to stdout. Local only.

| Option | Description |
|---|---|
| `PATH` | Path to the state file. Required. |
| `-s SEED`, `--seed SEED` | Seed for the anonymization RNG. Random when omitted. |

## stategraph states resolve [OPTION]...

Prints one state ID.

| Option | Description |
|---|---|
| `--name NAME` | Resolve by state name. Ignores `stategraph.json` and `--workspace`. |
| `--workspace WS` | Resolve through `stategraph.json` in the current directory. |
| `--state UUID` | State ID or group ID. |
| `--group` | Print the group ID instead. |

## stategraph states delete [OPTION]...

| Option | Description |
|---|---|
| `--state UUID` | State to delete. Without it, the state comes from `stategraph.json`. |
| `--workspace VAL` | Workspace. Default `default`. |
| `--auto-approve` | Skip the confirmation prompt. Without it and without a terminal the command prints `Aborted`. |

Prints `State <ID> deleted`.

## Tables for verification (`stategraph sql query`)

Filter every query with `WHERE state_id = '<ID>'`.

| Table | Columns |
|---|---|
| `states` | `id`, `name`, `workspace`, `group_id`, `tenant_id`, `schema_version`, `created_at`, `updated_at` |
| `resources` | `state_id`, `address`, `fq_address`, `module`, `mode`, `name`, `provider`, `type` |
| `instances` | `state_id`, `address`, `fq_address`, `fq_resource_address`, `resource_address`, `attributes`, `index_key`, `status` |
| `files` | `state_id`, `filepath`, `module_address`, `content_hash`, `mode`, `template_vars` |
| `tfvars` | `state_id`, `var_address`, `data`, `file` |
