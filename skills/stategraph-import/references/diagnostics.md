# Reading a diagnostics run

`stategraph diagnostics run` evaluates the root module locally and prints `SG_TFEVAL_*` lines as `key=value` records. The last line is the rollup. Nothing is uploaded, and no value contents are recorded.

## The rollup

```bash
stategraph diagnostics run --mode rollup --var-file prod.tfvars 2>&1 | grep SG_TFEVAL_ROLLUP
```

```text
SG_TFEVAL_ROLLUP nodes=13 node_unresolved=10 node_dropped=0 module_inputs=0 input_findings=0 attributes=278 attr_unresolved=8 unresolved_capability_gap=0 attr_dropped=0 dropped_capability_gap=0 dropped_type_review=0 dropped_config_error=0 file_failures=0 file_warnings=0 validations=0 validation_failures=0 modules_bridged=1 vars_missing=0 blocks_iterating=0 blocks_pruned=0 blocks_meta_unresolved=0 coercions=0 coercion_defaulted=0 coercion_failed=0 steps=638 step_errors=65
```

Fields that decide whether to import:

| Field | Meaning | Action when above 0 |
|---|---|---|
| `vars_missing` | Declared variables with no value. | Add `--var KEY=VALUE` or `--var-file FILE` to the diagnostics run and to `import tf`. |
| `file_failures` | A `file()`, `templatefile()`, or `fileset()` call whose file could not be read. | Check the path from the `SG_TFEVAL_FILE` line. |
| `file_warnings` | A file path known only in part, e.g. it contains a variable. Stategraph bundles every file matching the known part. | Import works. Use `--attach-files` only when the bundled set is wrong. |
| `node_dropped`, `attr_dropped` | A local, output, module input, or attribute that could not be evaluated at all. | Read the `SG_TFEVAL_NODE` or `SG_TFEVAL_ATTR` line for the leaf cause. |
| `validation_failures` | A `variable` validation block failed. | Fix the variable value. |

`node_unresolved`, `attr_unresolved`, and `step_errors` count values known only at apply time, such as data source outputs and remote state. They do not block an import.

## Line kinds

| Line | Reports |
|---|---|
| `SG_TFEVAL_ROOT` | Root module directory and workspace. |
| `SG_TFEVAL_NODE` | A variable, local, output, or module input with its outcome and leaf cause. |
| `SG_TFEVAL_ATTR` | A resource attribute: resolved, unresolved, or dropped. |
| `SG_TFEVAL_REF` | A reference and how its address resolved. `resolution=undefined` marks a reference with no target. |
| `SG_TFEVAL_FILE` | A file function call, with `level=warn` or a failure reason. |
| `SG_TFEVAL_MODULE` | A module call and its declared-input reconciliation. |
| `SG_TFEVAL_VALIDATION` | A variable validation and its outcome. |
| `SG_TFEVAL_META`, `SG_TFEVAL_COERCE` | `count`/`for_each` shapes and per-field coercions. `--mode detail` and above. |
| `SG_TFEVAL_STEP` | One line per evaluated expression. `--mode verbose` only. |
| `SG_TFEVAL_PERF*` | Timing breakdown. `--mode perf`. |

## Useful greps

```bash
stategraph diagnostics run --out diagnostics.txt --var-file prod.tfvars
grep SG_TFEVAL_ROLLUP diagnostics.txt
grep SG_TFEVAL_FILE diagnostics.txt
grep -E '^SG_TFEVAL_(NODE|ATTR) .*outcome=dropped' diagnostics.txt
grep 'resolution=undefined' diagnostics.txt
```

On `SG_TFEVAL_NODE` and `SG_TFEVAL_ATTR` lines, `outcome=concrete` is a value known now, `outcome=unresolved` is a value known after apply (`detail` names the cause, e.g. a missing variable or a remote state output), and `outcome=dropped` is a value that could not be evaluated. `SG_TFEVAL_STEP` lines also carry `outcome=dropped` for apply-time unknowns; they are counted in `step_errors` and do not block an import.

To trace another directory: `stategraph diagnostics run ./infra/networking --out networking-diagnostics.txt`.

The default mode is `verbose` and writes one line per expression, so use `--out FILE` for a large module. The trace is safe to share: data-map keys are hidden unless `SG_DIAG_SHOW_KEYS=1` is set.
