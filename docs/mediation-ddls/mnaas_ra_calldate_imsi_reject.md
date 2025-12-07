**File:** `mediation-ddls\mnaas_ra_calldate_imsi_reject.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates (or replaces) the view `mnaas.ra_calldate_imsi_reject`. The view extracts records from the raw traffic table (`ra_calldate_traffic_table`) that have been flagged with `imsi_reject = 'Y'` (i.e., the IMSI field is missing or invalid). It joins each record to `org_details` to obtain the organization name, aggregates CDR counts and usage metrics, and tags every row with the constant reason *“IMSI is missing”*. The view is used downstream by billing‑reject, reporting, and data‑quality jobs that need to isolate IMSI‑related rejects.

---

## 2. Core Artifact(s)

| Artifact | Type | Responsibility |
|----------|------|-----------------|
| `mnaas.ra_calldate_imsi_reject` | Hive **VIEW** | Provides a consolidated, aggregated view of all traffic records rejected because the IMSI is missing. Supplies columns needed for downstream reject handling (filename, processed_date, callmonth, org, proposition, country, usage_type, aggregated CDR count, aggregated usage, reject reason). |
| `createtab_stmt` (internal variable) | HQL string | Holds the `CREATE VIEW` DDL; the script may be templated or invoked by a wrapper that extracts this variable. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Source tables** | `mnaas.ra_calldate_traffic_table` (raw CDR data) <br> `mnaas.org_details` (organization master) |
| **Required columns** | From `ra_calldate_traffic_table`: `filename, processed_date, callmonth, tcl_secs_id, proposition_addon, country, usage_type, cdr_count, bytes_sec_sms, calltype, usagefor, partnertype, imsi_reject, file_prefix, row_num, tadigcode` <br> From `org_details`: `orgno, orgname` |
| **Filters applied** | `tcl_secs_id > 0` <br> `(proposition_addon != '' AND usagefor != 'Interconnect') OR (usagefor = 'Interconnect' AND upper(partnertype) = 'INGRESS')` <br> `country != ''` <br> `tadigcode != ''` <br> `imsi_reject = 'Y'` <br> `file_prefix != 'SNG'` <br> `filename NOT LIKE '%NonMOVE%'` <br> `row_num = 1` |
| **Aggregations** | `sum(cdr_count) AS cdr_count` <br> `sum( CASE WHEN calltype='Data' THEN bytes_sec_sms/(1024*1024) WHEN calltype IN ('Voice','ICVOICE') THEN bytes_sec_sms/60 ELSE bytes_sec_sms END ) AS cdr_usage` |
| **Output** | Hive view `mnaas.ra_calldate_imsi_reject` (persisted in the Hive metastore). |
| **Side effects** | DDL operation – creates or replaces the view. No data is written to HDFS directly, but the view metadata is stored in the Hive metastore. |
| **Assumptions** | - Hive (or Spark‑SQL) engine is available and configured for the `mnaas` database. <br> - Underlying tables exist with the exact column names/types. <br> - The environment has sufficient permissions to create/replace views. <br> - No partitioning is required for this view (it inherits partitions from the source tables at query time). |

---

## 4. Integration Points

| Connected Component | How it connects |
|---------------------|-----------------|
| **Downstream reject‑handling jobs** (e.g., `mnaas_ra_calldate_billing_reject.hql`, `mnaas_ra_calldate_country_reject.hql`) | These scripts query the `ra_calldate_imsi_reject` view to generate billing‑reject reports or to feed into downstream ETL pipelines. |
| **Reporting dashboards** (BI tools) | The view is exposed to reporting layers (e.g., Tableau, PowerBI) via Hive/Presto connectors to show IMSI‑reject statistics. |
| **Data quality monitoring** | A nightly data‑quality job may run `SELECT COUNT(*) FROM mnaas.ra_calldate_imsi_reject` and raise alerts if the count exceeds thresholds. |
| **Orchestration framework** (Oozie / Airflow) | The script is typically invoked as a Hive action within a workflow that processes a daily batch of CDR files. The workflow ensures the view is refreshed before downstream steps execute. |
| **Configuration / templating** | The script may be stored in a version‑controlled repository and rendered with environment‑specific variables (e.g., database name, schema prefixes) before execution. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – source tables change (column rename, type change) | View creation fails, downstream jobs break | Add a schema‑validation step (e.g., `DESCRIBE` + assert expected columns) before view creation. |
| **Large join/aggregation** – high data volume leads to long Hive jobs or OOM | Job timeout, cluster resource exhaustion | Ensure appropriate Hive execution settings (`hive.exec.reducers.bytes.per.reducer`, `mapreduce.memory.mb`). Consider materializing intermediate tables with partition pruning. |
| **Missing permissions** – user lacks `CREATE VIEW` rights | Script aborts, no view available | Document required Hive role (`CREATE VIEW` on `mnaas`) and enforce via CI pipeline checks. |
| **Incorrect filter logic** – future changes to `imsi_reject` flag semantics | Wrong records included/excluded | Add unit‑test queries against a known test dataset; keep filter logic in a single, well‑commented place. |
| **View staleness** – underlying tables are refreshed but view is not re‑created | Downstream reports use outdated data | Include `CREATE OR REPLACE VIEW` (already used) and schedule the script after source table loads. |

---

## 6. Running & Debugging the Script

1. **Standard execution (CLI)**  
   ```bash
   hive -hiveconf hive.execution.engine=mr -f mediation-ddls/mnaas_ra_calldate_imsi_reject.hql
   ```
   - The `-hiveconf` options may be injected by the orchestration layer (e.g., to set the target database).

2. **Via Oozie / Airflow**  
   - Define a Hive action that points to the script file.  
   - Set the `oozie.use.system.libpath=true` (or equivalent) to ensure Hive libraries are available.

3. **Debug steps**  
   - **Check view creation:** `SHOW CREATE VIEW mnaas.ra_calldate_imsi_reject;`  
   - **Validate data:** `SELECT * FROM mnaas.ra_calldate_imsi_reject LIMIT 10;`  
   - **Explain plan:** `EXPLAIN SELECT * FROM mnaas.ra_calldate_imsi_reject;` to verify join/aggregation strategy.  
   - **Log inspection:** Hive logs are written to `$HIVE_LOG_DIR` or captured by the orchestration engine; look for `FAILED` or `SemanticException`.

4. **Testing in isolation**  
   - Create a temporary view with a subset of the source tables (e.g., using `CREATE TEMPORARY VIEW test_src AS SELECT ... FROM ra_calldate_traffic_table LIMIT 1000;`) and replace the source table names in the script for quick validation.

---

## 7. External Configuration / Environment Variables

| Variable / File | Purpose |
|-----------------|---------|
| `HIVE_HOME`, `HADOOP_CONF_DIR` | Locate Hive binaries and Hadoop configuration for the CLI execution. |
| `HIVE_EXECUTION_ENGINE` (optional) | Determines whether MapReduce or Tez is used; may be set by the orchestration layer. |
| `DATABASE` (if templated) | Some deployments replace the hard‑coded `mnaas` schema with a variable; check the wrapper that renders the script. |
| `LOG_LEVEL` (Hive) | Controls verbosity of Hive logs; useful for debugging. |

If the script is stored in a version‑controlled repository, a **properties** file (e.g., `hive-env.properties`) may define these values for CI pipelines.

---

## 8. Suggested Improvements (TODO)

1. **Add explicit partition pruning** – If `ra_calldate_traffic_table` is partitioned by `callmonth` or `processed_date`, modify the view to include those partitions in the `WHERE` clause (e.g., `WHERE callmonth = '${process_month}'`) to reduce scan size.

2. **Document column lineage** – Add a comment block at the top of the script describing each source column used, its business meaning, and why it is part of the reject criteria. This aids future maintainers and supports data‑governance tools.  

---