**File:** `mediation-ddls\mnaas_msisdn_level_daily_usage_aggr_jlr_view.hql`  

---

## 1. High‑Level Summary
This HiveQL script creates (or replaces) the view **`mnaas.msisdn_level_daily_usage_aggr_jlr_view`**. The view aggregates raw daily usage records from the table `mnaas.msisdn_level_daily_usage_aggr` for the Jaguar‑Land‑Rover (JLR) business units. It groups usage by device identifiers (VIN, EUICCID, SIM), geography (country of sale, country of usage), device characteristics (type, make, model) and month, producing summed metrics for data (MB), voice (minutes) and SMS usage. The view is intended for downstream JLR‑specific reporting, billing, and analytics pipelines.

---

## 2. Core Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `msisdn_level_daily_usage_aggr_jlr_view` | Hive **VIEW** | Provides a pre‑aggregated, month‑level usage snapshot filtered to JLR business units, exposing columns: `vin`, `euiccid`, `sim`, `countryofsale`, `countryofusage`, `device_type`, `make`, `model`, `Data_Usage_in_MB`, `Voice_Usage_in_min`, `SMS_Usage`, `month`. |
| `mnaas.msisdn_level_daily_usage_aggr` | Hive **TABLE** (source) | Holds raw daily usage rows with columns referenced in the view (`vin`, `euiccid`, `sim`, `countryofsale`, `country`, `device_type`, `make`, `model`, `data_usage_in_mb`, `voice_usage_in_min`, `sms`, `calldate`, `tcl_secs_id`, `businessunit`). |
| `createtab_stmt` (metadata wrapper) | Script‑level variable | Holds the full `CREATE VIEW` DDL; used by the build/orchestration framework to execute the statement. |

*No procedural code, functions, or classes are defined in this file; it is a declarative DDL script.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - Hive table `mnaas.msisdn_level_daily_usage_aggr` (must exist and contain the listed columns).<br>- Implicit filter values: `tcl_secs_id = 24048` and `businessunit` in `('JLR Production', 'JLR Production PSIMs', 'JLR FOC Engg Support')`. |
| **Outputs** | - Hive view `mnaas.msisdn_level_daily_usage_aggr_jlr_view` (created or replaced). |
| **Side‑Effects** | - Registers/overwrites the view definition in the Hive metastore.<br>- May trigger downstream materialized view refreshes or dependent jobs that reference the view. |
| **Assumptions** | - Hive/Tez/MapReduce execution engine is available and configured.<br>- The `mnaas` database exists and the user running the script has `CREATE`/`ALTER` privileges.<br>- `calldate` is stored as a string where the first 7 characters represent `YYYY‑MM`.<br>- No partitioning is defined on the source table; the view relies on the underlying table’s performance characteristics. |
| **External Services** | - Hive Metastore (metadata).<br>- Underlying HDFS or cloud storage where the source table resides. |

---

## 4. Integration Points  

| Connected Component | Relationship |
|---------------------|--------------|
| **Downstream reporting jobs** (e.g., JLR billing, usage dashboards) | Query the view to obtain month‑level aggregates without re‑scanning the raw daily table. |
| **`mnaas_month_billing.hql` / `mnaas_month_ppu_billing.hql`** | Likely consume this view (or its columns) when calculating JLR‑specific charges. |
| **Orchestration framework (e.g., Oozie, Airflow, custom scheduler)** | Executes this DDL as part of a daily/weekly “metadata refresh” DAG. The script may be referenced via a `createtab_stmt` variable in a master manifest. |
| **Data quality / validation scripts** (not shown) | May run `SELECT COUNT(*) FROM mnaas.msisdn_level_daily_usage_aggr_jlr_view` to verify row counts after creation. |
| **Configuration files** | Typically a `hive-site.xml` or environment variable (`HIVE_DB=mnaas`) that determines the target database. No explicit config is referenced inside the file. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Source schema drift** – columns renamed/removed in `msisdn_level_daily_usage_aggr`. | View creation fails; downstream jobs break. | Add a schema‑validation step (e.g., `DESCRIBE` + assert required columns) before executing the DDL. |
| **Large GROUP BY** on an unpartitioned table may cause long execution times or OOM failures. | Job stalls, resource exhaustion. | Ensure the source table is partitioned by `calldate` (or `month`) and add `WHERE calldate >= …` to limit data. Consider using `MAPJOIN` hints if the dimension set is small. |
| **Incorrect filter values** (e.g., `tcl_secs_id` change). | Data for JLR may be omitted or polluted. | Externalize filter constants to a config file or Hive variable (`${JLR_TCL_SECS_ID}`) and version‑control them. |
| **View name collision** – another process recreates the view with different definition. | Inconsistent analytics. | Adopt a naming convention with version suffix (e.g., `_v20251201`) and enforce via CI checks. |
| **Missing privileges** for the execution user. | DDL fails silently in batch runs. | Include a pre‑flight check that `SHOW GRANT USER <user>` contains `CREATE`/`ALTER` on `mnaas`. |

---

## 6. Running & Debugging the Script  

1. **Execution**  
   ```bash
   # Using Hive CLI
   hive -f mediation-ddls/mnaas_msisdn_level_daily_usage_aggr_jlr_view.hql

   # Or via Beeline (preferred)
   beeline -u "jdbc:hive2://<hive-host>:10000/mnaas" -f mediation-ddls/mnaas_msisdn_level_daily_usage_aggr_jlr_view.hql
   ```

2. **Verification**  
   ```sql
   -- Verify view exists
   SHOW CREATE VIEW mnaas.msisdn_level_daily_usage_aggr_jlr_view;

   -- Sample data check
   SELECT * FROM mnaas.msisdn_level_daily_usage_aggr_jlr_view LIMIT 10;
   ```

3. **Debugging Tips**  
   - Run the SELECT part alone against the source table to ensure it returns rows.  
   - Use `EXPLAIN EXTENDED` on the SELECT to inspect the execution plan and spot missing partitions.  
   - Check Hive logs (`/var/log/hive/`) for “SemanticException” messages if creation fails.  
   - If the job times out, increase `hive.exec.reducers.max` or enable `hive.optimize.skewjoin` as needed.

---

## 7. External Configuration / Environment Variables  

| Variable / File | Purpose | Usage |
|-----------------|---------|-------|
| `HIVE_CONF_DIR` / `hive-site.xml` | Hive connection & execution settings (Metastore URI, execution engine). | Loaded automatically by Hive/Beeline. |
| `HIVE_DB` (optional) | Target database name; defaults to `mnaas`. | Could be referenced as `${hiveconf:HIVE_DB}` in a wrapper script. |
| `JLR_TCL_SECS_ID` (suggested) | Allows the `tcl_secs_id` filter to be changed without editing the script. | Replace literal `24048` with `${hiveconf:JLR_TCL_SECS_ID}`. |
| `JLR_BUSINESS_UNITS` (suggested) | List of business units to include. | Replace the hard‑coded IN clause with `${hiveconf:JLR_BUSINESS_UNITS}`. |

No such variables are currently used; the script contains hard‑coded literals.

---

## 8. Suggested Improvements (TODO)

1. **Parameterize filter constants** – replace the hard‑coded `tcl_secs_id` and business‑unit list with Hive variables (e.g., `${hiveconf:JLR_TCL_SECS_ID}`) and document them in a central config file. This eases future changes and supports different environments (dev/test/prod).

2. **Add a month partition to the source table and include it in the view definition** – modify the source table to be partitioned by `substr(calldate,1,7)` and add `PARTITIONED BY (month STRING)` to the view. This will dramatically improve query performance for downstream jobs that filter on `month`.  

---