**Report Query Configuration – `move-mediation-scripts/config/reportQuery.yml`**  

---

### 1. High‑Level Summary
`reportQuery.yml` is a central, YAML‑style configuration file that maps logical report identifiers to parametrised Hive/Impala SQL statements. Each entry is used by the mediation‑layer scripts (e.g., the nightly Spark jobs, Bash wrappers, and ad‑hoc reporting utilities) to fetch raw CDR, addon, balance‑notification, DPI/Applet, and 5G‑NSA data from the `mnaas` data‑warehouse. The file supplies the only source of query text for the “report” family of jobs, allowing a single point of change for schema evolution or partition‑pruning logic.

---

### 2. Important Keys / Consumers  

| Consumer (script / component) | How it uses `reportQuery.yml` | Typical parameters supplied |
|-------------------------------|------------------------------|-----------------------------|
| `traffic_nodups_summation.py` (Spark job) | Loads the `trafficusages` query to read de‑duplicated CDRs for a date range. | `partition_date` (YYYY‑MM‑DD) and `tcl_secs_id` (integer). |
| `traffic_aggr_adhoc_refresh.sh` (Bash wrapper) | Calls the `addonexpiry`, `lowbalnotification`, `summonusage`, etc., for ad‑hoc aggregation runs. | Date string (used with `substr(?,1,7)`) and a SECS ID. |
| `usage_overage_charge.sh` | Retrieves `trafficusages` and `msisdndailyaggr` to compute over‑usage charges. | Same as above. |
| DPI/Applet reporting jobs (`dpiapplet_report`, `dpiapplet_summary_report`) | Pulls the `dpiapplet_report` and `dpiapplet_summary_report` queries for monthly service‑asset mapping. | Reference month (`?` formatted as `yyyy‑MM‑dd`). |
| 5G‑NSA reporting jobs (`fivegnsa`, `fivegnsa_summary_report`) | Uses `fivegnsa` and `fivegnsa_summary_report` to build 5G‑NSA asset tables. | Reference month (`?`). |
| Asset‑package generation (`asset_pkg`) | Executes the `asset_pkg` query to build the `tcl_asset_pkg_mapping` table. | Reference month (`?`). |

*Note:* The exact loading mechanism is typically a small utility (e.g., a Bash function or a Python helper) that parses the file, splits on `==>` and substitutes the `?` placeholders with runtime values.

---

### 3. Inputs, Outputs, Side Effects & Assumptions  

| Query ID | Input Parameters | Expected Output | Side Effects | Key Assumptions |
|----------|------------------|----------------|--------------|-----------------|
| `addonexpiry` | `date` (YYYY‑MM‑DD), `tcl_secs_id` | Rows of addon expiry records for the month of the supplied date. | None (read‑only). | `mnaas.traffic_details_nodups_addon` partitioned by `partition_date`. |
| `trafficusages` | `partition_date` (YYYY‑MM‑DD), `tcl_secs_id` | Full CDR rows for the day and SECS. | None. | Table `mnaas.traffic_details_raw_daily_with_no_dups` is up‑to‑date and partitioned by `partition_date`. |
| `msisdndailyaggr` | `partition_date`, `tcl_secs_id` | Daily per‑MSISDN aggregation rows. | None. | Source view `mnaas.msisdn_level_daily_usage_aggr` is refreshed before use. |
| `lowbalnotification` | `date`, `tcl_secs_id` | Low‑balance notification rows for the month. | None. | Table `mnaas.traffic_details_nodups_balance` contains `notification_event_month`. |
| `summonusage` | `date`, `tcl_secs_id` | Summation‑level CDR rows (post‑aggregation). | None. | Table `mnaas.traffic_details_raw_daily_with_no_dups_summation` is partitioned by `partition_month`. |
| `dpiapplet_report` | `date` (month) | All DPI applet summary rows for the month. | None. | Table `mnaas.dpi_applet_record_summary` partitioned by `event_month`. |
| `dpiapplet` | `date` (month) | Complex INSERT‑OVERWRITE that populates `tcl_service_asset_mapping_records` for DPI & Applet services. | Writes to Hive table `mnaas.tcl_service_asset_mapping_records`. | All referenced source tables (`vaz_dm_*`, `asset_based_service_status`, etc.) are refreshed for the same month. |
| `dpiapplet_summary_report` | `date` (month) | Inserts aggregated summary rows into `tcl_service_asset_mapping_summary`. | Writes to Hive table `mnaas.tcl_service_asset_mapping_summary`. | Relies on data produced by `dpiapplet`. |
| `fivegnsa` | `date` (month) | Inserts 5G‑NSA service‑asset rows into `tcl_service_asset_mapping_records`. | Write‑only. | Source table `mnaas.nsa_fiveg_product_subscription` contains the month’s data. |
| `fivegnsa_summary_report` | `date` (month) | Inserts 5G‑NSA summary rows into `tcl_service_asset_mapping_summary`. | Write‑only. | Depends on `fivegnsa` output. |
| `asset_pkg` | `date` (month) | Inserts asset‑package mapping rows into `tcl_asset_pkg_mapping`. | Write‑only. | Requires `tcl_service_asset_mapping_records` to be up‑to‑date. |

*All queries assume the Hive metastore is reachable, the `mnaas` database exists, and the user executing the job has read/write permissions on the referenced tables.*

---

### 4. Integration with the Rest of the System  

1. **Configuration Loader** – A shared library (e.g., `config_loader.sh` or `query_loader.py`) reads `reportQuery.yml`, builds a dictionary `QUERY_MAP[<id>] = <SQL>`.  
2. **Orchestration Layer** – The nightly Airflow / Oozie DAGs invoke Bash wrappers (`traffic_aggr_adhoc_refresh.sh`, `usage_overage_charge.sh`, etc.) which:  
   * source environment files (`env.sh`) for DB connection strings,  
   * call the loader to retrieve the required query,  
   * substitute the `?` placeholders with runtime arguments (date, SECS ID),  
   * pipe the final SQL to `beeline`/`spark-sql` or embed it in a Spark DataFrame read.  
3. **Data Flow** –  
   * **Read‑only queries** (`addonexpiry`, `trafficusages`, `msisdndailyaggr`, `lowbalnotification`, `summonusage`, `dpiapplet_report`) feed downstream Spark transformations or Hive INSERT‑OVERWRITE statements.  
   * **Write‑heavy queries** (`dpiapplet`, `dpiapplet_summary_report`, `fivegnsa`, `fivegnsa_summary_report`, `asset_pkg`) populate the reporting tables that are later consumed by BI dashboards, SLA monitors, and downstream billing systems.  
4. **Downstream Consumers** – The tables created by the write‑heavy queries are read by:  
   * BI reporting tools (Tableau, PowerBI) via Hive/Presto,  
   * Billing engines that calculate usage‑based charges,  
   * SLA alerting scripts that monitor asset activation/de‑activation trends.  

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – source tables change (column added/removed) breaking the hard‑coded SELECT lists. | Job failures, silent data loss. | Add unit‑test scripts that validate each query against the current Hive schema (e.g., `spark-sql -e "EXPLAIN <query>"`). Keep a versioned copy of the schema in source control. |
| **Parameter substitution errors** – wrong date format or missing SECS ID leads to empty result sets. | Incorrect reports, downstream billing errors. | Centralise parameter validation in the loader (reject if `!date =~ /^\d{4}-\d{2}-\d{2}$/`). Log the final rendered SQL at DEBUG level. |
| **Long‑running INSERT‑OVERWRITE** – `dpiapplet` and `fivegnsa` can lock target tables for hours. | Pipeline stalls, contention with ad‑hoc queries. | Schedule these jobs during low‑traffic windows, enable Hive ACID/transactional tables if supported, or use `INSERT INTO` with partition overwrite instead of full table overwrite. |
| **Missing partitions** – queries filter on `partition_date=substr(?,1,7)`; if the partition does not exist, Hive may scan the whole table. | Performance degradation, high cluster cost. | Pre‑check partition existence (`SHOW PARTITIONS <table> LIKE '<YYYY-MM>'`) before running the job; abort early with a warning. |
| **Credential leakage** – the loader may embed DB passwords in logs when printing rendered SQL. | Security breach. | Mask credentials in logs, use Kerberos or LDAP authentication, and keep DB connection strings in a secured vault (e.g., HashiCorp Vault). |

---

### 6. Running / Debugging the File  

1. **Typical Operator Flow**  
   ```bash
   # Example: Run the ad‑hoc traffic aggregation for SECS 24048, month 2024‑09
   export SEC_ID=24048
   export RUN_MONTH=2024-09-01
   ./bin/traffic_aggr_adhoc_refresh.sh $RUN_MONTH $SEC_ID
   ```
   The wrapper script will:  
   * source `config/reportQuery.yml` via the loader,  
   * replace `?` with `$RUN_MONTH` and `$SEC_ID`,  
   * execute the resulting Hive SQL with `beeline -u $HIVE_JDBC_URL -e "<SQL>"`.  

2. **Debugging Steps**  
   * **Validate query rendering** – add `set -x` in the Bash wrapper or enable `DEBUG` flag in the Python loader to print the final SQL.  
   * **Check partition existence** – run `hive -e "SHOW PARTITIONS mnaas.traffic_details_raw_daily_with_no_dups LIKE '${RUN_MONTH}%';"` before the job.  
   * **Run a single query manually** – copy the rendered SQL into a Hive console to see row counts or errors.  
   * **Log inspection** – the wrapper writes to `/var/log/move/traffic_aggr_$(date +%Y%m%d).log`; look for “Query execution time” and any “FAILED” messages.  

3. **Developer Unit Test** (Python example)  
   ```python
   from query_loader import load_queries
   q = load_queries()['trafficusages']
   rendered = q.replace('?', "'2024-09-15'").replace('?', '24048')
   assert 'partition_date=?' not in rendered
   # Optionally run via Spark:
   df = spark.sql(rendered)
   assert df.count() > 0
   ```

---

### 7. External Configurations & Environment Variables  

| Variable / File | Purpose |
|-----------------|---------|
| `HIVE_JDBC_URL` (env) | JDBC connection string used by `beeline`/Spark to reach the Hive metastore. |
| `SEC_ID` / `RUN_MONTH` (passed as script arguments) | Runtime parameters that replace the `?` placeholders. |
| `move-mediation-scripts/config/property_compare.sh` | May contain helper functions for comparing query results; referenced by some wrappers. |
| `move-mediation-scripts/config/query.yml` | Another query‑definition file; some scripts import both `query.yml` and `reportQuery.yml` depending on the report type. |
| `move-mediation-scripts/config/MNAAS_Traffic_Adhoc_Aggr.sh` | Holds default values (e.g., default SECS IDs, date formats) used by ad‑hoc aggregation scripts. |
| `move-mediation-scripts/config/query_xm.yml` | Contains XML‑specific queries; not directly used by `reportQuery.yml` but part of the same family. |

---

### 8. Suggested Improvements (TODO)

1. **Parameter‑Placeholder Standardisation** – Replace the ambiguous `?` with named placeholders (e.g., `:partition_date`, `:tcl_secs_id`) and use a templating engine (Jinja2) to improve readability and reduce substitution bugs.  
2. **Schema‑Version Tagging** – Add a top‑level `schema_version: 2025-01` field and a small validation script that aborts execution if the version in the file does not match the version expected by the consuming job. This will make coordinated upgrades safer.  

--- 

*End of documentation.*