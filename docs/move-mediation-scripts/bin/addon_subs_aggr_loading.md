**File:** `move-mediation-scripts\bin\addon_subs_aggr_loading.sh`  

---

## 1. High‑Level Summary
This script refreshes the *addon subscription aggregation* data in the MNAAS Impala data‑warehouse. It first clears the staging notification table, reloads it from a view, then truncates the target aggregation table (`mnaas.addon_subs_aggr`), inserts the latest aggregated rows from the view `mnaas.v_addon_subs_aggr`, and finally issues a table refresh so downstream queries see the new data. All actions are logged to a daily log file.

---

## 2. Important Variables & Commands  

| Item | Responsibility |
|------|-----------------|
| `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` | Sources common environment variables (e.g., `IMPALAD_HOST`, `MNAASLocalLogPath`). |
| `addon_subs_aggr_tblname` | Logical name of the target table (`addon_subs_aggr`). |
| `addon_subs_aggr_tblname_truncate` | Impala SQL to truncate the target table. |
| `addon_subs_aggr_tblname_insert` | Impala SQL to insert aggregated rows from `v_addon_subs_aggr`. |
| `addon_subs_aggr_tblname_refresh` | Impala SQL to refresh metadata for the target table. |
| `addon_subs_aggr_loadingLogPath` | Path of the daily log file (`$MNAASLocalLogPath/addon_subs_aggr_loading.log_YYYY-MM-DD`). |
| `impala-shell -i $IMPALAD_HOST -q "<SQL>"` | Executes each SQL statement against the Impala daemon. |
| `exec 2>>$addon_subs_aggr_loadingLogPath` | Redirects all subsequent STDERR to the log file. |

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | • `MNAAS_CommonProperties.properties` (defines `IMPALAD_HOST`, `MNAASLocalLogPath`, etc.)<br>• Impala views: `mnaas.v_addon_subs_notif`, `mnaas.v_addon_subs_aggr` |
| **Outputs** | • Daily log file with any error messages.<br>• Updated Impala tables: `mnaas.addon_subs_notif` (refreshed), `mnaas.addon_subs_aggr` (new rows). |
| **Side Effects** | • `TRUNCATE` removes *all* rows from `addon_subs_aggr` before insert.<br>• `INSERT` populates the table; any failure leaves the table empty until next run.<br>• `REFRESH` forces metadata reload for downstream queries. |
| **Assumptions** | • Impala daemon reachable at `$IMPALAD_HOST`.<br>• The source views are materialized and contain valid data.<br>• The user executing the script has `INSERT`, `TRUNCATE`, and `REFRESH` privileges on the target tables.<br>• Sufficient disk space for the log file and temporary Impala buffers. |

---

## 4. Integration with Other Scripts / Components  

| Connected Component | Relationship |
|---------------------|--------------|
| **MNAAS_Weekly_KYC_Feed_tbl_loading.sh**, **MNAAS_Traffic_tbl_…** | All scripts share the same `MNAAS_CommonProperties.properties` and run sequentially in nightly pipelines to populate related dimension/fact tables. |
| **Downstream reporting / Tableau** | The refreshed `addon_subs_aggr` table is consumed by reporting dashboards and KPI calculations. |
| **Orchestration layer (e.g., Oozie / Airflow)** | Typically invoked as a task in a nightly DAG after the source view `v_addon_subs_aggr` has been materialized by upstream ETL jobs. |
| **Alerting/Monitoring** | Errors written to the log are usually scraped by a log‑monitoring system (e.g., Splunk) and trigger alerts if the script exits with a non‑zero status (currently not implemented). |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Data loss if truncate succeeds but insert fails** | Add a transactional guard: run `INSERT` into a staging table, verify row count, then `SWAP` with the production table. |
| **Silent failures – script always returns 0** | Capture the exit status of each `impala-shell` call (`|| exit 1`) and propagate a non‑zero exit code to the orchestrator. |
| **Log file growth** | Rotate logs daily (already includes date) and enforce a retention policy (e.g., `logrotate`). |
| **Impala connectivity issues** | Include a retry loop with exponential back‑off around each `impala-shell` command. |
| **Schema drift** | Validate that the column list in `INSERT` matches the target table schema before execution; fail fast if mismatched. |

---

## 6. Example Run / Debugging Procedure  

1. **Prerequisite** – Ensure the environment variables from `MNAAS_CommonProperties.properties` are correct (especially `IMPALAD_HOST` and `MNAASLocalLogPath`).  
2. **Execute** – From the command line:  
   ```bash
   cd /app/hadoop_users/MNAAS/move-mediation-scripts/bin
   ./addon_subs_aggr_loading.sh
   ```
3. **Check Log** – Tail the log to verify success:  
   ```bash
   tail -f $MNAASLocalLogPath/addon_subs_aggr_loading.log_$(date +%F)
   ```
4. **Debug** – Add `set -x` at the top of the script to echo each command, or run a single statement manually:  
   ```bash
   impala-shell -i $IMPALAD_HOST -q "select count(*) from mnaas.v_addon_subs_aggr"
   ```
5. **Validate** – After completion, confirm row count matches source view:  
   ```bash
   impala-shell -i $IMPALAD_HOST -q "select count(*) from mnaas.addon_subs_aggr"
   ```

---

## 7. External Configuration / Environment Variables  

| Variable (from properties) | Usage |
|----------------------------|-------|
| `IMPALAD_HOST` | Hostname/port of the Impala daemon used by all `impala-shell` calls. |
| `MNAASLocalLogPath` | Base directory for the daily log file (`addon_subs_aggr_loading.log_YYYY-MM-DD`). |
| Any other variables defined in `MNAAS_CommonProperties.properties` (e.g., Hadoop classpath, Kerberos tickets) may be required for Impala authentication but are not referenced directly in this script. |

---

## 8. Suggested TODO / Improvements  

1. **Add robust error handling** – Capture the exit status of each `impala-shell` invocation and abort on failure, returning a non‑zero status to the orchestrator.  
2. **Implement a staging‑swap pattern** – Load into a temporary table (`addon_subs_aggr_stg`), verify row counts, then atomically rename/swap with the production table to eliminate the window where the target table is empty.  

---