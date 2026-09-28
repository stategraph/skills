---
name: stategraph-refactor
description: |
  Restructure a Terraform root module that is wired to Stategraph without losing state. Runs a refactor session: record address moves while carving resources into child modules or renaming them, prove the plan is moves only, then apply, which writes the moved blocks file and rewrites the state addresses.

  Use when the user says things like: "carve this root module into child modules", "move these resources into a module without destroying them", "refactor this repo and keep the state", "rename this resource or module without a destroy and create", "generate moved blocks for this restructure", "split the environments into shared modules plus env roots".

  Do not use for: adding, deleting, or changing resources (stategraph-change); moving resources between two Stategraph states; importing or wiring a repo (stategraph-import); read-only questions (stategraph-query); multi-directory `stategraph tf mtx` runs, which do not drive a session.

tags:
  - stategraph
  - refactor
  - terraform
  - modules
metadata:
  author: Stategraph
  version: "3.0"
---

# Stategraph refactor

## Setup

- Run every command from the root module directory: the one with `stategraph.json`, where `stategraph tf plan` runs. A session covers only that module.
- Env vars replace flags: `STATEGRAPH_API_BASE` (`--api-base`), `STATEGRAPH_API_KEY` (auth), `STATEGRAPH_TENANT_ID` (`--tenant`), `STATEGRAPH_WORKSPACE` (`--workspace`, default `default`). `--state` comes from `stategraph.json`.
- No `stategraph.json`: stop and route to stategraph-import.
- `.stategraph/` holds the session. Do not commit it. Commit the moved blocks file with the HCL.
- Exit codes: 0 ok; 1 session errors (below); 2 `tf plan --detailed-exitcode` ran and the plan is not empty; 123 error on stderr; 124 wrong flag; 125 internal bug. Report an error once and stop. Do not retry the same command.

## The loop

```bash
stategraph hcl addresses                                   # 1 declared addresses before
mkdir -p .stategraph && stategraph refactor start          # 2 open the session (silent, exit 0)
# 3 edit the HCL: one child module, or one rename, per step
stategraph hcl addresses                                   # 4 declared addresses after
stategraph refactor step                                   # 5 record the moves
stategraph tf plan --detailed-exitcode --skip-costs --skip-security --out plan.json   # 6 gate
stategraph tf show --json plan.json | jq -r '.resource_changes[] | select(.previous_address != null) | "\(.previous_address) -> \(.address) \(.change.actions)"'
stategraph tf show --json plan.json | jq '[.resource_changes[] | select(.change.actions != ["no-op"])] | length'   # must print 0
stategraph tf apply plan.json                              # 7 finalize, once, after the last step
```

Repeat 3 to 6 per module. Apply once at the end. The `mkdir -p` is required: without the directory `start` fails with `Start_write_err`.

## Reading `refactor step`

- `OLD -> NEW`, one per line: a rename that detection matched.
- `New entries:` then `added: <address>` lines: new blocks with nothing to map (the module call, its outputs, its `terraform` block). A move into a new module prints only this. Its mappings are recorded silently and appear in the plan gate as `previous_address`.
- `No changes detected.`: nothing changed since the last step.
- Exit 1 with stderr `Unable to determine mapping for removed entries:` and `removed:` lines: detection is ambiguous, for example one removed address and two identical new candidates. Give the mapping: `stategraph refactor step OLD=NEW [OLD=NEW ...]`, then continue. Addresses are full, for example `module.naming.random_pet.app_name`.
- Exit 1 with `Step_read_err ... .stategraph/refactor.json: No such file or directory`: no session is open. Run step 2.

## Reading the plan gate

While a session is open, stderr always carries `WARNING: Running in refactor mode.` and the plan runs a step itself.

| Exit | Output | Meaning | Action |
|---|---|---|---|
| 2 | `No changes. Your infrastructure matches the configuration.` | Moves only. The plan carries the moves, so `--detailed-exitcode` returns 2. | Run the two jq lines: every action is `no-op` and each moved resource has `previous_address`. Proceed. |
| 2 | `Plan: N to add, N to change, N to destroy.` | A real diff. | Not a refactor. Fix the HCL (an attribute changed, a reference was rewritten wrong, a resource was left behind) and plan again. Never apply. |
| 1 | stderr `Refactor step has unmatched entries.`, `added:` and `removed:` lines, `Run 'stategraph refactor step' to resolve.` | Ambiguous move. No plan file is written. | `stategraph refactor step OLD=NEW`, then plan again. |
| 0 | `No changes detected.` | The HCL evaluates the same as the stored configuration. | Nothing moved. Check the edit. |
| 123 | Error on stderr | HCL or server error. | Report the error lines and stop. |

## Finalize

`stategraph tf apply plan.json` prints `Apply complete! Resources: 0 added, 0 changed, 0 destroyed.` and, on stderr, `WARNING: Running in refactor mode. Finalizing refactor session.`. It rewrites the state addresses, writes `moved_stategraph.tf` in the module directory, and ends the session. Commit `moved_stategraph.tf` with the refactored HCL.

If the user does not want an apply, stop after the gate and report. The session stays open and `plan.json` can be applied later. `stategraph refactor abort` discards the session (silent, exit 0).

## Verify after apply

```bash
stategraph tf plan --detailed-exitcode --skip-costs --skip-security     # No changes detected., exit 0
STATE_ID=$(stategraph states resolve)
stategraph sql query "SELECT address, module FROM resources WHERE state_id = '$STATE_ID' ORDER BY address" --format=json
```

Root resources have `module` null. `sql query` has no `--state` flag: filter with `WHERE state_id = '...'`.

## Rules

1. One module, or one rename, per step. Do not rename and move the same resource in one step.
2. Change no attribute. A `Plan:` line with a non-zero count means the refactor is wrong.
3. Rewrite every reference to a moved resource to the module output (`random_pet.app_name.id` becomes `module.naming.app_name`), including `count`, `for_each`, `depends_on`, and root outputs.
4. The child module declares `terraform { required_providers { ... } }`. No empty `provider "x" {}` blocks.
5. Do not run `terraform state mv`, hand-write `moved` blocks, or run `stategraph tf mtx`.
6. Default layout: `./modules/<name>/`, called from the existing root. Read `references/carve-up.md` before proposing a split for a multi-environment repo or a custom layout.
7. Own the carve-up. Read the `.tf` files, propose the modules and their resources in one short message, start with the first, and continue without asking which resource to move next.

Read `references/outputs.md` for the verbatim output and error strings of every command above.
