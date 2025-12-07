**File:** `mediation-ddls\mnaas_ra_calldate_duplicate_reject.hql`  

---

## 1. High‑Level Summary
This script creates the Hive view `mnaas.ra_calldate_duplicate_reject`. The view isolates Call Detail Records (CDRs) that have been identified as duplicates (i.e., `row_num > 1`) from the raw traffic table, excludes certain file prefixes and non‑MOVE files, aggregates the duplicate count and usage per logical grouping, and tags each row with the constant rejection reason **“Not Billable – Duplicate CDRs”**. The view is consumed downstream by billing‑reject pipelines and reporting jobs to ensure duplicate CDRs are not billed.

---

## 2. Core Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `ra_calldate_duplicate_reject` | Hive **VIEW** | Presents a de‑duplicated, aggregated snapshot of duplicate CDRs with a fixed rejection reason. |
| `ra_calldate_traffic_table` | Hive **TABLE** (source) | Holds raw, per‑call CDR data, including `row_num`, `filename`, `processed_date`, `callmonth`, `tcl_secs_id`, `orgname`, `proposition_addon`, `country`, `usage_type`, `calltype`, `bytes_sec_sms`, `cdr_count`. |
| `org_details` | Hive **TABLE** (lookup) | Provides organization metadata (`orgno` → `orgname`, etc.) for enrichment of the view. |
| `createtab_stmt` | Variable (script placeholder) | Holds the DDL string that Hive executes; in this file it contains the `CREATE VIEW` statement. |

*No procedural functions or classes are defined; the file is a pure DDL definition.*

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Inputs** | - `mnaas.ra_calldate_traffic_table` (must contain the columns used in the SELECT).<br>- `mnaas.org_details` (joined on `tcl_secs_id = orgno`). |
| **Outputs** | - Hive **VIEW** `mnaas.ra_calldate_duplicate_reject` (persisted in the metastore). |
| **Side Effects** | - Overwrites the view if it already exists (Hive `CREATE VIEW` without `IF NOT EXISTS`). |
| **Assumptions** | - `row_num` is generated upstream (likely via a window function) and correctly identifies duplicate rows (`row_num > 1`).<br>- `file_prefix` and `filename` columns exist and follow naming conventions used elsewhere (e.g., `SNG` prefix, `NonMOVE` suffix).<br>- `calltype` values are limited to `'Data'`, `'Voice'`, `'ICVOICE'` (others fall into the generic `bytes_sec_sms` conversion).<br>- The Hive metastore is reachable and the `mnaas` database exists. |

---

## 4. Integration Points  

| Connected Script / Component | Relationship |
|------------------------------|--------------|
| **`mnaas_ra_calldate_traffic_table` creation scripts** (e.g., `mnaas_ra_calldate_cust_monthly_rep.hql`) | Provide the raw CDR table that this view reads from. |
| **`mnaas_ra_calldate_billing_reject.hql`**, **`mnaas_ra_calldate_country_reject.hql`**, **`mnaas_ra_calldate_cust_monthly_rep.hql`** | Sibling reject‑reason views; downstream jobs union multiple reject views to build a master “non‑billable” dataset. |
| **Billing Orchestration Jobs** (e.g., nightly Spark/MapReduce jobs) | Consume `ra_calldate_duplicate_reject` to exclude duplicate CDRs from invoicing. |
| **Reporting Dashboards** (e.g., Tableau/PowerBI via Hive ODBC) | Query the view for operational metrics on duplicate CDR volume. |
| **Data Quality Monitoring** (e.g., Airflow DAGs) | May run a `SELECT COUNT(*) FROM ra_calldate_duplicate_reject` to alert on spikes. |

*Because the view is stored in the Hive metastore, any downstream process that references `mnaas.ra_calldate_duplicate_reject` will automatically see the latest definition after this script runs.*

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Source table schema drift** (e.g., column rename or type change) | View creation fails or returns incorrect data. | Add a pre‑deployment validation step that checks column existence/types (`DESCRIBE` + assertions). |
| **Missing `row_num` column** (duplicate detection logic not executed upstream) | All rows may be treated as duplicates or none at all. | Enforce a contract in upstream DAGs that `row_num` is populated; fail fast if absent. |
| **Large data volume** causing long view creation time or OOM in Hive Metastore. | Job timeout, resource contention. | Partition the view by `callmonth` or `processed_date` if supported; monitor Hive job duration. |
| **Hard‑coded database name (`mnaas`)** limits reuse across environments (dev, test, prod). | Deployment errors in non‑prod clusters. | Externalize the database name via a Hive variable or config file (e.g., `${hiveconf:DB_NAME}`). |
| **Silent overwrite of existing view** may hide unintended changes. | Operators may not notice that view definition changed. | Use `CREATE OR REPLACE VIEW` with versioned comments, or add a changelog table entry after each deployment. |

---

## 6. Running / Debugging the Script  

1. **Execute the DDL**  
   ```bash
   hive -f mediation-ddls/mnaas_ra_calldate_duplicate_reject.hql
   # or, with Beeline:
   beeline -u jdbc:hive2://<host>:10000/mnaas -f mediation-ddls/mnaas_ra_calldate_duplicate_reject.hql
   ```

2. **Validate Creation**  
   ```sql
   SHOW CREATE VIEW mnaas.ra_calldate_duplicate_reject;
   ```

3. **Sample Data Check**  
   ```sql
   SELECT * FROM mnaas.ra_calldate_duplicate_reject LIMIT 20;
   ```

4. **Verify Duplicate Logic**  
   ```sql
   SELECT filename, COUNT(*) AS dup_rows
   FROM mnaas.ra_calldate_traffic_table
   WHERE row_num > 1
   GROUP BY filename
   HAVING dup_rows > 0
   LIMIT 10;
   ```

5. **Performance Debug**  
   - Run `EXPLAIN EXTENDED SELECT ... FROM mnaas.ra_calldate_duplicate_reject LIMIT 1;` to see the execution plan.  
   - Check Hive logs (`/var/log/hive/`) for any “SemanticException” or “OutOfMemoryError”.  

---

## 7. External Configuration & Environment Variables  

| Config / Variable | Usage |
|-------------------|-------|
| `hive.metastore.uris` | Hive client must be able to reach the metastore to create the view. |
| `DB_NAME` (optional) | If the script is refactored, this variable could replace the hard‑coded `mnaas` schema. |
| `HIVE_CONF_DIR` | Points to Hive configuration (e.g., `hive-site.xml`) that defines default file system, execution engine, etc. |
| No explicit env‑vars are referenced in the current file; all identifiers are literal. |

---

## 8. Suggested Improvements (TODO)

1. **Parameterize the database/schema name** – replace the literal `mnaas` with a Hive variable (`${hiveconf:DB_NAME}`) to enable the same script across dev/test/prod clusters without modification.  
2. **Add a view comment** that documents the duplicate‑detection rule (`row_num > 1`) and the exclusion criteria (`file_prefix != 'SNG'`, `filename NOT LIKE '%NonMOVE%'`). This improves discoverability for downstream teams.  

---