**File:** `mediation-ddls\mnaas_ra_calldate_tadig_reject.hql`  

---

## 1. High‑Level Summary
This HiveQL script creates (or replaces) the view `mnaas.ra_calldate_tadig_reject`. The view isolates call‑detail‑record (CDR) rows that have a missing **TADIG code** (the identifier used for inter‑operator routing). It aggregates the number of CDRs and the usage amount per file, month, organization, proposition/add‑on, and country, and tags each row with the rejection reason *“Tadig Code is missing”*. The view is later consumed by downstream rejection‑handling jobs (e.g., reporting, error‑file generation, or re‑processing pipelines).

---

## 2. Core Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `ra_calldate_tadig_reject` | Hive **VIEW** (in schema `mnaas`) | Presents a summarized, filtered set of traffic records where `tadigcode = ''`. Provides columns needed for rejection reporting and downstream handling. |
| `mnaas.ra_calldate_traffic_table` | Hive **TABLE** | Source CDR data for the mediation day. Contains raw fields such as `filename`, `processed_date`, `callmonth`, `tcl_secs_id`, `orgname`, `proposition_addon`, `country`, `usage_type`, `calltype`, `bytes_sec_sms`, `tadigcode`, `file_prefix`, `row_num`, etc. |
| `mnaas.org_details` | Hive **TABLE** | Lookup table mapping `orgno` → `orgname` (and possibly other org metadata). Joined to enrich the view with organization names. |

*No procedural code (functions, classes) is present; the script is a DDL statement.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - `mnaas.ra_calldate_traffic_table` (must contain the columns referenced in the SELECT).<br>- `mnaas.org_details` (must contain `orgno` and `orgname`). |
| **Outputs** | - Hive view `mnaas.ra_calldate_tadig_reject`. The view does **not** materialize data; it is a logical definition that will be evaluated at query time. |
| **Side‑effects** | - Overwrites the view if it already exists (implicit `CREATE OR REPLACE VIEW` semantics in many Hive versions). No data is written to disk beyond the view metadata. |
| **Assumptions** | - Hive/Impala engine is configured with the `mnaas` database in the current session.<br>- The source tables are refreshed/partitioned daily by upstream ingestion jobs.<br>- `tadigcode` is stored as an empty string (`''`) when missing (not NULL).<br>- `file_prefix` values and the pattern `%NonMOVE%` are used consistently across all mediation scripts. |
| **External Services / Config** | - Hive metastore connection (usually via `hive-site.xml` or environment variables `HIVE_CONF_DIR`).<br>- May rely on Kerberos tickets or LDAP credentials supplied by the execution environment (e.g., Airflow, Oozie, or a custom scheduler). |

---

## 4. Integration Points (How This File Connects to the Rest of the System)

| Connected Component | Relationship |
|---------------------|--------------|
| **Other “*_reject” view scripts** (e.g., `mnaas_ra_calldate_imsi_reject.hql`, `mnaas_ra_calldate_proposition_reject.hql`) | All reject‑type views share the same source tables and filtering logic; downstream jobs typically UNION all reject views to produce a master rejection report. |
| **Rejection Reporting Job** (e.g., a nightly Spark/MapReduce job) | Consumes `ra_calldate_tadig_reject` together with other reject views to generate CSV/Parquet files sent to the OSS/BSS or to an SFTP drop zone. |
| **Error‑File Generation** | The view feeds a process that creates per‑file error logs for the originating system (e.g., partner network element). |
| **Data Quality Dashboard** | Queries the view to display counts of missing TADIG codes per day, per partner, etc. |
| **Orchestration Layer** (Airflow, Oozie, Control-M) | The script is executed as a Hive task step, usually after the `ra_calldate_traffic_table` load step and before the “reject aggregation” step. |
| **Configuration Files** | May reference a central `hive-variables.conf` that defines the target database (`mnaas`) and any runtime parameters (e.g., `hive.exec.dynamic.partition`). The script itself does not import variables, but the execution wrapper often injects them. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **View becomes stale** if source tables are re‑partitioned or schema‑changed (e.g., column renamed). | Queries return errors or wrong counts. | Include a CI validation step that parses the view definition against the current schema (e.g., using `hive -e "DESCRIBE FORMATTED …"`). |
| **Performance degradation** due to full table scans on large daily traffic tables. | Long query runtimes, possible OOM in downstream jobs. | Ensure `ra_calldate_traffic_table` is partitioned by `processed_date` or `callmonth`; add `WHERE processed_date = '${run_date}'` in the view definition if feasible. |
| **Incorrect rejection logic** if `tadigcode` can be NULL instead of empty string. | Missing records that should be rejected. | Add `OR tadigcode IS NULL` to the filter, and verify with data profiling. |
| **Missing join key** (`tcl_secs_id` not present in `org_details`). | Rows drop silently, reducing rejection counts. | Use a LEFT OUTER JOIN (already present) and monitor `orgname` null percentages; add a data‑quality alert if > 5 % null. |
| **Accidental view overwrite** during a redeploy of a different reject view with the same name. | Loss of expected view definition. | Adopt a naming convention that includes the rejection reason (e.g., `ra_calldate_reject_tadig_missing`). Use version‑controlled DDL scripts and a deployment checklist. |

---

## 6. Running / Debugging the Script  

1. **Typical Execution (operator)**  
   ```bash
   # From the orchestration host
   hive -f mediation-ddls/mnaas_ra_calldate_tadig_reject.hql
   ```
   *The Hive CLI (or Beeline) reads the file and registers/updates the view.*

2. **Verification**  
   ```sql
   USE mnaas;
   SHOW CREATE VIEW ra_calldate_tadig_reject;
   SELECT COUNT(*) AS total_rejects FROM ra_calldate_tadig_reject;
   SELECT * FROM ra_calldate_tadig_reject LIMIT 10;
   ```
   *Check that the row count matches expectations and that the `reason` column is populated.*

3. **Debugging Tips**  
   - **Syntax errors**: Run `hive -e "EXPLAIN SELECT …"` on the inner SELECT to isolate parsing problems.  
   - **Missing columns**: Query `DESCRIBE ra_calldate_traffic_table;` and compare with the script.  
   - **Performance**: Run `EXPLAIN EXTENDED SELECT … FROM ra_calldate_tadig_reject;` to view the execution plan; look for full scans.  
   - **Log inspection**: Hive logs are typically under `/tmp/hive.log` or the YARN application logs; search for `Error` or `Failed to create view`.  

---

## 7. External Configuration / Environment Variables  

| Variable / File | Purpose |
|-----------------|---------|
| `HIVE_CONF_DIR` (env) | Points to Hive configuration (metastore URI, authentication). |
| `HADOOP_USER_NAME` (env) | Determines the Hadoop user under which the Hive session runs. |
| `hive.exec.dynamic.partition` (property) | If the view is later materialized, this controls partition creation. |
| `mnaas` database name | Hard‑coded in the script; could be externalized to a variable in a wrapper script for multi‑tenant deployments. |
| `run_date` (parameter passed by scheduler) | Not used directly in this view, but upstream jobs may filter the traffic table by date; ensure the view is evaluated after the appropriate date partition is loaded. |

*The script itself does not reference any external files, but the execution environment must provide the standard Hive configuration.*

---

## 8. Suggested Improvements (TODO)

1. **Add NULL handling for `tadigcode`**  
   ```sql
   AND (tadigcode = '' OR tadigcode IS NULL)
   ```
   This guards against schema variations where missing values are stored as `NULL`.

2. **Introduce a partition‑pruned filter** (if the traffic table is partitioned by `processed_date`):  
   ```sql
   AND processed_date = '${run_date}'
   ```
   Pass `${run_date}` from the orchestration layer to limit the scan to the current batch, dramatically improving performance.

*Both changes are backward‑compatible and can be tested in a staging environment before promotion.*