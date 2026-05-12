# Hyphenated Catalog Support — Change Log

**Customer:** Mondelez
**Branch base:** `origin/main @ 5ff2a41` (post-rebase from `edcfbb6`; v1 of this patch preserved at tag `archive/hyphen-fix-v1-edcfbb6-base`)
**Upstream:** https://github.com/databricks-solutions/databricks-genie-workbench
**Fork:** https://github.com/ajnaik123/databricks-genie-workbench

## Problem

Mondelez's Unity Catalog catalogs contain hyphens (e.g. `mondelez-prod`). The Genie Space Optimizer (GSO) and workbench app build fully-qualified SQL identifiers as raw f-strings (`f"{catalog}.{schema}.{table}"`) without backticks. Spark/DBSQL parses the hyphen as the minus operator, producing a parse error at every FQN site.

**Failure surface previously observed:** Optimizer fails at **Preflight / create run** stage. The fix below covers the optimizer, the integration API, the workbench app, GSO state management, and human-facing SQL suggestions in error messages.

## Root Cause

Inconsistent quoting across the codebase. A handful of sites use existing helpers (`_quote_identifier_fqn()` in `benchmarks.py`, `_q()` local to `evaluation.py`); most use raw f-strings. This patch propagates safe backtick-quoting through one central helper plus per-site fixes everywhere else.

## Files Modified (18 total)

### 1. Central helper — auto-fixes many downstream callers

| File | Change |
|---|---|
| `packages/.../common/delta_helpers.py` | Added module-level `_q(ident)` quoter. Updated `_fqn(catalog, schema, table)` to return `` `cat`.`sch`.`tbl` ``. This automatically fixes `read_table` (line ~200), `insert_row` (line ~249), `update_row` (line ~285), and **every caller of `_fqn()` in `state.py`** (~30 sites) and `scan_snapshots.py`, `labeling.py`, `runs.py`. |

### 2. GSO package — `genie-space-optimizer/src/genie_space_optimizer/`

| File | Sites | Change |
|---|---|---|
| `common/warehouse.py` | 95, 119, 184, 195, 221 | Imported `_q`. Replaced raw `{catalog}.{schema_name}` with `{_q(catalog)}.{_q(schema_name)}` in the INSERT, SELECT, and three UPDATE statements. |
| `optimization/state.py` | 32, 65, 77 | Added `_q` to delta_helpers import. Quoted `CREATE SCHEMA IF NOT EXISTS`. Quoted `_ALL_DDL` template substitution so all generated `CREATE TABLE IF NOT EXISTS` statements receive backticked catalog/schema. |
| `optimization/labeling.py` | 24, 559, 604, 676, 716 | Imported `_fqn` from delta_helpers. Replaced 4 raw `fqn = f"{catalog}.{schema}.genie_opt_flagged_questions"` with `_fqn(catalog, schema, "genie_opt_flagged_questions")`. |
| `optimization/optimizer.py` | 5357 | Inlined backtick quoting on the `read_asi_from_uc` SELECT (single site — avoiding an import in this 12k-line file). |
| `optimization/evaluation.py` | 5455 | `DROP FUNCTION IF EXISTS` now splits the incoming `fqn` and quotes each part. |
| `optimization/benchmarks.py` | 212, 1147, 1271, 1303 | Wrapped each `table_name = f"{uc_schema}.genie_benchmarks_{domain}"` assignment with the existing `_quote_identifier_fqn()` helper. Removed now-redundant `_quote_identifier_fqn(...)` wrap at the `_safe_refresh` call site. |
| `optimization/scan_snapshots.py` | 23, 43, 97 | Imported `_fqn`. Replaced both `fqn = f"{catalog}.{schema}.{TABLE_SCAN_SNAPSHOTS}"` with `_fqn(...)`. |
| `backend/router.py` | 33, 84, 88, 103, 130, 152–154, 161, 180–181, 190, 208, 224, 232, 249 | Added a `qfqn = f"`{cat}`.`{sch}`"` companion to the existing `fqn`. Used `qfqn` in every SQL statement and every `CREATE SCHEMA` / `CREATE VOLUME` / `GRANT` suggestion shown to the user; kept `fqn` unquoted only for display strings. |
| `backend/routes/spaces.py` | 540, 749–755, 797, 1002, 1037 | Quoted the `bench_table` benchmark SELECT, the `CREATE SCHEMA IF NOT EXISTS` suggestion in `_actionable_setup_error`, and three `SELECT/UPDATE` statements in the SQL-warehouse fallback path. |
| `backend/routes/runs.py` | 2830 | Switched the `labeling_session_url` lookup to `_fqn(catalog, schema, "genie_opt_runs")`. |
| `backend/job_launcher.py` | 93–94 | Backticked the `CREATE VOLUME IF NOT EXISTS` suggestion in the volume-creation error message. |
| `integration/apply.py` | 69, 97 | Quoted iterations SELECT and runs UPDATE. |
| `integration/trigger.py` | 98, 110, 236, 270 | Quoted two runs SELECTs and two runs UPDATEs. |
| `integration/discard.py` | 76 | Quoted runs UPDATE on user-discarded path. |
| `jobs/_handoff.py` | 149, 270, 383, 461 | Quoted the bootstrap `genie_opt_runs` SELECT plus three `genie_opt_iterations` fallback SELECTs. |

### 3. Workbench app (separate from GSO wheel)

| File | Sites | Change |
|---|---|---|
| `backend/routers/auto_optimize.py` | 127–130, 1500, 1514 | Updated `_delta_table()` to return a backtick-quoted FQN and quoted both `genie_opt_runs` SELECTs in the SQL-warehouse fallback. |
| `backend/services/scanner.py` | 190 | Quoted the GSO Delta fallback SELECT (`SELECT run_id, space_id, status, …`). |

### 4. Deploy-time scripts (run locally before workspace deploy)

These are invoked by `make deploy` / `scripts/install.sh` and execute via the SQL Statement Execution API against the target warehouse. Missed in the original v2 audit because the audit scoped to runtime package code; surfaced when first-time deploy against `dev-amer-geniepoc-catalog` failed on `CREATE SCHEMA`.

| File | Sites | Change |
|---|---|---|
| `scripts/deploy_lib/uc.py` | 69, 74, 80, 81, 88, 104 | Added local `_q()` helper. Quoted `ensure_schema` CREATE SCHEMA, `ensure_volume` CREATE VOLUME, `ensure_tables` `_ALL_DDL` substitution, and `enable_change_data_feed` ALTER TABLE. |
| `scripts/grant_permissions.py` | 28, 97, 121, 174, 190 | Added local `_q()` helper. Quoted `_ensure_schema` CREATE SCHEMA, `_ensure_volume` CREATE VOLUME, `_ensure_tables` `_ALL_DDL` substitution, and CDF ALTER TABLE statement. REST-API call sites (`_update_grants(full_name=...)`, `_get_grants(full_name=...)`) intentionally left unquoted — those are SDK args, not SQL. |

## Deliberately NOT Modified

These look like FQNs but are not SQL identifiers in their respective contexts:

- **`uc_schema = f"{catalog}.{schema}"`** in `harness.py` (~14 sites), `preflight.py`, `auto_optimize.py:1029`, every `jobs/run_*.py` — stored as **data**. Written to MLflow tags, the `genie_opt_runs.uc_schema` column, error-message display, and passed to helpers (`check_prompt_registry`, `_check_sp_data_access`, etc.) that split + quote on their own.
- **`mlflow.genai.datasets.get_dataset(name=...)`** / **`session.sync(to_dataset=...)`** — MLflow SDK rejects backticked names; the convention is unquoted there.
- **`ws.grants.update(full_name=...)`** in `backend/app.py:57` — REST API field, not SQL.
- **`backend/services/uc_client.py:221,228,238`** — logger statements and `client.tables.get(...)` REST API; not SQL.
- **`backend/services/create_agent_tools.py:1092`** — `"full_name"` REST API response.
- **`/Volumes/{catalog}/{schema}/...`** filesystem paths — hyphens are valid in volume paths as-is.
- **`preflight.py:108,337,384,428,468`** — already uses a local `fq_quoted` variable for SQL; raw `fqn` is only used for display.
- **`evaluation.py`** SHOW USER FUNCTIONS, quoted_table_name, SHOW TABLES (lines 4143, 12500, 12506) — already wrapped with the file's local `_q()` helper.
- **`backend/services/plan_builder.py:672`** — join SQL using user-supplied `left_table`/`right_table` from Genie space join specs. Quoting requires parsing the upstream identifier shape (1-part vs 3-part) and is a separate change.

## Validation

- All patched files pass `python3 -m py_compile` (18 runtime + 2 deploy-time).
- Confirmed end-to-end against catalog `dev-amer-geniepoc-catalog`, schema `genie_workbench_abhi` — install completes, optimizer creates run successfully.

## Deploy Procedure

> **Critical:** GSO runs as a separate Databricks Job (`GSO_JOB_ID`), built from a wheel under `packages/genie-space-optimizer/`. **Redeploying the app alone will NOT update the job** — the wheel must be rebuilt and re-uploaded.

```bash
cd "/Users/abhishek.naik/Mondelez Projects/databricks-genie-workbench"
make deploy WAREHOUSE_ID=<wh-id>
```

If `make deploy` is not the right entry point, check `databricks.yml` and `scripts/` for the bundle workflow.

## Reference

- **v1 of this patch** (against base `edcfbb6`, applied 2026-05-11) is preserved at tag `archive/hyphen-fix-v1-edcfbb6-base` on `origin` (the fork). Useful for diffing if a resolution choice needs to be revisited.
- **Existing local quoters** in the codebase (unchanged): `optimization/benchmarks.py:60` (`_quote_identifier`), `:348` (`_quote_identifier_fqn`); `optimization/evaluation.py:4169` (`_quote_identifier`), `:12497` (`_q`); `optimization/harness.py:3462` (`_quote_identifier`); `optimization/scorers/syntax_validity.py:36` (`_quote_identifier`).
- **Open follow-up:** `optimization/preflight.py:816` defines `_fq(tbl)` — verify whether it quotes (it lives in a function body, separate from the now-quoted `_fqn` in `delta_helpers.py`). If preflight errors recur post-deploy, that helper is the next suspect.
