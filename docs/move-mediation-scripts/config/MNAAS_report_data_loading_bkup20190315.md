# Summary
Defines runtime constants for the **MNAAS_report_data_loading** batch job. Provides paths, process‑status file, log location, source/backup directories, script name, database name, and three associative arrays that map input file names → Hive tables, temporary tables, and INSERT‑SELECT statements used to load static report data into Hive.

# Key Components
- **MNAASConfPath** – base directory for configuration files.  
- **MNAASLocalLogPath** – base directory for cron logs.  
- **MNAAS_report_data_loading_ProcessStatusFileName** – full path to the process‑status file (idempotency/monitoring).  
- **MNAAS_report_data_loading_log_file_path** – full path to the job log file.  
- **MNAAS_report_data_loading_source_data_location** – source directory for report files (`$customer_dir/Reports`).  
- **MNAAS_report_data_loading_backup_data_location** – backup directory for processed files (`$BackupDir/Reports`).  
- **MNAAS_report_data_loading_scriptName** – name of the executable shell script (`MNAAS_report_data_loading.sh`).  
- **dataBase_name** – Hive database name (`mnaas`).  
- **declare -A file_format_and_table** – associative array mapping each source file name to its target Hive table.  
- **declare -A table_tmpTable_mapping** – associative array mapping each target Hive table to a staging temporary table.  
- **declare -A insert_query** – associative array mapping each target Hive table to the SELECT clause used in the final INSERT‑SELECT statement.

# Data Flow
1. **Input** – Flat‑file reports located in `$customer_dir/Reports`.  
2. **Processing** – Shell script reads the associative arrays:  
   - Determines target Hive table (`file_format_and_table`).  
   - Loads file into corresponding temporary Hive table (`table_tmpTable_mapping`).  
   - Executes `INSERT INTO ${dataBase_name}.${target_table} ${insert_query[$target_table]}`.  
3. **Output** – Populated Hive tables in database `mnaas`.  
4. **Side Effects** –  
   - Source files are copied/moved to `$BackupDir/Reports`.  
   - Process‑status file is updated to reflect success/failure.  
   - Log entries written to `$MNAASLocalLogPath/MNAAS_report_data_loading.log`.  

# Integrations
- **MNAAS_CommonProperties.properties** – sourced at the top; provides common variables (`$customer_dir`, `$BackupDir`, etc.).  
- **Hive** – used for staging temporary tables and final inserts.  
- **Cron** – job scheduled via `$MNAASLocalLogPath` logs.  
- **Other ETL scripts** – may depend on the same Hive tables as downstream consumers (e.g., reporting, analytics pipelines).  

# Operational Risks
- **Schema drift** – changes in source file columns break the hard‑coded SELECT statements. *Mitigation*: version control of `insert_query` and validation step before load.  
- **Missing source files** – job aborts if any expected file is absent. *Mitigation*: pre‑flight existence check and alerting.  
- **Process‑status file corruption** – could cause false‑positive idempotency. *Mitigation*: atomic write (temp file → rename).  
- **Insufficient Hive permissions** – load fails silently. *Mitigation*: verify Hive user rights during deployment.  

# Usage
```bash
# Source the properties file
. /path/to/MNAAS_report_data_loading_bkup20190315.properties

# Execute the batch job (normally invoked by cron)
bash $MNAAS_report_data_loading_scriptName
```
To debug, enable `set -x` inside `MNAAS_report_data_loading.sh` and inspect the generated log file.

# Configuration
- **External config files**: `MNAAS_CommonProperties.properties` (provides `$customer_dir`, `$BackupDir`, etc.).  
- **Environment variables**: none required beyond those defined in the sourced common properties.  

# Improvements
1. Externalize the associative arrays to a JSON/YAML file and parse at runtime to simplify maintenance and enable schema validation.  
2. Implement a checksum‑based file integrity check before loading and record the checksum in the process‑status file for auditability.