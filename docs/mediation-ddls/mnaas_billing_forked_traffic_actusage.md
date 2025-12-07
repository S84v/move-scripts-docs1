**File:** `mediation-ddls\mnaas_billing_forked_traffic_actusage.hql`  

---

## 1. High‑Level Summary
This script creates the Hive view `mnaas.billing_forked_traffic_actusage`. The view aggregates raw forked CDR (Call Detail Record) traffic data (`billing_traffic_forked_cdr`) by month, SIM, service, call type, traffic direction, destination type, and several other dimensions. It computes both raw usage (seconds/kilobytes) and “30‑second‑rounded” usage (used for voice billing), and produces counts of CDRs per direction (MO/MT) and per destination (domestic, international/roaming). The view is a core input for downstream billing, revenue assurance, and reporting pipelines.

---

## 2. Key Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| **`cdr` CTE** | Common Table Expression | Normalises raw CDR fields, extracts `callmonth` (YYYY‑MM), and creates two usage columns: `actualusage_sec_sms` (raw) and `actualusage_30sec_sms` (rounded to 30 s for voice). |
| **`billing_forked_traffic_actusage` view** | Hive View | Performs a grouped aggregation on the `cdr` CTE, summing usage, counting records, and separating inbound/outbound and domestic/international/roaming metrics via `CASE` expressions. |
| **Source tables** | Hive tables (`mnaas.billing_traffic_forked_cdr`, `mnaas.month_billing`) | Provide raw forked CDR rows and the month‑to‑billing‑period mapping (`bill_month`). |
| **Derived columns** | – | `actualusage_kb_sec_sms`, `actualusage_100kb_30sec_sms`, `in_usage_*`, `out_usage_*`, `in_dom_*`, `out_dom_*`, `in_int_rom_*`, `out_int_rom_*`, `rom_count`, etc. – each representing a specific usage or count metric required by the billing engine. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - `mnaas.billing_traffic_forked_cdr` (raw forked CDRs) <br> - `mnaas.month_billing` (mapping of calendar month to billing month, column `bill_month` = `month`) |
| **Outputs** | - Hive view `mnaas.billing_forked_traffic_actusage` (no physical table is created; the view materialises on‑demand). |
| **Side‑Effects** | - Overwrites the view if it already exists (CREATE VIEW without `IF NOT EXISTS`). No data mutation occurs. |
| **Assumptions** | - Columns referenced in the source tables exist with the expected data types (e.g., `actualusage` numeric, `calltype` string). <br> - `bill_month` values match the `month` column in `month_billing`. <br> - `calldate` is a string where `substring(calldate,1,7)` yields `YYYY‑MM`. <br> - No NULL values in the key columns used for grouping; NULL handling is not explicitly coded. |
| **External Services** | None directly invoked. The view is consumed by downstream batch jobs, reporting dashboards, or billing micro‑services that query Hive/Presto/Trino. |

---

## 4. Integration Points  

| Connected Component | How It Connects |
|---------------------|-----------------|
| **Earlier ingestion scripts** (e.g., `mnaas_billing_traffic_forked_cdr.hql`) | Populate `billing_traffic_forked_cdr` with raw CDRs after parsing, validation, and enrichment. |
| **Month mapping script** (`mnaas_month_billing.hql`) | Supplies `month_billing` used to join on `bill_month`. |
| **Downstream billing calculations** (`mnaas_billing_*` views/tables) | Consume `billing_forked_traffic_actusage` to compute charges per SIM, per proposition, and to feed revenue‑assurance checks. |
| **Reporting dashboards** (e.g., Tableau/PowerBI via Hive/Trino) | Query the view for usage KPIs, churn analysis, and regulatory reporting. |
| **Data quality jobs** (e.g., `mnaas_billing_forked_traffic_actusage_qc.hql`) | May run validation queries against the view to ensure counts match source CDR totals. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Schema drift** – source tables change (column rename, type change) | View creation fails; downstream jobs break. | Add a schema‑validation step (e.g., `DESCRIBE` + assert) before view creation; version‑control DDL. |
| **Heavy aggregation** – full‑month CDR volume can be tens of millions of rows, causing long Hive jobs or OOM. | Job timeouts, cluster resource exhaustion. | Partition the view on `callmonth` (or create a materialised table partitioned by month). Use `SET hive.exec.reducers.bytes.per.reducer` to tune parallelism. |
| **Incorrect rounding logic** – `ceil(actualusage/30)` applied to all voice records; may double‑count if `actualusage` already in 30‑sec units. | Billing inaccuracies. | Add unit‑test queries on a known data set; document the expected unit of `actualusage`. |
| **Logical precedence in CASE statements** – expressions like `traffictype = 'MT' AND destinationtype = 'INTERNATIONAL' OR destinationtype = 'ROAMING'` may evaluate unexpectedly. | Mis‑classification of roaming usage. | Add parentheses to make intent explicit, e.g., `traffictype='MT' AND (destinationtype='INTERNATIONAL' OR destinationtype='ROAMING')`. |
| **NULL values in grouping columns** – may create a “null” bucket, inflating counts. | Data quality issues. | Coalesce key columns to a sentinel value or filter out rows with NULLs before aggregation. |

---

## 6. Running / Debugging the Script  

1. **Execute the script**  
   ```bash
   hive -f mediation-ddls/mnaas_billing_forked_traffic_actusage.hql
   # or using Beeline
   beeline -u jdbc:hive2://<host>:10000/default -f mediation-ddls/mnaas_billing_forked_traffic_actusage.hql
   ```

2. **Validate creation**  
   ```sql
   SHOW CREATE VIEW mnaas.billing_forked_traffic_actusage;
   ```

3. **Sample data check**  
   ```sql
   SELECT * FROM mnaas.billing_forked_traffic_actusage
   WHERE callmonth = '2024-09'
   LIMIT 20;
   ```

4. **Cross‑check totals** (example for a given month)  
   ```sql
   -- Total CDR count from source
   SELECT COUNT(*) FROM mnaas.billing_traffic_forked_cdr
   WHERE substring(calldate,1,7) = '2024-09';

   -- Total CDR count from view
   SELECT SUM(cdr_count) FROM mnaas.billing_forked_traffic_actusage
   WHERE callmonth = '2024-09';
   ```

5. **Log inspection**  
   - Hive logs are written to `$HIVE_LOG_DIR` (often `/tmp/hive.log`). Look for `FAILED` or `SemanticException`.  
   - If the job is long, monitor YARN UI for container memory/CPU usage.

---

## 7. External Configuration / Environment Variables  

The script itself does **not** reference external config files or environment variables. It assumes:

- The Hive metastore contains the `mnaas` database.  
- The two source tables (`billing_traffic_forked_cdr`, `month_billing`) are already populated.  

If the deployment environment uses a variable for the database name (e.g., `${DB_NAME}`), the script would need to be templated; currently it is hard‑coded.

---

## 8. Suggested Improvements (TODO)

1. **Add explicit partitioning** – Convert the view into a **materialised, partitioned table** on `callmonth` to improve query performance and reduce runtime on large data sets.  
2. **Clarify CASE precedence** – Refactor all `CASE` expressions that mix `AND`/`OR` with parentheses to avoid logical bugs, and add inline comments describing each metric’s business rule.  

---