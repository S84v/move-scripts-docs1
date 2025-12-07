**File:** `mediation-ddls\mnaas_v_traffic_details_billing.hql`  

---

## 1. High‑Level Summary
This script creates the Hive/Impala view `mnaas.v_traffic_details_billing`. The view consolidates raw CDR records from `traffic_details_raw_daily` (joined with `month_billing`) into a billing‑ready dataset. It filters out non‑MOVE files, applies a series of business rules (e.g., usage calculation, proposition add‑on selection, telesur reject flag), de‑duplicates records using `row_number()`, and then unions four result sets to handle:
* Data/Voice CDRs (both directions)  
* SMS MT CDRs  
* SMS MO CDRs (regular)  
* SMS MO A2P CDRs (where the calling party is on the A2P exclusion list)  

The resulting view is consumed by downstream billing aggregation jobs (e.g., `mnaas_traffic_details_billing.hql`) and reporting layers.

---

## 2. Core Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `v_traffic_details_billing` | **VIEW** | Exposes a cleaned, filtered, and de‑duplicated set of CDRs ready for billing calculations. |
| `filtered_cdr` | **CTE** | Performs the heavy lifting: joins raw daily CDRs with month‑billing metadata, applies file‑name, date, and business‑rule filters, computes derived columns (`usage`, `actualusage`, `proposition_addon`, `usagefor`, `telesur_reject`, `gen_filename`). |
| `row_number() OVER (PARTITION BY cdrid, chargenumber ORDER BY cdrversion DESC)` | **Window function** | Keeps only the latest version of a CDR per `cdrid`/`chargenumber`. |
| `decode()` / `CASE` expressions | **SQL expressions** | Translate call types to usage values, decide proposition add‑on, flag telesur rejects, and map used‑type to a readable label. |
| `billing_a2p_calling_party` | **TABLE** | Provides the list of calling parties that should be treated as A2P traffic. |
| `month_billing` | **TABLE** | Supplies billing period metadata (used for the `BETWEEN start_date AND end_date` filter). |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Input tables** | `mnaas.traffic_details_raw_daily`, `mnaas.month_billing`, `mnaas.billing_a2p_calling_party` |
| **External parameters** | Hive variables `start_date` and `end_date` (expected to be supplied at runtime). |
| **Derived columns** | `sponsor` (first 5 digits of IMSI), `usage`, `actualusage`, `proposition_addon`, `usagefor`, `telesur_reject`, `gen_filename` (prefixed with `CDR_`). |
| **Output** | Creation/overwrite of view `mnaas.v_traffic_details_billing`. |
| **Side effects** | DDL operation (CREATE VIEW). No data mutation; view definition is stored in the metastore. |
| **Assumptions** | • Hive/Impala engine with support for `WITH`, `ROW_NUMBER`, `DECODE`. <br>• `traffic_details_raw_daily` is partitioned by `partition_date`. <br>• Files whose names contain “HOL” are the only ones to be processed; “NonMOVE” files must be excluded. <br>• `tcl_secs_id`, `usedtype`, `balancetypeid` columns exist and contain expected codes. |

---

## 4. Integration Points  

| Connected Component | Relationship |
|---------------------|--------------|
| **`mnaas_traffic_details_raw_daily.hql`** | Populates `traffic_details_raw_daily` that this view reads. |
| **`mnaas_month_billing.hql`** (or similar) | Populates `month_billing` used for date range filtering. |
| **`mnaas_billing_a2p_calling_party.hql`** | Maintains the exclusion list referenced in the A2P UNION branch. |
| **Downstream billing scripts** (e.g., `mnaas_traffic_details_billing.hql`, `mnaas_v_ictraffic_details_ppu_billing.hql`) | Query `v_traffic_details_billing` to compute charges, generate invoices, or feed reporting dashboards. |
| **Orchestration layer** (Oozie / Airflow) | Likely runs this DDL as a “create view” step before the billing aggregation jobs. |
| **Parameter store / config** | `start_date` / `end_date` are typically injected from a job configuration file or environment variables (e.g., Hive `-hiveconf start_date=2024-01-01`). |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or malformed `start_date`/`end_date`** | View creation fails; downstream jobs receive no data. | Validate parameters at job start; fail fast with clear log message. |
| **Large UNION causing long compile time** | Delays in nightly run windows. | Ensure tables are partition‑pruned (filter on `partition_date`). Consider materializing intermediate filtered tables or using a *materialized view* if supported. |
| **Incorrect `telesur_reject` logic** (e.g., wrong `balancetypeid` values) | Wrong CDRs may be billed or omitted. | Add unit‑test queries on a sample dataset; keep a reference table for reject rules. |
| **Row‑number de‑duplication may drop needed versions** | Billing on stale data. | Verify that `cdrversion` monotonicity holds; optionally keep a “latest_version” flag for audit. |
| **A2P exclusion list out‑of‑sync** | Mis‑classification of traffic. | Schedule regular refresh of `billing_a2p_calling_party`; add alert if row count changes dramatically. |
| **Schema drift (new columns added to source tables)** | View creation may break. | Include schema‑validation step or use `SELECT *` with explicit column list versioned. |

---

## 6. Running / Debugging the Script  

1. **Set required Hive variables** (example via CLI):  
   ```bash
   hive -hiveconf start_date=2024-01-01 -hiveconf end_date=2024-01-31 -f mediation-ddls/mnaas_v_traffic_details_billing.hql
   ```
2. **Verify view creation**:  
   ```sql
   SHOW CREATE VIEW mnaas.v_traffic_details_billing;
   ```
3. **Sample data check**:  
   ```sql
   SELECT COUNT(*) AS total_rows,
          SUM(CASE WHEN calltype='Data' THEN 1 ELSE 0 END) AS data_cnt,
          SUM(CASE WHEN calltype='Voice' THEN 1 ELSE 0 END) AS voice_cnt,
          SUM(CASE WHEN calltype='SMS' THEN 1 ELSE 0 END) AS sms_cnt
   FROM mnaas.v_traffic_details_billing;
   ```
4. **Inspect a problematic CDR** (replace `<cdrid>`):  
   ```sql
   SELECT * FROM mnaas.v_traffic_details_billing WHERE cdrid = '<cdrid>';
   ```
5. **Log inspection**: In the orchestration UI (Oozie/Airflow), view the Hive task logs for any “SemanticException” or “AnalysisException” messages.

---

## 7. External Config / Environment Dependencies  

| Item | Usage |
|------|-------|
| Hive variables `start_date`, `end_date` | Define the billing period window for the view. |
| Metastore connection (JDBC URL, Kerberos tickets) | Required for view creation. |
| `mnaas` database location (HDFS path) | Determines where the view metadata is stored. |
| Optional: `hive.exec.dynamic.partition` settings | May be needed if the view is later used in INSERT … SELECT with dynamic partitions. |

---

## 8. Suggested Improvements (TODO)

1. **Parameter Validation Block** – Add a pre‑execution `SET` block that checks `start_date` and `end_date` are non‑null and that `start_date <= end_date`. Abort with a clear error if validation fails.
2. **Performance Tuning** – Replace the four‑way `UNION` with a single SELECT that uses conditional logic (`CASE WHEN … THEN … END`) where possible, or create a temporary filtered table and apply a single `INSERT OVERWRITE VIEW` to reduce planner overhead.  

---