# Command reference for stategraph-change

Every command takes `--api-base URL` (env `STATEGRAPH_API_BASE`) and authenticates with `STATEGRAPH_API_KEY`. `--loggers`, `-q`, `-v`, and `--verbosity LEVEL` (`quiet`, `error`, `warning`, `info`, `debug`) exist on every command and are omitted below.

## stategraph tf plan

Run from the root module directory. Compares local HCL with the current state, opens a transaction, refreshes state, and prints the diff.

| Flag | Env | Meaning |
|---|---|---|
| `--tenant UUID` | `STATEGRAPH_TENANT_ID` | Required |
| `--workspace VAL` | `STATEGRAPH_WORKSPACE` | Default `default` |
| `--state UUID` | | State id. Default: read from `stategraph.json` |
| `--out FILE` | | Plan file for `tf apply`. Without it: read-only preview, no file, transient transaction |
| `--detailed-exitcode` | | Exit 2 when the plan has changes, 0 when it has none |
| `--var KEY=VALUE` | | Repeatable, as in `terraform plan` |
| `--var-file PATH` | | As in `terraform plan` |
| `--force GLOB` | | Add a node address to the plan, for example `data.*` or `module.networking.*` |
| `--skip-refresh` | | Pass `-refresh=false` to the underlying plan |
| `--skip-data-source-refresh` | | Do not open a transaction when the configuration has not changed |
| `--skip-costs` | | No cost preview. Rejected together with `--costs-wait` |
| `--costs-wait SECONDS` | `STATEGRAPH_COSTS_WAIT_SECONDS` | Wait for the cost delta. Default 3 |
| `--skip-security` | | No security preview. Rejected together with `--security-wait` |
| `--security-wait SECONDS` | `STATEGRAPH_SECURITY_WAIT_SECONDS` | Wait for the security impact. Default 3 |
| `--tx-tags JSON` | `STATEGRAPH_TX_TAGS` | Tags on the transaction this plan creates |
| `--diagnostics[=MODE]` | `STATEGRAPH_DIAGNOSTICS` | HCL evaluation tracing: `rollup`, `on`, `detail`, `verbose`, `perf` |
| `--batch-size INT` | `STATEGRAPH_TX_BATCH_SIZE` | Transaction log append batch size |
| `--parallel-batches INT` | `STATEGRAPH_TX_PARALLEL_BATCHES` | Parallel batches for log appends |

Output with changes:

```text
Stategraph will perform the following actions:

  # null_resource.web[2] will be created
  + resource "null_resource" "web" {
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Costs:
  Totals:
    Monthly:  — → —   (— USD)
    Hourly:   — → —   (— USD)
    Resources: 8 → 5 (-3)
    Coverage: 0.0% (unchanged)
  Per state:
    acme-app: —/mo  (resources -3)
      + null_resource.web[2]                                +0.00
Security: impact not yet ready. Run `stategraph security impact summary --tx 6963ca5b-...` to view it later.
```

Dashes in `Costs:` mean nothing is priced on that side. Without `--out` the security line reads `Because --out wasn't passed, this transaction is transient and the security impact cannot be obtained after the fact.`

Output without changes: `No changes detected.` Exit 0, also with `--detailed-exitcode`. With `--out`, the file is written without a `tx_id`.

The plan file:

```json
{"tx_id":"6963ca5b-...","task_id":"5ac3d025-...","revision_hash":"72e422dd...","dirs":[{"dir":".","entry":{"workspace":"default"}}]}
```

## stategraph tf apply [PLAN]

With PLAN: commits the saved plan without prompting. Prints the resource actions and `Apply complete! Resources: N added, N changed, N destroyed.` Exit 0.

| Flag | Meaning |
|---|---|
| `--skip-revision-check` | Apply even when the HCL or tfvars changed after the plan |
| `--auto-approve` | Plan-less form only: skip the approval prompt |
| `--silent` (env `STATEGRAPH_SILENT`) | Suppress prompt output; use with `--auto-approve` |

Without PLAN: runs a fresh plan, shows the diff, prompts, then commits. Takes the `tf plan` flags except `--out`, `--diagnostics`, and `--detailed-exitcode`. With a non-tty stdin and no `--auto-approve` it fails after the plan with exit 1:

```text
Error: the approval prompt for `stategraph apply` requires an interactive terminal, but stdin is not a tty. Re-run with --auto-approve to apply without prompting.
```

Its transaction is aborted. Passing a plan-less flag together with PLAN is a usage error.

## stategraph tf show [--json] PLAN

Re-displays the saved plan. Text form ends with the `Plan:` line and has no cost or security block. `--json` emits a JSON object with keys `configuration`, `errored`, `format_version`, `planned_values`, `prior_state`, `relevant_attributes`, `resource_changes`, `terraform_version`, `timestamp`. Exit 0.

## stategraph tf mtx --out PLAN DIR...

Plans every workspace in each directory's group in one atomic transaction. `--tenant` and `--out` are required. Takes `--force`, `--skip-refresh`, `--skip-costs`, `--costs-wait`, `--skip-security`, `--security-wait`, `--batch-size`, `--parallel-batches`, `--detailed-exitcode`. Each DIR must hold `stategraph.json`. The plan file has the same shape as for `tf plan`, and `tf show` and `tf apply` accept it. No refactor session is driven.

## stategraph states resolve

Prints one id and nothing else. Exit 0.

| Flag | Meaning |
|---|---|
| `--name NAME` | Resolve by exact state name. Ignores `stategraph.json` and `--workspace` |
| `--workspace VAL` | Workspace to look up in `stategraph.json`. Default `default` |
| `--state UUID` | State id or group id |
| `--group` | Print the group id instead of the state id |

Unknown name: `Error: No state with workspace 'NAME' found for NAME`, exit 1.

## stategraph states delete

| Flag | Meaning |
|---|---|
| `--state UUID` | State to delete. Default: read from `stategraph.json` |
| `--workspace VAL` | Workspace, default `default` |
| `--auto-approve` | Skip the confirmation prompt. Required without a tty |

Success: `State <id> deleted`, exit 0. Without `--auto-approve` and no tty: `Aborted`, exit 1, nothing deleted. `-y` and `STATEGRAPH_NON_INTERACTIVE` do not replace `--auto-approve`. Unknown id: `Error: No state with workspace 'default' found for <id>`, exit 1. Requires admin on the tenant or the installation.

## stategraph tx

All output is JSON. `--format` is rejected: `stategraph: unknown option --format`, exit 124. `--tx` reads `STATEGRAPH_TX_ID`; `--tenant` reads `STATEGRAPH_TENANT_ID`.

| Command | Required flag | Notes |
|---|---|---|
| `tx list` | `--tenant` | `{"results": [...]}`, newest first |
| `tx logs list` | `--tx` | `{"results": [...]}`; empty transaction gives `{ "results": [] }` |
| `tx costs` | `--tx` | Prints the `Costs:` block of a plan transaction |
| `tx abort` | `--tx` | Prints the transaction with `"state": "aborted"`. Aborting an aborted transaction prints it again, exit 0 |
| `tx create` | `--tenant` | `--tag KEY=VALUE` (repeatable), `--tags JSON` (wins over `--tag`). Prints the transaction with `"state": "open"` |
| `tx create-with-session` | `--tenant` | Same flags. Prints only a session token (JWT) |

`tx list` entry fields: `id`, `state`, `created_at`, `created_by`, `created_by_name`, `params` (for example `{"skip_refresh": true}`), `tags`, `state_names` (states the transaction touches), `plan_summary` (`{"add": N, "change": N, "destroy": N}` once a plan ran), `completed_at` and `completed_by` after commit or abort.

`tx logs list` entry fields: `id`, `action`, `object_type`, `state_id`, `user_id`, `created_at`, `data`. Actions seen after a plan and apply: `hcl_set` (object `hcl`, the changed blocks with `file`, `fq_address`, `refs`, `edges`) and `state_set` (objects `state_metadata`, `resource`, `provider`, `instance`).

`tx costs` on a transaction without a preview: `Transaction not found, or aborted (abort drops the preview; committed and retryable-failed txs keep theirs).`, exit 1.

Transaction states: `open` (created), `previewing` (plan running), `previewed` (plan ready), `committing` (apply running), `committed`, `failed` (preview failed), `failed-committed` (apply failed after commit started), `aborted`.

## stategraph config files

Edits `stategraph.json` in the current directory without a server call. Creates the file when it is missing.

`attach --workspace WS --address ADDR [--dest DEST] [--expr EXPR] [-r] FILE_GLOB...`

- `--address` is a glob over block addresses, same grammar as `--force`.
- `--dest` supports `${path.module}` and `${path.root}` with an optional subdirectory. Default: bundle root.
- `--expr` names the exact HCL file call the attachment is for, for example `--expr 'fileset(path.module, "**/*.yaml")'`. The next plan fails when no matched block contains it.
- `-r`, `--recursive` treats arguments as paths; a directory contributes its contents with their relative paths.
- Address, `--dest`, and `--expr` together identify an attachment. Same triple again: globs are unioned.

Prints `Attached 1 glob(s) at module.db.* for workspace default`. The entry lands under `workspaces.<ws>.attached_files` as `{"address_glob": ..., "globs": [...]}`.

`remove --workspace WS --address ADDR [FILE_GLOB...]`

- Named globs are removed from every attachment at the address; an attachment left empty is dropped.
- No globs: every attachment at the address is removed. Prints `Removed address module.db.* from workspace default`.

## Error catalogue

| Message | Exit | Cause and next step |
|---|---|---|
| `A state must exist before you can run 'plan' or 'apply'.` | 1 | No `stategraph.json` in the directory. Change directory, or route to stategraph-import |
| `Error: Unable to find remote state` / `No stored state was found for the given workspace in the given backend.` | 1 | A `terraform_remote_state` data source with a local backend points at a missing file. Report the path. Stategraph resolves remote state itself only for `backend = "http"` with `https://stategraph/<state-id>` |
| `Local revision hash does not match the planned revision hash.` | 1 | HCL or tfvars changed after the plan. Plan again, or `--skip-revision-check` when the user asks |
| `Error: Transaction conflict` | 1 | Another transaction committed overlapping resources. The plan's transaction is aborted. Plan again |
| `the approval prompt for stategraph apply requires an interactive terminal` | 1 | Plan-less apply without a tty. Plan with `--out`, then apply the file |
| `Aborted` (from `states delete`) | 1 | Confirmation prompt with no tty. Add `--auto-approve` |
| `--skip-costs is mutually exclusive with --costs-wait.` | 124 | Drop one flag. Same for `--skip-security` with `--security-wait` |
| `stategraph: unknown option --format` (from `tx`) | 124 | `tx` output is always JSON |
| `Security is disabled on the server (STATEGRAPH_SECURITY)` | 0 | Scanning is off. Report once, do not retry |
| `Costs:` with only dashes and `Coverage: 0.0%` | 0 | No pricing configured. Report once, do not retry |
