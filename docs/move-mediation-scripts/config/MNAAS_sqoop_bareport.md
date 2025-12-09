# Summary
`MNAAS_sqoop_bareport.properties` is a Bash‑sourced configuration fragment that supplies runtime constants for the **Bare‑Report Sqoop** step of the MNAAS daily‑processing pipeline. It defines script metadata, status‑file locations, log directories, HDFS staging paths, source‑database table names, and the SQL statements used by `MNAAS_Sqoop_Bareport.sh` to import raw billing‑report data into staging Hive/Impala tables.

# Key Components
- **Source statement** – imports shared constants from `MNAAS_CommonProperties.properties`.  
- **SCRIPT_NAME** – identifier for logging and monitoring.  
- **STATUS_FILE** – path to a file that records step success/failure.  
- **LOG_DIR / LOG_FILE** – locations for stdout/stderr capture.  
- **HDFS_STAGING_DIR** – HDFS directory where Sqoop lands raw files before Hive load.  
- **DB_TABLE_RAW** – fully‑qualified source database table (e.g., `BAREPORT_RAW`).  
- **HIVE_TABLE_STAGING** – target Hive/Impala table for the staging layer.  
- **SQOOP_IMPORT_QUERY** – custom SELECT used by Sqoop `--query` option.  
- **SQOOP_OPTIONS** – common Sqoop flags (e.g., `--connect`, `--username`, `--password-file`, `--target-dir`, `--as-avrodatafile`).  

# Data Flow
1. **Input** – JDBC connection to the operational billing database; `SQOOP_IMPORT_QUERY` selects raw report rows.  
2. **Sqoop** – pulls data into `HDFS_STAGING_DIR` as Avro/Parquet files.  
3. **Hive/Impala** – external table points to staging files; subsequent ETL scripts load into final reporting tables.  
4. **Outputs** – status file (`STATUS_FILE`), log file (`LOG_FILE`), and HDFS files in `HDFS_STAGING_DIR`.  
5. **Side Effects** – creation of HDFS directories, possible temporary Kerberos tickets, and update of monitoring dashboards via status file.

# Integrations
- **MNAAS_CommonProperties.properties** – provides base paths, DB credentials, and environment flags.  
- **MNAAS_Sqoop_Bareport.sh** – consumes this property file to construct the Sqoop command line.  
- **Downstream Hive scripts** – read from `HIVE_TABLE_STAGING` to populate final reporting tables.  
- **Monitoring/Orchestration** – status file is polled by the daily‑processing scheduler (e.g., Oozie/Airflow).  

# Operational Risks
- **Credential leakage** – DB passwords stored in plain‑text files; mitigate by using Hadoop credential provider or keytab.  
- **Schema drift** – source table changes break `SQOOP_IMPORT_QUERY`; add schema validation step.  
- **HDFS space exhaustion** – staging directory not cleaned; schedule periodic cleanup or use `--delete-target-dir`.  
- **Network latency / timeouts** – large imports may exceed default Sqoop timeout; tune `--fetch-size` and JDBC timeout.  

# Usage
```bash
# Source the configuration
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_sqoop_bareport.properties

# Run the Sqoop import (debug mode)
set -x
${SQOOP_CMD} import \
  ${SQOOP_OPTIONS} \
  --query "${SQOOP_IMPORT_QUERY} AND \$CONDITIONS" \
  --target-dir "${HDFS_STAGING_DIR}" \
  --split-by "report_id" \
  --num-mappers 4
set +x
```
To debug, set `setparameter='set -x'` in the property file and re‑run the wrapper script `MNAAS_Sqoop_Bareport.sh`.

# Configuration
- **Environment Variables**  
  - `HADOOP_CONF_DIR`, `JAVA_HOME`, `HIVE_CONF_DIR` (inherited from common properties).  
- **Referenced Config Files**  
  - `MNAAS_CommonProperties.properties` (global paths, DB connection strings).  
- **Local Variables Defined in This File**  
  - `SCRIPT_NAME`, `STATUS_FILE`, `LOG_DIR`, `LOG_FILE`, `HDFS_STAGING_DIR`, `DB_TABLE_RAW`, `HIVE_TABLE_STAGING`, `SQOOP_IMPORT_QUERY`, `SQOOP_OPTIONS`.  

# Improvements
1. **Secure Credential Management** – replace inline DB passwords with Hadoop Credential Provider or Kerberos keytab integration.  
2. **Idempotent Staging Cleanup** – add a pre‑run step that removes stale files in `HDFS_STAGING_DIR` and archives previous runs to a retention bucket.