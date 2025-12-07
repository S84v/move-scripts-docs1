**File:** `mediation-ddls\mnaas_billing_volte_roamers.hql`

---

## 1. High‑Level Summary
This HiveQL script creates (or replaces) the view `mnaas.billing_volte_roamers`. The view aggregates daily mediation data for VoLTE roaming customers, exposing a distinct list of TCL sector IDs (`tcl_secs_id`) together with the integer‑cast count of roamers (`countofroamers`). The view is filtered to the execution window (`transactiondate` between `start_with_time` and `end_with_time`) and limited to records where the partner MCC/MNC is `'ALL'` and the sector ID is not `'ALL'`. Down‑stream billing and reporting jobs consume this view to calculate roaming charges and usage metrics.

---

## 2. Key Objects & Responsibilities

| Object | Type | Responsibility |
|--------|------|-----------------|
| `billing_volte_roamers` | Hive **VIEW** | Provides a de‑duplicated mapping of sector IDs to the number of VoLTE roamers for the current processing window. |
| `mnaas.mobilium_mediation_raw_daily` | Hive **TABLE** | Source of raw daily mediation records (including `secsid`, `countofroamers`, `partner_mccmnc`, `transactiondate`). |
| `mnaas.month_billing` | Hive **TABLE** | Calendar / billing period reference used to bound the query by `transactiondate`. |
| `start_with_time` / `end_with_time` | Hive **VARIABLE** (passed via `-hiveconf`) | Define the inclusive start and end timestamps for the processing window. |

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | - Hive tables: `mnaas.mobilium_mediation_raw_daily`, `mnaas.month_billing`<br>- Hive variables: `start_with_time`, `end_with_time` (expected to be in a format comparable to `transactiondate` column, e.g., `yyyy-MM-dd`). |
| **Outputs** | - Hive **VIEW** `mnaas.billing_volte_roamers` (replaces any existing view of the same name). |
| **Side Effects** | - Overwrites the view definition; no data files are written directly.<br>- May trigger recompilation of dependent queries downstream. |
| **Assumptions** | - Both source tables exist and are up‑to‑date for the processing window.<br>- `transactiondate` column is comparable to the supplied timestamps.<br>- `partner_mccmnc` and `secsid` contain the literal `'ALL'` for rows that must be excluded.<br>- The Hive metastore is reachable and the user has `CREATE VIEW` privileges. |
| **External Services** | - Hadoop/Hive cluster (Metastore, execution engine).<br>- Scheduling/orchestration system (e.g., Oozie, Airflow) that injects the time variables. |

---

## 4. Integration Points & Connectivity

| Connected Component | How It Connects |
|---------------------|-----------------|
| **Pre‑processing scripts** (e.g., `mnaas_billing_traffic_*` DDLs) | Populate `mnaas.mobilium_mediation_raw_daily` with daily mediation records that this view consumes. |
| **Down‑stream billing jobs** (e.g., `mnaas_billing_volte_*` aggregation scripts) | Query `billing_volte_roamers` to compute roaming revenue, generate invoices, or feed into BI dashboards. |
| **Orchestration layer** | Passes `start_with_time` / `end_with_time` via `-hiveconf` or environment variables; may also trigger a `DROP VIEW IF EXISTS` step before execution (not present in this file). |
| **Reporting / Analytics** | Tools like Tableau, PowerBI, or custom Spark jobs reference the view for ad‑hoc analysis. |
| **Configuration repository** | Holds the variable definitions (e.g., a properties file `billing_window.properties` that the scheduler reads). |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Missing/invalid time variables** (`start_with_time`, `end_with_time`) | View may return empty set or cause query failure. | Validate variables at script start; fail fast with a clear error message. |
| **Source tables absent or schema changed** | View creation fails; downstream jobs break. | Add a pre‑flight check (`SHOW TABLES LIKE ...`) and schema validation (e.g., `DESCRIBE`). |
| **Duplicate view creation without DROP** | Hive may error if view already exists (depending on Hive version). | Prefix script with `DROP VIEW IF EXISTS mnaas.billing_volte_roamers;`. |
| **Performance degradation on large daily tables** | Full scan may exceed cluster resources, causing job timeouts. | Ensure appropriate partitioning on `transactiondate`; add `/*+ MAPJOIN */` hint if needed. |
| **Data type mismatch on `countofroamers`** | CAST to INT could truncate or error on non‑numeric values. | Add a safe conversion (`CASE WHEN regexp_like(countofroamers, '^[0-9]+$') THEN CAST(countofroamers AS INT) ELSE 0 END`). |
| **Incorrect filtering (`partner_mccmnc='ALL'`)** | May unintentionally include/exclude rows. | Confirm business rule with product owners; consider parameterising the filter. |

---

## 6. Execution & Debugging Guide

1. **Typical Run (via scheduler)**  
   ```bash
   hive -f mediation-ddls/mnaas_billing_volte_roamers.hql \
        -hiveconf start_with_time=2025-01-01 \
        -hiveconf end_with_time=2025-01-31
   ```

2. **Manual Debugging**  
   - Verify variables: `set start_with_time; set end_with_time;`  
   - Run the SELECT part alone to inspect results:  
     ```sql
     SELECT DISTINCT secsid tcl_secs_id,
                     CAST(countofroamers AS INT) countofroamers
     FROM   mnaas.mobilium_mediation_raw_daily rd
            CROSS JOIN mnaas.month_billing mb
     WHERE  rd.transactiondate BETWEEN '${hiveconf:start_with_time}'
                                 AND '${hiveconf:end_with_time}'
       AND  upper(rd.partner_mccmnc) = 'ALL'
       AND  upper(rd.secsid) != 'ALL';
     ```
   - Check row counts, data types, and any unexpected `NULL` values.

3. **Log Inspection**  
   - Hive logs are written to `$HIVE_LOG_DIR` (or captured by the orchestration tool). Look for `FAILED: SemanticException` or `Error while compiling view`.

4. **Rollback**  
   - If the view needs to be removed: `DROP VIEW IF EXISTS mnaas.billing_volte_roamers;`.

---

## 7. External Config / Environment Dependencies

| Item | Source | Usage |
|------|--------|-------|
| `start_with_time` / `end_with_time` | Supplied by the orchestration framework (e.g., Airflow DAG variables, Oozie workflow properties) or a properties file (`billing_window.properties`). | Limits the `transactiondate` range for the view. |
| Hive Metastore connection details | Typically defined in `hive-site.xml` on the execution node. | Required for view creation. |
| Hadoop/YARN resource configuration | `yarn-site.xml`, `core-site.xml`. | Determines job resources; may need tuning for large scans. |

---

## 8. Suggested Improvements (TODO)

1. **Add Idempotent View Management**  
   ```sql
   DROP VIEW IF EXISTS mnaas.billing_volte_roamers;
   CREATE VIEW mnaas.billing_volte_roamers AS ...
   ```
   Guarantees repeatable runs without manual cleanup.

2. **Parameterise Filters**  
   Replace hard‑coded `'ALL'` literals with Hive variables (`partner_filter`, `secsid_exclude`) so the same script can be reused for other partner scopes without code changes.

---