**File:** `mediation-ddls\mnaas_v_traffic_details_ppu_billing.hql`

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive view `mnaas.v_traffic_details_ppu_billing`.  
The view consolidates raw CDR records for Pay‑Per‑Use (PPU) billing, filtering on file‑name patterns, time windows, usage type, and destination type. It normalises usage fields, resolves the applicable commercial proposition (subscription vs. add‑on), and produces a unified record set that distinguishes Data/Voice, SMS‑MT, SMS‑MO, and A2P‑SMS‑MO traffic. Down‑stream billing and reporting jobs query this view to generate daily PPU invoices and analytics.

---

## 2. Key Components & Responsibilities  

| Component | Responsibility |
|-----------|-----------------|
| **`CREATE VIEW … AS WITH filtered_cdr AS (…)`** | Defines a Common Table Expression (`filtered_cdr`) that pre‑filters raw CDR rows and adds derived columns (`sponsor`, `usage`, `actualusage`, `usagefor`, `gen_filename`, `row_num`). |
| **`filtered_cdr` CTE** | Joins `mnaas.traffic_details_raw_daily` with `mnaas.month_ppu_billing` and applies the core business rules: file‑name inclusion/exclusion, insertion‑time window (`start_inserttime`/`end_inserttime`), positive `tcl_secs_id`, allowed `usedtype`, `destinationtype`, non‑empty country/tadigcode/imsi, and permitted `calltype`/`traffictype`. |
| **`row_number() …`** | Guarantees a deterministic “latest version” per (`cdrid`, `chargenumber`) by ordering on `cdrversion` descending. |
| **First SELECT (Data/Voice)** | Returns one row per CDR for Data or Voice calls (`row_num = 1`). |
| **Second SELECT (SMS‑MT)** | Returns one row per CDR for SMS Mobile‑Terminated traffic (`calltype='SMS' AND traffictype='MT'`). |
| **Third SELECT (SMS‑MO, non‑A2P)** | Returns one row per CDR for SMS Mobile‑Originated traffic where the called party is not an A2P excluded number and the calling party is **not** flagged as A2P in `billing_a2p_calling_party`. |
| **Fourth SELECT (A2P‑SMS‑MO)** | Same as the third SELECT but for calling parties **marked** as A2P; the `calltype` column is forced to `'A2P'`. |
| **`proposition_addon` expression** | Chooses the commercial offer description for subscription traffic or the active add‑on name for add‑on traffic. |
| **`timebandindex`** | Hard‑coded to `0` (placeholder for future time‑band logic). |

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Input Tables / Views** | `mnaas.traffic_details_raw_daily` (raw CDRs), `mnaas.month_ppu_billing` (billing period metadata), `mnaas.billing_a2p_calling_party` (A2P exclusion list). |
| **External Variables** | `start_inserttime` and `end_inserttime` – Hive variables that must be set before execution (e.g., `--hiveconf start_inserttime=2024‑01‑01_00:00:00`). |
| **Output** | Hive view `mnaas.v_traffic_details_ppu_billing`. The view is a logical object; no physical data is written at creation time. |
| **Side Effects** | DDL operation – creates or replaces the view. If the view already exists, it is overwritten, which may temporarily invalidate dependent queries. |
| **Assumptions** | • The source tables are partitioned by `partition_date` and are up‑to‑date for the requested insertion window.<br>• `filename` values contain the substring `'HOL'` for “home‑location” CDRs and never contain `'NonMOVE'` for the same set.<br>• `usedtype` values are limited to `'0'`, `'1'`, `'2'` (subscription, subscription duplicate, add‑on).<br>• All referenced columns exist with the expected data types (e.g., `cdrversion` numeric, `calltype` string). |

---

## 4. Integration Points  

| Connected Component | Relationship |
|---------------------|--------------|
| **Up‑stream ingestion jobs** (e.g., `mnaas_traffic_pre_active_raw_sim.hql`, `mnaas_traffic_details_raw_reject_daily.hql`) | Populate `traffic_details_raw_daily` and set the `filename` pattern used in this view. |
| **Month‑level billing tables** (`mnaas.month_ppu_billing`) | Provide the temporal window (`start_inserttime`, `end_inserttime`) and may contain additional billing metadata referenced indirectly. |
| **A2P exclusion list** (`mnaas.billing_a2p_calling_party`) | Determines whether an SMS‑MO record is classified as regular or A2P. |
| **Down‑stream billing views** (`mnaas_v_traffic_details_billing.hql`, `mnaas_v_traffic_details_billing_late_cdr.hql`) | Consume `v_traffic_details_ppu_billing` to calculate daily PPU charges, generate invoices, and feed revenue assurance processes. |
| **Reporting / analytics pipelines** (e.g., Tableau, PowerBI extracts) | Query the view for operational dashboards on PPU usage. |
| **Orchestration layer** (Airflow, Oozie, or custom scheduler) | Executes this DDL as part of a daily “build‑billing‑views” DAG, typically after the raw‑CDR load completes. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing Hive variables** (`start_inserttime` / `end_inserttime`) | Script fails, view not created, downstream jobs break. | Validate variables at the start of the DAG; provide default safe window or abort with clear error. |
| **View recreation during peak query time** | Dependent jobs may see a transient “view not found” or stale schema. | Schedule view rebuild during low‑traffic windows; use `CREATE OR REPLACE VIEW` (already implied) and ensure downstream jobs have retry logic. |
| **Data‑skew in `row_number()` partition** | Excessive memory consumption on a single reducer, job failure. | Verify that `cdrid` + `chargenumber` provides sufficient cardinality; consider adding additional partition keys if needed. |
| **Incorrect filename pattern** (`%HOL%` / `%NonMOVE%`) | Unexpected records included/excluded, leading to billing errors. | Keep pattern definitions in a central config file; add unit tests that validate a sample of filenames against the pattern. |
| **A2P exclusion list out‑of‑sync** | Mis‑classification of SMS traffic (billing under wrong rate). | Refresh `billing_a2p_calling_party` daily; add a data‑quality check that flags new MSISDNs not present in the list. |
| **Hard‑coded `timebandindex = 0`** | Future time‑band logic may be silently ignored. | Document the placeholder and add a TODO to replace with dynamic calculation when the feature is ready. |

---

## 6. Example Execution & Debugging  

1. **Run the script (Hive CLI / Beeline)**  
   ```bash
   hive --hiveconf start_inserttime=2024-01-01_00:00:00 \
        --hiveconf end_inserttime=2024-01-01_23:59:59 \
        -f mediation-ddls/mnaas_v_traffic_details_ppu_billing.hql
   ```

2. **Validate view creation**  
   ```sql
   SHOW CREATE VIEW mnaas.v_traffic_details_ppu_billing;
   ```

3. **Sample data check** (run after view creation)  
   ```sql
   SELECT calltype, traffictype, COUNT(*) AS cnt
   FROM mnaas.v_traffic_details_ppu_billing
   GROUP BY calltype, traffictype
   ORDER BY cnt DESC
   LIMIT 20;
   ```

4. **Debugging tips**  
   * If the script fails with “Undefined variable”, verify that the two Hive variables are passed.  
   * To isolate filtering logic, run the CTE query alone:  
     ```sql
     WITH filtered_cdr AS ( … original CTE … )
     SELECT * FROM filtered_cdr LIMIT 10;
     ```  
   * Check for data‑skew by inspecting the distribution of `cdrid` values:  
     ```sql
     SELECT cdrid, COUNT(*) FROM filtered_cdr GROUP BY cdrid HAVING COUNT(*) > 1000;
     ```

---

## 7. External Config / Environment Dependencies  

| Item | Usage |
|------|-------|
| **Hive variables** `start_inserttime`, `end_inserttime` | Define the insertion‑time window for the raw CDRs to be considered. |
| **File‑name pattern constants** (`%HOL%`, `%NonMOVE%`) | Hard‑coded in the script; may be externalised to a properties file for easier change management. |
| **Database / schema** `mnaas` | All referenced tables and the view reside in this schema; the schema must be accessible to the execution user. |
| **Cluster resources** (YARN memory, Hive execution engine) | Large joins and window functions require sufficient container memory; monitor job logs for OOM warnings. |

---

## 8. Suggested Improvements (TODO)

1. **Externalise filter patterns & constants** – Move `%HOL%`, `%NonMOVE%`, and the list of excluded called parties (`'3159272','59272'`) into a configuration table or properties file. This reduces the need for code changes when business rules evolve.

2. **Add a partition column to the view** – If downstream processes query by `partition_date`, consider adding `PARTITIONED BY (partition_date)` to the view definition (or create a materialised table) to improve query pruning and performance.  

---