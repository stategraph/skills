---
name: stategraph-import
description: |
  Onboard a Terraform root module into Stategraph with the CLI: import a Terraform state file plus its HCL into a new Stategraph state, replace or re-import a state, import HCL only, upload a state file only, preflight a module before importing, and anonymize a state file for sharing.

  Use this skill when the user says things like "import this state into Stategraph", "onboard this repo", "get my terraform.tfstate into Stategraph", "wire this directory to Stategraph", "re-import after the failed import", "record the HCL for this state", "will this module import cleanly", "check the HCL before importing", or "anonymize this state file".

  Do not use it for querying or summarizing states that already exist (stategraph-query), plan or apply (stategraph-change), moving resources between modules (stategraph-refactor), or cost questions (stategraph-cost).

tags:
  - stategraph
  - import
  - onboarding
  - terraform
metadata:
  author: Stategraph
  version: "3.0"
---

# Stategraph import

Environment replaces flags: `STATEGRAPH_API_BASE` for `--api-base`, `STATEGRAPH_TENANT_ID` for `--tenant`, `STATEGRAPH_WORKSPACE` for `--workspace` (default `default`), `STATEGRAPH_NON_INTERACTIVE=1` for `-y`. Without `-y` or that variable, a run with no terminal stops at `Continue? (y/n)` and prints `User chose not to continue`.

Exit codes: 0 ok, 123 error on stderr, 124 wrong flag or missing argument, 125 internal bug. The import errors listed under Failures exit 1 with the message on stderr.

Import is a write. Before the import command, state the source file, the tenant, the state name, and the workspace in one line. Then run it.

## Import a Terraform project (state file + HCL)

`stategraph import tf` creates the state and imports the state file and the HCL of the current directory in one step. Run every command below from the root module directory, with every child module directory present.

1. Find the source. Use the file the user names. Otherwise use `terraform.tfstate` in the module directory. For a remote backend run `terraform state pull > terraform.tfstate` first (needs the backend credentials).
2. Preflight when the HCL is unfamiliar or uses variables, modules, or file functions. Both commands run locally and make no API call.
   ```bash
   stategraph hcl addresses
   stategraph diagnostics run --mode rollup --var-file prod.tfvars 2>&1 | grep SG_TFEVAL_ROLLUP
   ```
   Read `vars_missing`, `file_failures`, `file_warnings`, and `node_dropped`. A value above 0 means a missing `--var` or `--var-file`, or a file path Stategraph could not pin. Fix that before importing. `references/diagnostics.md` explains the fields.
3. Import. Pass the same `--var` and `--var-file` flags as `terraform plan`, so the HCL evaluates as in Terraform. Omit them when the module declares no variables without defaults.
   ```bash
   stategraph import tf terraform.tfstate --name NAME --var-file prod.tfvars --var region=us-east-1
   ```
   Success prints `Terraform state and HCL imported successfully for state <ID> (transaction <TX>)`. Take the state ID from that line. The command writes `stategraph.json` (`{ "group_id": "..." }`) to the current directory. Later commands find the state through it, so tell the user to commit it. The warning about existing backend configuration is informational.
4. Verify.
   ```bash
   stategraph states summary --state ID --format=json
   stategraph states resources summary --state ID --format=json
   stategraph sql query "SELECT count(*) AS resources FROM resources WHERE state_id = 'ID'" --format=json
   ```
   `summary` returns `resources`, `instances`, `edges`, `modules`, `providers`. `resources summary` returns instance counts per resource type. `sql query` has no `--state` flag; filter with `WHERE state_id`.
5. Report the command, the state name and ID, the counts, and the next step: `stategraph tf plan` in this directory, or a query.

To get the ID again: `stategraph states resolve --name NAME`, or `stategraph states resolve --workspace default` in the directory that holds `stategraph.json`.

## Variants

| Task | Command | Notes |
|---|---|---|
| Replace an existing state (re-import) | `stategraph import tf terraform.tfstate --overwrite` | Run in the directory whose `stategraph.json` names the state. Replaces the contents of the state: anything the import does not contain is deleted. Transaction history, cost data, and security data stay. |
| Re-import when `stategraph.json` is absent | `stategraph import tf terraform.tfstate --overwrite --name NAME` | Creates NAME when no `stategraph.json` exists. |
| HCL only, new state (infrastructure not created yet) | `stategraph import tf --hcl --name NAME` | No FILE argument. Resources and instances stay empty. Prints `HCL configuration imported successfully for state <ID>`. |
| HCL only, re-record the bound state | `stategraph import tf --hcl --overwrite` | Run in the directory with `stategraph.json`. Replaces configuration, variable values, attached files, and revision hashes. Recorded resources and instances stay unchanged. |
| Empty state, nothing to import yet | `stategraph states create --name NAME` | Prints the new state as JSON (`id`, `group_id`, `name`, `workspace`). Writes `stategraph.json` unless `--no-write-config`. |
| State file only, no HCL | `stategraph states import terraform.tfstate --name NAME` | Prints the new state as JSON (`id`, `group_id`, `name`, `workspace`). Writes no `stategraph.json`. |
| Share a state file safely | `stategraph states anonymize terraform.tfstate --seed 42 > anonymized.tfstate` | Local only. Replaces values with `anon_*` tokens. |
| Import without writing `stategraph.json` | add `--no-write-config` | |
| Another workspace or a group | add `--workspace WS` or `--group GROUP_ID` | |
| Remove a state this task created | `stategraph states delete --state ID --workspace default --auto-approve` | Prints `State ID deleted`. |

## Files read by file(), templatefile(), fileset()

Stategraph discovers and bundles files whose path evaluates to a literal. When a path depends on a value known only at apply time, the import warns `some HCL file references could not be pinned` and bundles every file that matches the known part. Use `--attach-files` and `--attach-dest` only for those unresolved calls:

```bash
stategraph import tf terraform.tfstate --name NAME \
  --attach-files 'aws_sqs_queue_policy.*=templates/*.tpl' \
  --attach-dest 'aws_sqs_queue_policy.*=${path.module}/templates'
```

Attaching to a call Stategraph already resolved fails with `ATTACHED_FILES_SHADOW_RESOLVED_CALL`, and the import is rolled back. Narrow the address glob to the unresolved block; `stategraph hcl addresses` lists the addresses. Check the result with `stategraph sql query "SELECT filepath, module_address FROM files WHERE state_id = 'ID'" --format=json`.

## Failures

Report each once and stop. Do not retry the same command.

| Message | Cause | Action |
|---|---|---|
| `Error: a state with that name already exists` | `--name` is taken. Nothing was created. | Choose another name, or run with `--overwrite` in the directory that holds that state's `stategraph.json`. |
| `Error loading HCL: \`Load_root_module_dir_read_err (("modules/net", "modules/net: No such file or directory"))` | A directory named in a `module` block is missing, or the command ran outside the root module. | Run from the root module with every module directory present. A failed `import tf` deletes the state it started. |
| `User chose not to continue` | The prompt got no `y`. | Add `-y` or set `STATEGRAPH_NON_INTERACTIVE=1`. |
| `Error: No state with workspace 'NAME' found for NAME` from `states resolve --name` | No state has that name. | Check `stategraph states list --format=json`. |
| `Aborted` from `states delete` | No terminal and no `--auto-approve`. | Add `--auto-approve`. |

## Do not

- Do not run `stategraph hcl import`. It is deprecated and leaves the state's schema version stale. Run `stategraph import tf --hcl --overwrite` in the directory with `stategraph.json` instead. `import tf` has no `--state` flag.
- Do not pass `--state` to `sql query`. Filter with `WHERE state_id = 'ID'`.
- Do not import the same file under a second name to retry. Use `--overwrite`.
- Do not delete a state this task did not create.
- After the import, route module carve-ups to stategraph-refactor, plan and apply to stategraph-change, and inventory questions to stategraph-query.

Read `references/flags.md` for every option of `import tf`, `diagnostics run`, `states import`, `states anonymize`, `states resolve`, and `states delete`.
