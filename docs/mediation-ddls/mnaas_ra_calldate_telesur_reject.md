**File:** `mediation-ddls\mnaas_ra_calldate_telesur_reject.hql`  

---

## 1. High‑Level Summary
This HQL script creates (or replaces) the Hive view `mnaas.ra_calldate_telesur_reject`. The view aggregates traffic records that have been flagged as “Telesur addon reject” and provides per‑file, per‑day, per‑month, per‑organization metrics (CDR count and usage) together with a static reason string. It is part of a family of “*_reject” views that isolate rejected call‑detail records (CDRs) for downstream reporting, billing adjustments, and exception handling.

---

## 2. Core Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `mnaas.ra_calldate_telesur_reject` | Hive **VIEW** | Exposes a filtered, grouped summary of rejected Telesur‑addon CDRs. |
| `mnaas.ra_calldate_traffic_table` | Hive **TABLE** | Source CDR data (raw or pre‑processed) containing columns used in the view (`filename`, `processed_date`, `callmonth`, `tcl_secs_id`, `orgname`, `proposition_addon`, `country`, `usage_type`, `cdr_count`, `bytes_sec_sms`, `calltype`, `telesur_reject`, `tadigcode`, `file_prefix`, `row_num`). |
| `mnaas.org_details` | Hive **TABLE** | Lookup table that maps `tcl_secs_id` to organization metadata (`orgno`, `orgname`). |
| `createtab_stmt` | Script variable (convention) | Holds the DDL statement; used by the orchestration framework to execute the CREATE VIEW. |

*No procedural code (functions, classes) is present; the script is a declarative DDL definition.*

---

## 3. Data Flow, Inputs, Outputs & Side‑Effects  

| Aspect | Details |
|--------|---------|
| **Inputs** | - `mnaas.ra_calldate_traffic_table` (raw traffic) <br> - `mnaas.org_details` (org metadata) |
| **Filters Applied** | `telesur_reject = 'Y'` <br> `proposition_addon != ''` <br> `country != ''` <br> `tadigcode != ''` <br> `file_prefix != 'SNG'` <br> `filename NOT LIKE '%NonMOVE%'` <br> `row_num = 1` |
| **Aggregations** | `SUM(cdr_count)` → `cdr_count` <br> `SUM( CASE … )` → `cdr_usage` (MB for Data, minutes for Voice, raw for others) |
| **Outputs** | Hive view `mnaas.ra_calldate_telesur_reject` (virtual table). |
| **Side‑Effects** | Creation (or replacement) of the view in the `mnaas` database. No physical data is written. |
| **Assumptions** | - All referenced columns exist with expected data types. <br> - The underlying tables are refreshed before this view is built (e.g., daily load job). <br> - Hive metastore is reachable and the executing user has `CREATE VIEW` privileges. |

---

## 4. Integration with the Rest of the System  

| Connected Component | Relationship |
|---------------------|--------------|
| **Sibling reject‑view scripts** (`*_ic_partner_reject.hql`, `*_ic_geneva_reject.hql`, etc.) | Same pattern; downstream jobs may UNION all reject views for a consolidated “rejects” dataset. |
| **ETL orchestration (e.g., Oozie / Airflow)** | The script is typically invoked as a *DDL* task after the daily traffic table load. The orchestration framework reads the `createtab_stmt` variable and runs it via Hive CLI / Beeline. |
| **Reporting / Billing jobs** | Consumers query `mnaas.ra_calldate_telesur_reject` to generate exception reports, trigger manual adjustments, or feed downstream data‑warehouses. |
| **Data quality monitoring** | A separate validation job may compare row counts from the view against source table counts to detect unexpected drops. |
| **Configuration files** | The script relies on the default Hive connection settings defined in the environment (e.g., `hive-site.xml`, Kerberos principal). No explicit external config is referenced inside the file. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – underlying tables change (column rename, type change) | View creation fails; downstream jobs break. | Add a schema‑validation step before view creation; version‑control DDL with explicit column lists. |
| **Large full‑table scans** – missing partitions or filters cause heavy Hive jobs | Resource exhaustion, long runtimes. | Ensure `ra_calldate_traffic_table` is partitioned (e.g., by `callmonth` or `processed_date`) and add partition predicates if possible. |
| **Incorrect reject flag** – data quality issue where `telesur_reject` flag is mis‑populated | Wrong records appear in the reject view, leading to false alerts. | Implement upstream data‑quality checks that verify flag consistency; log counts of rejected rows for audit. |
| **Permission issues** – user lacks `CREATE VIEW` or cannot read source tables | Job aborts, no view generated. | Use a dedicated service account with documented privileges; test permissions in a staging environment. |
| **Naming collision** – view already exists with incompatible definition | Hive throws “already exists” error. | Use `CREATE OR REPLACE VIEW` (if supported) or drop‑and‑recreate pattern with explicit logging. |

---

## 6. Running / Debugging the Script  

1. **Typical execution (via orchestration)**  
   ```bash
   # Example Oozie action (shell or Hive action)
   hive -e "$(cat mediation-ddls/mnaas_ra_calldate_telesur_reject.hql)"
   ```
   The orchestration engine extracts the `createtab_stmt` variable and passes it to Hive/Beeline.

2. **Manual run (developer)**  
   ```bash
   hive -f mediation-ddls/mnaas_ra_calldate_telesur_reject.hql
   ```
   Verify the view appears:  
   ```sql
   SHOW VIEWS IN mnaas;
   DESCRIBE FORMATTED mnaas.ra_calldate_telesur_reject;
   ```

3. **Debugging steps**  
   - Run the SELECT portion alone to inspect row counts:  
     ```sql
     SELECT COUNT(*) FROM (
       SELECT ... FROM mnaas.ra_calldate_traffic_table
       LEFT JOIN mnaas.org_details ON tcl_secs_id = orgno
       WHERE telesur_reject='Y' AND ... 
     ) t;
     ```  
   - Check for missing columns: `DESCRIBE mnaas.ra_calldate_traffic_table;`  
   - Review Hive job logs for “SemanticException” messages.  

4. **Validation** (post‑creation)  
   ```sql
   SELECT COUNT(*) AS reject_rows,
          SUM(cdr_count) AS total_cdrs,
          SUM(cdr_usage) AS total_usage
   FROM mnaas.ra_calldate_telesur_reject;
   ```

---

## 7. External Configuration / Environment Variables  

| Item | Usage |
|------|-------|
| Hive metastore connection (`hive.metastore.uris`) | Implicit – required for any Hive DDL. |
| Kerberos principal / keytab | Implicit – if the cluster is secured. |
| `mnaas` database name | Hard‑coded in the script; any change requires script edit. |
| No explicit config files or env vars are referenced inside this HQL file. |

If the deployment environment uses a variable substitution mechanism (e.g., `${DB_NAME}`), the current script does **not** leverage it; consider externalising the database name for flexibility.

---

## 8. Suggested Improvements (TODO)

1. **Add `CREATE OR REPLACE VIEW` semantics** (or a pre‑drop step) to make the script idempotent across re‑runs, reducing orchestration complexity.  
2. **Document partitioning strategy**: If `ra_calldate_traffic_table` is partitioned, add a partition filter (e.g., `WHERE callmonth = '${process_month}'`) and expose the month as a parameter to limit scan size and improve performance.  

---