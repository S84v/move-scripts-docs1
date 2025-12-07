**File:** `mediation-ddls\mnaas_v_ictraffic_details_ppu_billing_month.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive/Impala view **`mnaas.v_ictraffic_details_ppu_billing_month`**. The view consolidates raw inter‑connect CDRs with the monthly billing calendar, filters and normalises the data (call‑type decoding, sponsor extraction, usage calculation), de‑duplicates records using a `row_number()` window, and exposes a clean, month‑scoped dataset that downstream billing and reporting jobs consume for per‑unit‑price (PPU) billing calculations.

---

## 2. Core Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `v_ictraffic_details_ppu_billing_month` | View | Provides a month‑filtered, de‑duplicated, and transformed set of inter‑connect traffic details for PPU billing. |
| `filtered_cdr` | CTE (Common Table Expression) | Performs the heavy lifting: joins `interconnect_traffic_details_raw` with `month_billing`, applies business filters, derives derived columns (`sponsor`, `calltype`, `usage`, `actualusage`, etc.), and assigns a `row_num` for deduplication. |
| `decode()` calls | Function (Hive/Impala) | Maps raw `calltype` values to internal codes (`ICSMS`, `ICVoice`) and selects the appropriate usage field (`icbillableduration` for voice, constant `1` for SMS). |
| `row_number() over (partition … order by filename desc)` | Window function | Guarantees a single record per `(cdrid, chargenumber, timebandindex)` tuple, keeping the most recent file version. |
| `substring(imsi,1,5)` | Expression | Extracts the first five digits of the IMSI to identify the sponsoring carrier. |
| `substring(CAST(calldatetime AS STRING),1,10)` | Expression | Normalises the call timestamp to a `YYYY‑MM‑DD` date string (`calldate`). |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Primary Input Tables** | `mnaas.interconnect_traffic_details_raw` (raw CDRs) <br> `mnaas.month_billing` (calendar with `start_date`, `end_date` columns) |
| **Key Input Columns** | `partition_date`, `cdrid`, `chargenumber`, `timebandindex`, `calltype`, `traffictype`, `imsi`, `customernumber`, `filename`, `calldatetime`, etc. |
| **External Parameters** | `start_date` and `end_date` are read from `month_billing` rows; they define the month window for the view. |
| **Output** | A **view** (`mnaas.v_ictraffic_details_ppu_billing_month`) exposing the columns listed in the final `SELECT`. No physical data is written; the view is a logical definition. |
| **Side Effects** | DDL operation – creates or replaces the view in the `mnaas` schema. |
| **Assumptions** | • Hive/Impala engine is available and configured for the `mnaas` database.<br>• Source tables exist with the expected schema (especially `partition_date`, `customernumber`, `calltype`, `traffictype`).<br>• `month_billing` contains exactly one row for the target month (or the script is run per‑month).<br>• No column name collisions between the two source tables (the script uses a cross join style without explicit join condition – relies on the `WHERE` clause for filtering). |

---

## 4. Integration Points  

| Connected Component | How It Connects |
|---------------------|-----------------|
| **Down‑stream billing jobs** (e.g., `mnaas_traffic_details_billing.hql`, `mnaas_traffic_details_billing_late_cdr.hql`) | These scripts query the view to compute monthly invoice lines, apply tariffs, and generate settlement files. |
| **Monthly calendar loader** (`month_billing` table population) | A separate ETL populates `month_billing` with `start_date`/`end_date` for each billing cycle; this view depends on that data being present before execution. |
| **Raw CDR ingestion pipeline** (`interconnect_traffic_details_raw` loader) | The raw CDR table is refreshed daily/ hourly; the view always reflects the latest data that falls within the month window. |
| **Reporting dashboards** (e.g., BI tools) | Analysts may query the view directly for ad‑hoc PPU analysis. |
| **Orchestration layer** (Airflow/Oozie) | The view‑creation script is typically scheduled as a **pre‑billing** task, executed after the raw CDR and month calendar loads complete. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Schema drift** – source tables gain/lose columns used in the view. | View creation fails; downstream jobs break. | Add a schema‑validation step (e.g., `DESCRIBE` checks) before `CREATE VIEW`. |
| **Full table scan** – missing partition pruning on `interconnect_traffic_details_raw`. | High Hive/Impala resource consumption, possible OOM. | Ensure `partition_date` is a table partition; add `PARTITIONED BY (partition_date)` if not already. |
| **Duplicate CDRs not resolved** – `row_number()` ordering may pick the wrong file version. | Incorrect billing amounts. | Verify that `filename` lexical order reliably reflects ingestion time; consider using a timestamp column instead. |
| **Incorrect month window** – `month_billing` contains overlapping or missing rows. | Data leakage across months. | Enforce a unique constraint on `(start_date, end_date)` in `month_billing`; add a sanity check that exactly one row matches the target month. |
| **View recreation race condition** – concurrent runs may drop/recreate the view while another job reads it. | Transient query failures. | Use `CREATE OR REPLACE VIEW` (already implied) and schedule the task with exclusive locks or a “run‑once” flag. |

---

## 6. Execution & Debugging Guide  

1. **Run the script** (typical in a CI/CD or orchestration job):  
   ```bash
   hive -f mediation-ddls/mnaas_v_ictraffic_details_ppu_billing_month.hql
   # or for Impala:
   impala-shell -f mediation-ddls/mnaas_v_ictraffic_details_ppu_billing_month.hql
   ```

2. **Validate creation**:  
   ```sql
   SHOW CREATE VIEW mnaas.v_ictraffic_details_ppu_billing_month;
   ```

3. **Sample data check** (limit to a few rows):  
   ```sql
   SELECT * FROM mnaas.v_ictraffic_details_ppu_billing_month LIMIT 10;
   ```

4. **Debugging tips**:  
   * If the view fails to create, inspect the Hive/Impala logs for “semantic error” messages – they usually point to missing columns.  
   * Run the inner CTE alone to verify filtering logic:  
     ```sql
     SELECT * FROM (
       SELECT ... FROM mnaas.interconnect_traffic_details_raw, mnaas.month_billing
       WHERE partition_date BETWEEN start_date AND end_date
     ) filtered_cdr LIMIT 20;
     ```  
   * Check that `month_billing` contains the expected `start_date`/`end_date` for the target month:  
     ```sql
     SELECT start_date, end_date FROM mnaas.month_billing WHERE ...;
     ```

5. **Operator checklist** before execution:  
   * Confirm that the latest raw CDR load has completed.  
   * Verify that the month calendar entry for the current billing cycle exists.  
   * Ensure no other job is currently recreating the same view.

---

## 7. External Configuration & Environment Variables  

| Config Item | Usage |
|-------------|-------|
| Hive/Impala connection settings (e.g., `HIVE_CONF_DIR`, `IMPALA_HOST`) | Determines which cluster the DDL is applied to. |
| `HADOOP_USER_NAME` (or similar) | Provides the identity under which the view is created – must have `CREATE` privileges on schema `mnaas`. |
| Optional **runtime parameters** (if the orchestration framework injects them) | Some pipelines pass the target month as `--hivevar start_date='2024-09-01' --hivevar end_date='2024-09-30'`; the script currently relies on the `month_billing` table, but a future version could accept these as variables. |

---

## 8. Suggested Improvements (TODO)

1. **Add explicit join condition** – replace the implicit cross‑join with an `INNER JOIN` on a common key (e.g., `partition_date`) to make the query intent clearer and avoid accidental Cartesian products if future columns are added.
2. **Parameterise month boundaries** – accept `start_date` and `end_date` as Hive variables, falling back to the `month_billing` table. This makes the view reusable for ad‑hoc month builds and simplifies testing.  

---