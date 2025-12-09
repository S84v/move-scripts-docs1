# Summary
`MOVEDAO.properties` is a Java‑style properties file that centralises all Hive/SQL statements required by the KLMReport component of the Move‑Mediation‑Ingest pipeline. It supplies DDL/DML for extracting KLM traffic data, populating staging tables, aggregating usage, and managing month‑level billing metadata for the offer **MOVE SIM AVKLM‑4Z‑EUS‑$500**.

# Key Components
- **klm.summary** – Simple SELECT for reporting traffic summary.  
- **klm.allin** – CTE‑based query that filters raw daily traffic, resolves missing offer description, limits to a specific `zoneid`, and aggregates usage per day/country.  
- **cdr.subset.truncate / cdr.subset.insert** – Truncate and populate `billing_traffic_filtered_cdr` with filtered Data‑type CDRs for the target offer and zone.  
- **date.wise.cdr.fetch** – Retrieves per‑day CDR counts and usage totals from the filtered CDR table.  
- **cdr.fetch** – Pulls individual CDR rows for a given `calldate`.  
- **in.usage.fetch / out.usage.fetch** – Compute cumulative inbound/outbound usage up to or from a given `cdrnum`.  
- **truncate.month / insert.month** – Reset and load month‑boundary metadata in `month_reports`.  
- **truncate.traffic / insert.traffic** – Refresh the `traffic_details_billing` table from the view `v_traffic_details_billing_klm`.  

# Data Flow
| Step | Input(s) | Operation | Output / Side‑Effect |
|------|----------|-----------|----------------------|
| 1 | Hive tables: `traffic_klm_summ_report`, `traffic_details_raw_daily`, `org_details`, `month_reports`, `mcc_cntry_mapping` | `klm.summary` / `klm.allin` | Result sets consumed by downstream Java reporting logic. |
| 2 | Parameter: `zoneid` (runtime) | `cdr.subset.truncate` → `cdr.subset.insert` | Populated `billing_traffic_filtered_cdr` (filtered CDRs). |
| 3 | No param | `date.wise.cdr.fetch` | Daily aggregates stored in memory / passed to report generator. |
| 4 | Param: `calldate` | `cdr.fetch` | Detailed CDR rows for the date. |
| 5 | Params: `calldate` (via `cdrnum` bounds) | `in.usage.fetch` / `out.usage.fetch` | Cumulative usage metrics for inbound/outbound windows. |
| 6 | Params: month metadata (6 values) | `truncate.month` → `insert.month` | Updated `month_reports` table (billing period definition). |
| 7 | No param | `truncate.traffic` → `insert.traffic` | Refresh of `traffic_details_billing` from materialised view. |
| **External Services** | Hive Metastore (`mnaas` database) | All statements executed via Hive/Impala JDBC driver. |
| **Side Effects** | Table truncates, inserts | Permanent modification of staging tables used by downstream billing jobs. |

# Integrations
- **Java DAO layer** (`MOVEDAO` class) loads this file, substitutes `?` placeholders with values supplied by the orchestration engine (e.g., Airflow, Oozie).  
- **Shell scripts** (`MNAAS_ShellScript.properties`) provide Hive/Impala connection strings and credentials used by the Java process.  
- **Email notification module** reads execution status (success/failure) from the same orchestration context and sends alerts.  
- **Reporting UI** consumes the result sets produced by `klm.summary` and `klm.allin` for dashboard visualisation.  

# Operational Risks
- **Table Truncation** – `truncate.*` statements erase data; accidental mis‑parameterisation could lead to data loss. *Mitigation*: enforce pre‑run validation and retain nightly backups.  
- **Hard‑coded TCL_SEC_ID (36378)** – Limits reuse; any change in offer ID requires code/config update. *Mitigation*: externalise to a property or environment variable.  
- **Missing Parameter Validation** – `zoneid` and `cdrnum` are supplied at runtime; invalid values cause empty result sets or runtime errors. *Mitigation*: add schema validation in the orchestration layer.  
- **Hive Compatibility** – Queries rely on window functions and CTEs; older Hive versions may reject them. *Mitigation*: enforce Hive ≥ 2.1 in the deployment environment.  

# Usage
```bash
# Example Java invocation (pseudo)
java -Dlogfile.name=klm.log \
     -cp myapp.jar com.move.mediation.klm.MOVEDAO \
     --zoneid 12 \
     --start_date 2024-01-01 \
     --end_date 2024-01-31
```
- The DAO reads `MOVEDAO.properties`, substitutes `?` with supplied args, executes statements sequentially, and logs progress via Log4j.  
- For debugging, set `log4j.rootLogger=DEBUG` and inspect the generated Hive queries in the log output.  

# Configuration
- **Properties Files**: `MOVEDAO.properties` (this file), `MNAAS_ShellScript.properties` (connection strings, credentials).  
- **Environment Variables**: `HIVE_JDBC_URL`, `HIVE_USER`, `HIVE_PASSWORD` (populated from `MNAAS_ShellScript.properties`).  
- **System Properties**: `logfile.name` (log file path).  

# Improvements
1. **Parameterise TCL_SEC_ID** – Replace hard‑coded `36378` with a configurable property (`tcl_secs_id`) to support multiple offers without code change.  
2. **Add Idempotent Load Logic** – Replace `truncate`/`insert` pattern with `MERGE` or `INSERT OVERWRITE` to reduce risk of accidental data loss and improve recoverability.