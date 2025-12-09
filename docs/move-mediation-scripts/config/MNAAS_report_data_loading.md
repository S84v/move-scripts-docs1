# Summary
`MNAAS_report_data_loading.properties` defines all runtime constants required by the **MNAAS_report_data_loading** batch job. The job extracts, transforms, and loads a set of static report files (pricing, mapping, etc.) into Hive staging tables, then inserts the data into final Hive tables. The properties file centralises paths, process‑status handling, log locations, source/backup directories, and the mapping of file names → Hive tables → temporary tables → INSERT SQL statements used by the shell/Hadoop/Sqoop orchestration scripts.

# Key Components
- **MNAASConfPath** – base directory for shared configuration files.  
- **MNAASLocalLogPath** – base directory for cron‑log files.  
- **MNAAS_report_data_loading_ProcessStatusFileName** – file used for idempotency/monitoring; written by the job to indicate success/failure.  
- **MNAAS_report_data_loading_log_file_path** – absolute path of the job’s log file.  
- **MNAAS_report_data_loading_source_data_location** – location of the incoming report files (`$customer_dir/Reports`).  
- **MNAAS_report_data_loading_backup_data_location** – location where processed files are archived (`$BackupDir/Reports`).  
- **MNAAS_report_data_loading_scriptName** – executable shell script (`MNAAS_report_data_loading.sh`).  
- **dataBase_name** – Hive database name (`mnaas`).  
- **declare -A file_format_and_table** – associative array mapping raw file names to target Hive tables.  
- **declare -A table_tmpTable_mapping** – associative array mapping target Hive tables to staging temporary tables.  
- **declare -A insert_query** – associative array mapping target Hive tables to the HiveQL `SELECT` clause used in the `INSERT OVERWRITE` statement.

# Data Flow
| Stage | Input | Processing | Output | Side‑effects |
|-------|-------|------------|--------|--------------|
| 1. Source ingestion | Files in `$customer_dir/Reports` (e.g., `customer_pricing.txt`) | Shell script reads each file, loads into HDFS staging area, then uses Hive `LOAD DATA` into `${table}_tmp`. | Temporary Hive tables (`*_tmp`). | Updates process‑status file to *running*. |
| 2. Transformation | Temporary tables | HiveQL `INSERT OVERWRITE ${dataBase_name}.${table}` using the `insert_query` SELECT clause. | Final Hive tables (`customer_pricing`, `dim_date`, …). | Writes success/failure flag to process‑status file; moves original files to `$BackupDir/Reports`. |
| 3. Logging | N/A | All stdout/stderr redirected to `$MNAASLocalLogPath/MNAAS_report_data_loading.log`. | Log file. | None beyond log rotation. |

External services: Hadoop HDFS, Hive Metastore, optional Sqoop for downstream export.

# Integrations
- **MNAAS_CommonProperties.properties** – sourced first to obtain shared variables (`MNAASConfPath`, `MNAASLocalLogPath`, etc.).  
- **MNAAS_report_data_loading.sh** – the orchestrating shell script that sources this properties file and executes the data‑load workflow.  
- **Other batch jobs** (e.g., `MNAAS_RawFileCount_Checks`, `MNAAS_redONE_MSISDNs_Check`) share the same process‑status directory and log conventions, enabling a unified monitoring dashboard.  
- **Hive** – tables defined in `file_format_and_table` must exist in the `mnaas` database prior to execution.  
- **Backup subsystem** – `$BackupDir` is managed by a separate archival script that purges files older than a configurable retention period.

# Operational Risks
- **Missing source files** – job will fail if any expected `.txt` file is absent. *Mitigation*: pre‑flight check that all keys in `file_format_and_table` exist; alert on missing files.  
- **Schema drift** – changes to source file column order break the `insert_query` SELECT list. *Mitigation*: version the property file with a schema descriptor; run automated validation against a sample file.  
- **Hive table lock contention** – concurrent jobs inserting into the same tables may cause lock timeouts. *Mitigation*: enforce job sequencing via cron or Airflow DAG.  
- **Insufficient HDFS storage** – loading large report files may exceed quota. *Mitigation*: monitor HDFS usage; fail fast with clear error code.  
- **Process‑status file stale** – if the job crashes without cleanup, the status file may remain in a *running* state. *Mitigation*: include a watchdog that resets status after a timeout.

# Usage
```bash
# Load environment
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_report_data_loading.properties

# Execute the job (normally via cron)
bash $MNAAS_report_data_loading_scriptName

# Debug – run with trace and force log to console
bash -x $MNAAS_report_data_loading_scriptName 2>&1 | tee /tmp/debug.log
```

# Configuration
- **Environment variables** required before sourcing: `customer_dir`, `BackupDir`.  
- **Referenced config files**: `MNAAS_CommonProperties.properties`.  
- **Properties defined in this file**: all variables listed under *Key Components*.  
- **Associative arrays** (`file_format_and_table`, `table_tmpTable_mapping`, `insert_query`) must be kept in sync; any addition of a new report file requires entries in all three arrays.

# Improvements
1. **Schema‑driven definition** – replace hard‑coded `insert_query` strings with a metadata table (e.g., JSON/YAML) that describes column mappings; generate HiveQL at runtime to reduce maintenance overhead.  
2. **Automated validation step** – add a pre‑execution stage that validates file existence, column count, and data types against the metadata, aborting early with a detailed error report.