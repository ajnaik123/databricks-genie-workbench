# Hyphenated Catalog Support — Change Log

**Customer:** Mondelez
**Date:** 2026-05-11
**Repo:** `databricks-genie-workbench` (branch: `main`, uncommitted working-tree changes)
**Upstream:** https://github.com/databricks-solutions/databricks-genie-workbench

## Problem

Mondelez Unity Catalog catalogs contain hyphens (e.g. `mondelez-prod`). The Genie Space Optimizer (GSO) and workbench app build fully-qualified SQL identifiers as raw f-strings (`f"{catalog}.{schema}.{table}"`) without backticks. Spark/DBSQL parses the hyphen as the minus operator, producing a parse error.

**Failure surface observed:** Optimizer fails at **Preflight / create run** stage.

## Root Cause

Inconsistent quoting across the codebase. A few sites use existing helpers (`_q()`, `_quote_identifier_fqn()`); the majority use raw f-strings. The patch propagates safe backtick-quoting through the central helper plus per-site fixes.

## Files Modified (13 total)

### Central helper — auto-fixes many downstream callers

| File | Change |
|---|---|
| `packages/genie-space-optimizer/src/genie_space_optimizer/common/delta_helpers.py` | Added `_q(ident)` quoter. Updated `_fqn(catalog, schema, table)` to return `` `cat`.`sch`.`tbl` ``. Transitively fixes `read_table`, `insert_row`, `update_row`, `_migrate_add_columns` (ALTER/DESCRIBE), and ~30 callers across `optimization/state.py`. |

### Genie Space Optimizer (GSO) package — site-specific patches

| File | Change |
|---|---|
| `common/warehouse.py` | `wh_create_run` INSERT, `wh_load_run` SELECT, `wh_reconcile_active_runs` UPDATEs. Introduced local `_runs_fqn`. |
| `optimization/state.py` (lines 54, 67) | `CREATE SCHEMA IF NOT EXISTS` and `_ALL_DDL` template injection. |
| `optimization/labeling.py` | Quoted `flagged_questions` FQN at 4 sites. Added module-level `_q` import. |
| `optimization/optimizer.py` (line 2842) | `read_asi_from_uc` SELECT. |
| `optimization/evaluation.py` (line 2006) | `DROP FUNCTION IF EXISTS` now splits + quotes the FQN. |
| `optimization/benchmarks.py` | Wrapped 4 `table_name = ...` sites with existing `_quote_identifier_fqn`. Removed double-quoting at line 223 (`_safe_refresh(spark, table_name)` — `table_name` was already quoted). |
| `backend/router.py` (line 33) | Quoted health-check `fqn` (used in DESCRIBE SCHEMA, SHOW TABLES, SELECT 1, DESCRIBE VOLUME, SHOW FUNCTIONS). |
| `backend/routes/spaces.py` (lines 211, 216, 222, 265, 269, 278, 544, 757) | `information_schema` queries, benchmark questions SELECT, `SCHEMA_NOT_FOUND` error-message `CREATE SCHEMA` suggestion. Added `qfqn`. |
| `backend/job_launcher.py` (line 94) | Quoted `CREATE VOLUME` suggestion in error message. |
| `integration/apply.py` (line 69) | `genie_opt_iterations` SELECT. |

### Workbench app (separate from GSO wheel)

| File | Change |
|---|---|
| `backend/routers/auto_optimize.py` | Added `_q()` helper. Fixed `_delta_table()` and two direct f-string SELECTs at the `space_id = ...` queries. |
| `backend/services/scanner.py` (line 394) | GSO Delta fallback SELECT. |

## Deliberately NOT Modified

These look like FQNs but are not SQL identifiers in their respective contexts:

- **`uc_schema = f"{catalog}.{schema}"`** (~14 sites in `harness.py`) — stored as data (MLflow tags, `genie_opt_runs.uc_schema` column, error-message display). Downstream SQL consumers (`benchmarks.py`, `evaluation.py:6267`, `_drop_benchmark_table`) split + quote properly themselves.
- **`mlflow.genai.datasets.get_dataset(name=uc_table_name)`** / **`session.sync(to_dataset=...)`** — MLflow SDK rejects backticked names; the convention is unquoted there.
- **`ws.grants.update(full_name=fqn)`** in `backend/app.py:57` — REST API field, not SQL.
- **`/Volumes/{catalog}/{schema}/...`** filesystem paths — hyphens are valid in volume paths as-is.

## Validation

- All 13 patched files pass `python3 -m py_compile`.
- Git working-tree diff stat: **+76 / -36 lines**, 13 files.

## Deploy Procedure

> **Critical:** GSO runs as a separate Databricks Job (`GSO_JOB_ID`), built from a wheel in `packages/genie-space-optimizer/`. **Redeploying the app alone will NOT update the job** — the wheel must be rebuilt and re-uploaded.

```bash
cd "/Users/abhishek.naik/Mondelez Projects/databricks-genie-workbench"
make deploy WAREHOUSE_ID=<wh-id>
```

If `make deploy` is not the right entry point, check `databricks.yml` and `scripts/` for the bundle workflow.

## Open Items / Likely Next Suspects

If the optimizer still fails after redeploy, these sites haven't been audited and are the most likely remaining culprits:

- `optimization/preflight.py` (lines 291, 341, 425) — uses `_fq(...)` / `fq_table`. **Verify whether `_fq` quotes** — if not, patch it the same way as `_fqn` in `delta_helpers.py`.
- Any remaining `f"{cat}.{sch}"` displays that were treated as informational but turn out to flow into SQL.

Grep starting point:
```bash
rg 'f"\{[a-z_]*catalog[a-z_]*\}\.\{[a-z_]*schema[a-z_]*\}' \
   packages/genie-space-optimizer/src backend/
```

## Reference — Pre-existing Quoting

The codebase already had local quoters in two places, proving the pattern was known but not consistently applied:

- `optimization/benchmarks.py:320-323` — `_quote_identifier_fqn`
- `optimization/evaluation.py:6264-6267` — `_quote_identifier_fqn`
- `auto_optimize.py:789-792` and `backend/routes/settings.py:256-261` — GRANT statements already used backticks (untouched).
