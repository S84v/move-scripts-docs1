# Summary
`MNAAS_report_data_loading_test.properties` is a Bash‑sourced configuration file that supplies runtime constants for the **MNAAS_report_data_loading** ETL job. It defines paths (configuration, logs, source, backup), process‑status handling, script name, target database, and three associative arrays that map input file names → Hive tables, temporary staging tables, and INSERT‑SELECT statements used to load static report data into Hive.

# Key Components
- **Sourced common properties** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Path variables**
  - `MNAASConfPath` – base config directory
  - `MNAASLocalLogPath` – cron‑log directory
  - `MNAAS_report_data_loading_ProcessStatusFileName` – status flag file
  - `MNAAS_report_data_loading_log_file_path` – job log file
  - `MNAAS_report_data_loading_source_data_location` – HDFS source directory (`$customer_dir/Reports`)
  - `MNAAS_report_data_loading_backup_data_location` – HDFS backup directory (`$BackupDir/Reports`)
  - `MNAAS_report_data_loading_scriptName` – executable shell script
  - `dataBase_name` – Hive database (`mnaas`)
- **Associative array `file_format_and_table`**
  - Key: source file name (e.g., `customer_pricing.txt`)
  - Value: target Hive table (e.g., `customer_pricing`)
- **Associative array `table_tmpTable_mapping`**
  - Key: target Hive table
  - Value: temporary staging table name (e.g., `customer_pricing_tmp`)
- **Associative array `insert_query`**
  - Key: target Hive table
  - Value: SELECT clause used in `INSERT INTO target_table SELECT … FROM tmp_table`

# Data Flow
| Stage | Input | Transformation | Output | Side Effects |
|-------|-------|----------------|--------|--------------|
| 1. Source ingestion | Files in `$MNAAS_report_data_loading_source_data_location` | None (raw files) | Files copied to `$MNAAS_report_data_loading_backup_data_location` for backup | Backup creation |
| 2. Load to Hive staging | Each file per `file_format_and_table` mapping | `LOAD DATA INPATH` into temporary table (`table_tmpTable_mapping`) | Temporary Hive tables populated | HDFS read/write |
| 3. Insert into final tables | Temporary tables | `INSERT INTO <target_table> SELECT …` using `insert_query` | Final Hive tables in `dataBase_name` updated | Hive transaction logs |
| 4. Process status | N/A | Write success/failure flag to `MNAAS_report_data_loading_ProcessStatusFileName` | Status file reflects job outcome | Used by downstream orchestrators |
| 5. Logging | N/A | Write stdout/stderr to `MNAAS_report_data_loading_log_file_path` | Log file for audit/debug | Log rotation external to script |

External services: Hadoop HDFS, Hive Metastore, optional Sqoop (if used for other jobs), cron scheduler.

# Integrations
- **MNAAS_report_data_loading.sh** – main driver script that sources this properties file.
- **MNAAS_CommonProperties.properties** – provides shared environment variables (`$customer_dir`, `$BackupDir`, etc.).
- **Hive** – target data warehouse; tables referenced in mappings must exist or be created prior to execution.
- **Cron** – schedules the job; reads the process‑status file to avoid overlapping runs.
- **HDFS** – source and backup directories are HDFS paths; the script uses `hdfs dfs -cp`/`-mv`.

# Operational Risks
- **Missing or malformed source files** – job will fail; mitigate with pre‑run validation of file existence.
- **Schema drift between source files and INSERT queries** – leads to data truncation or load errors; mitigate by version‑controlled schema definitions and unit tests.
- **Backup path misconfiguration** – data loss risk; mitigate by verifying `$BackupDir` resolves correctly and has sufficient quota.
- **Concurrent executions** – race condition on status file; mitigate by atomic lock file or using a scheduler that enforces single instance.
- **Hard‑coded SELECT clauses** – difficult to maintain; mitigate by externalizing queries to separate SQL files.

# Usage
```bash
# Load configuration into current shell
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source /app/hadoop_users/MNAAS/MNAAS_Configuration_Files/MNAAS_report_data_loading_test.properties

# Execute the ETL driver
bash $MNAAS_report_data_loading_scriptName
```
For debugging, echo key variables after sourcing:
```bash
echo "Source dir: $MNAAS_report_data_loading_source_data_location"
declare -p file_format_and_table
```

# Configuration
- **Referenced files**
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
  - `/app/hadoop_users/MNAAS/MNAAS_Configuration_Files` (directory containing this file)
- **Environment variables required**
  - `customer_dir` – base HDFS directory for customer data
  - `BackupDir` – base HDFS directory for backups
- **Editable parameters**
  - Path variables, database name, associative array entries, INSERT SELECT strings.

# Improvements
1. **Externalize mappings** – store `file_format_and_table`, `table_tmpTable_mapping`, and `insert_query` in JSON/YAML and load at runtime to simplify updates and enable validation scripts.
2. **Add schema validation step** – implement a pre‑load check that parses each source file against an expected Avro/CSV schema and aborts with a clear error if mismatched.