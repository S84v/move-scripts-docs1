# Summary
`reportQuery20250630.yml` is a YAML‑based query repository used by the telecom “move” mediation pipeline. Each entry maps a logical name to a Hive/Impala SQL statement that extracts, aggregates, or transforms raw traffic, addon, and DPI‑applet data for downstream reporting, statistics computation, and HDFS staging. The file is loaded at runtime by Bash or Java jobs (e.g., *Move_Table_Compute_Stats*, *Org_details_Sqoop*) to obtain the exact SQL text to execute against the `mnaas` data warehouse.

# Key Components
- **addonexpiry** – Retrieves addon usage rows for a given month and `tcl_secs_id`.
- **trafficusages** – Pulls raw daily traffic details filtered by `partition_date` and `tcl_secs_id`; includes extensive column list for usage metrics.
- **msisdndailyaggr** – Selects daily per‑MSISDN aggregated usage for a specific month and `tcl_secs_id`.
- **lowbalnotification** – Extracts low‑balance notification events for a month and `tcl_secs_id`.
- **summonusage** – Similar to `trafficusages` but targets the `*_summation` table; used for consolidated usage reporting.
- **dpiapplet_summary_report** – Complex CTE‑based query that builds a monthly DPI/Applet provisioning summary, inserting results into `monthly_dpi_applet_summary`.
- **dpiapplet_report** – Simple filter on `dpi_applet_record_summary` for a given month.
- **dpiapplet** – Multi‑CTE ingestion pipeline that joins inventory, QoS, and provisioning data, then overwrites `tcl_service_asset_mapping_records` with a unified asset‑service view.

# Data Flow
| Stage | Description |
|-------|-------------|
| **Input** | Parameter placeholders (`?`) supplied at job launch: `partition_date`, `tcl_secs_id`, and a reference date string (e.g., `yyyy-MM-dd`). |
| **Processing** | Queries are sent to Impala/Hive via JDBC or `impala-shell`. No DML side‑effects except the two `INSERT OVERWRITE` statements (`dpiapplet_summary_report`, `dpiapplet`). |
| **Output** | Result sets streamed to the invoking job (Sqoop, Spark, or custom Bash). For the two `INSERT` queries, data is persisted to Hive tables (`monthly_dpi_applet_summary`, `tcl_service_asset_mapping_records`). |
| **Side Effects** | Table overwrites may trigger downstream partition refreshes and affect downstream analytics if run concurrently. |
| **External Services** | - Impala service (`$IMPALAD_HOST:$IMPALAD_JDBC_PORT`) <br> - Hive metastore (for table metadata) <br> - HDFS (target tables reside on HDFS) |

# Integrations
- **Bash jobs** (`move-mediation-scripts/config/*.properties`) source this YAML via `yq`/`awk` to inject the SQL into `impala-shell` commands.
- **Java/Sqoop jobs** (`QueryResult.java`) may reference the same logical names when constructing free‑form queries for import.
- **Move_Table_Compute_Stats** reads the query strings to compute table statistics after data load.
- **Org_details_Sqoop** uses similar pattern for Oracle‑to‑HDFS imports; the YAML provides a consistent naming convention for reporting queries.

# Operational Risks
- **Parameter Mismatch** – Incorrect date format or missing `tcl_secs_id` leads to empty result sets or full‑table scans. *Mitigation*: Validate parameters before query execution; enforce ISO‑8601 date format.
- **Table Schema Drift** – Adding/removing columns in source tables without updating the YAML will cause query failures. *Mitigation*: Include schema‑validation step in CI pipeline.
- **Concurrent Overwrites** – `INSERT OVERWRITE` on shared tables can cause race conditions. *Mitigation*: Serialize jobs per partition month or use atomic staging tables.
- **SQL Injection** – Direct substitution of user‑supplied values into `?` placeholders without prepared statements. *Mitigation*: Use parameterized JDBC calls; never concatenate raw strings.

# Usage
```bash
# Example Bash snippet to fetch a query and run it via impala-shell
QUERY_NAME="trafficusages"
DATE_PARAM="2025-06-30"
SECS_ID="12345"

SQL=$(yq e ".${QUERY_NAME}" move-mediation-scripts/config/reportQuery20250630.yml | sed 's/^==>//')
impala-shell -i $IMPALAD_HOST -q "$SQL" --quiet -B -d ',' \
  -var partition_date=$DATE_PARAM -var tcl_secs_id=$SECS_ID > /tmp/${QUERY_NAME}.csv
```
For Java/Sqoop:
```java
String sql = yamlMap.get("dpiapplet_report").replace("?", dateParam);
SqoopImportJob job = new SqoopImportJob(sql, ...);
job.run();
```

# Configuration
- **Environment Variables** (imported via `MNAAS_CommonProperties.properties`): <br>
  `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`, `MNAASLocalLogPath`, `HDFS_STAGING_DIR`, etc.
- **Config Files**: <br>
  `move-mediation-scripts/config/reportQuery20250630.yml` (this file) <br>
  `move-mediation-scripts/config/MNAAS_CommonProperties.properties` (shared env vars)
- **External Parameters**: `partition_date`, `tcl_secs_id`, optional reference date for CTEs.

# Improvements
1. **Parameterization Layer** – Replace raw `?`