**File:** `mediation-ddls\mnaas_v_traffic_details_ppu_billing_month.hql`  

---

## 1. High‑Level Summary
This script creates the Hive/Impala view `mnaas.v_traffic_details_ppu_billing_month`. The view aggregates raw CDR records (voice, data, SMS) for the current billing month, filters out rejected or duplicate records, enriches each record with derived fields (e.g., sponsor, usage, proposition add‑on), and normalises A2P‑SMS traffic. The resulting dataset is the canonical source for “pay‑per‑use” (PPU) monthly billing calculations downstream in the mediation pipeline.

---

## 2. Core Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `v_traffic_details_ppu_billing_month` | **VIEW** | Exposes a cleaned, de‑duplicated, month‑scoped set of CDRs ready for PPU billing. |
| `filtered_cdr` | **CTE** (Common Table Expression) | Pulls raw daily CDRs (`traffic_details_raw_daily`) and joins to `month_billing` to restrict rows to the current month, applies extensive validation, computes derived columns, and assigns a row‑number per `(cdrid, chargenumber)` to keep the latest version. |
| `traffic_details_raw_daily` | **TABLE** (source) | Holds the raw, unprocessed CDRs ingested each day. |
| `month_billing` | **TABLE** (lookup) | Provides the billing month window (`start_date`, `end_date`) used to filter the raw CDRs. |
| `billing_a2p_calling_party` | **TABLE** (lookup) | Stores MSISDNs that are flagged as “A2P excluded”; used to separate regular SMS from A2P‑SMS traffic. |

---

## 3. Inputs, Outputs & Side‑Effects  

| Category | Details |
|----------|---------|
| **Inputs** | - `mnaas.traffic_details_raw_daily` (daily raw CDRs) <br> - `mnaas.month_billing` (contains `start_date`, `end_date` for the current month) <br> - `mnaas.billing_a2p_calling_party` (list of excluded MSISDNs for A2P detection) |
| **Outputs** | - Hive/Impala **VIEW** `mnaas.v_traffic_details_ppu_billing_month` (no physical data written, but metadata created). |
| **Side‑Effects** | - DDL operation: creates or replaces the view. <br> - No data mutation (INSERT/UPDATE/DELETE). |
| **Assumptions** | - `month_billing` always contains a single row for the active month. <br> - `traffic_details_raw_daily` partitions include a `partition_date` column that aligns with the month window. <br> - Filenames follow the pattern `…HOL…` for “home‑office‑load” and never contain `NonMOVE`. <br> - `tcl_secs_id`, `usedtype`, `balancetypeid` values are stable and documented elsewhere. |

---

## 4. Integration Points  

| Connected Component | How It Links |
|---------------------|--------------|
| **Earlier scripts** (e.g., `mnaas_traffic_details_raw_reject_daily.hql`, `mnaas_traffic_pre_active_raw_sim.hql`) | Populate `traffic_details_raw_daily` and perform early reject filtering; this view consumes that cleaned raw table. |
| **Month definition script** (`mnaas_month_billing.hql`) | Supplies the `start_date` / `end_date` window used in the CTE filter. |
| **A2P exclusion list script** (`mnaas_billing_a2p_calling_party.hql`) | Maintains the `billing_a2p_calling_party` table referenced for A2P‑SMS segregation. |
| **Downstream billing jobs** (e.g., `mnaas_v_traffic_details_ppu_billing.hql`, `mnaas_v_traffic_details_ppu_billing_late_cdr.hql`) | Query this view to compute monthly PPU charges, generate invoices, and feed settlement systems. |
| **Orchestration layer** (Oozie / Airflow DAG) | Executes this DDL as part of the “monthly‑billing‑view‑refresh” workflow, typically after the raw‑daily ingestion and month‑window tables are ready. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Performance blow‑up** – the CTE joins the entire raw daily table with `month_billing` and computes a `row_number()` over potentially millions of rows. | Long view creation time; possible OOM in Hive/Impala. | • Ensure `traffic_details_raw_daily` is partitioned on `partition_date`. <br>• Add a predicate push‑down on `partition_date` before the join (already present, but verify partition pruning). <br>• Consider materialising the view as a table with periodic refresh if latency becomes unacceptable. |
| **Incorrect month window** – if `month_billing` contains multiple rows or an unexpected date range, the view may include out‑of‑scope CDRs. | Billing inaccuracies. | • Add a validation step in the orchestration DAG that asserts exactly one active month row. |
| **Filename pattern drift** – reliance on `filename LIKE '%HOL%'` and exclusion of `'%NonMOVE%'` may miss new file naming conventions. | Missing or extra records. | • Centralise filename patterns in a configuration table; reference that table instead of hard‑coded literals. |
| **A2P exclusion list stale** – outdated entries in `billing_a2p_calling_party` could mis‑classify traffic. | Revenue leakage or over‑charging. | • Schedule a daily refresh of the exclusion list from the upstream CRM system. |
| **Schema change propagation** – adding/removing columns in source tables without updating the view will cause compile errors. | Job failures. | • Include a schema‑validation step in the pipeline; version‑control view definitions alongside source schema definitions. |

---

## 6. Running / Debugging the Script  

1. **Execution** (typical in production):  
   ```bash
   hive -f mediation-ddls/mnaas_v_traffic_details_ppu_billing_month.hql
   # or via Impala:
   impala-shell -f mediation-ddls/mnaas_v_traffic_details_ppu_billing_month.hql
   ```
   The script is usually invoked by an Oozie action or Airflow task named *create_ppu_month_view*.

2. **Verification**:  
   ```sql
   DESCRIBE FORMATTED mnaas.v_traffic_details_ppu_billing_month;
   SELECT COUNT(*) FROM mnaas.v_traffic_details_ppu_billing_month LIMIT 10;
   ```
   Compare the row count with expectations from the previous month’s view.

3. **Debugging Steps**:  
   - Replace the final `SELECT … FROM filtered_cdr …` with `SELECT * FROM filtered_cdr LIMIT 100;` to inspect the CTE output.  
   - Check the `row_num` distribution: `SELECT row_num, COUNT(*) FROM filtered_cdr GROUP BY row_num;` to ensure duplicates are being collapsed correctly.  
   - Validate the month window: `SELECT start_date, end_date FROM mnaas.month_billing;`.  
   - Verify A2P exclusion logic: `SELECT COUNT(*) FROM mnaas.billing_a2p_calling_party WHERE excluded='Y';`.

---

## 7. External Configuration / Environment Dependencies  

| Item | Usage |
|------|-------|
| `mnaas.month_billing` table | Provides `start_date` / `end_date` for the month; must be populated before this script runs. |
| Hive/Impala connection settings (e.g., `hive.metastore.uris`, `impala.host`) | Defined in the cluster’s `hive-site.xml` / `impala-shell` environment; not hard‑coded in the script. |
| Optional: **Filename pattern** constants | Currently hard‑coded (`'%HOL%'`, `'%NonMOVE%'`). If a configuration table (e.g., `mnaas.file_patterns`) is introduced, the script would reference it via a sub‑query. |

---

## 8. Suggested Improvements (TODO)

1. **Parameterise filename patterns** – move `'%HOL%'` and `'%NonMOVE%'` into a configuration table or Hive variable so that changes to naming conventions require no code change.  
2. **Materialise the view** – create a partitioned table (e.g., `v_traffic_details_ppu_billing_month_tbl`) refreshed nightly; this will dramatically improve downstream query performance and allow explicit partition pruning on `partition_date`.  

---