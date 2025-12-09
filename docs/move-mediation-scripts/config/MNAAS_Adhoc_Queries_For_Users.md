# Summary
The `MNAAS_Adhoc_Queries_For_Users.properties` file supplies environment‑specific constants and per‑function metadata for the **MNAAS Ad‑hoc Queries for Users** job. It is sourced by `MNAAS_Adhoc_Queries_for_users.sh`, which uses the defined variables to drive a series of Hive/Impala SELECT statements (functions 1‑7), write the results as delimited CSV files to a backup directory, and maintain process‑status and log artifacts.

# Key Components
- **Global variables**
  - `MNAAS_Adhoc_Queries_for_users_LogPath` – timestamped local log file.
  - `MNAAS_Adhoc_Queries_for_users_output_dir` – HDFS backup directory for CSV output.
  - `MNAAS_Adhoc_Queries_for_users_ProcessStatus_FileName` – HDFS flag file indicating job success/failure.
  - `MNAAS_Adhoc_Queries_for_users_SriptName` – name of the driver shell script.
  - `MNAAS_Adhoc_Queries_Property_filename` – path to this properties file (self‑reference).
  - `MNASS_Adhoc_Temp_Scripts_filename` – temporary script location used by the driver.

- **Function blocks (func_1 … func_7)**
  - `*_name` – identifier used by the driver to select the block.
  - `*_should_run_or_not` – `"Y"`/`"N"` toggle controlling execution.
  - `*_start_date` / `*_end_date` – inclusive partition dates (YYYY‑MM‑DD) for the WHERE clause.
  - `*_filename` – target CSV file name (written to `MNAAS_Adhoc_Queries_for_users_output_dir`).
  - `*_column_names` – ordered list of columns to SELECT.
  - `*_where` – optional additional Hive filter predicates (e.g., IMSI, BU, Country, etc.).

# Data Flow
1. **Input**  
   - Hive/Impala table `traffic_details_raw_daily_with_no_dups` (partitioned by `partition_date`).  
   - Date range and optional predicates supplied by each function block.

2. **Processing** (`MNAAS_Adhoc_Queries_for_users.sh`)  
   - Source this properties file.  
   - For each `func_n` where `*_should_run_or_not="Y"`:  
     a. Construct a HiveQL `SELECT <*_column_names> FROM traffic_details_raw_daily_with_no_dups WHERE partition_date BETWEEN <*_start_date> AND <*_end_date> <*_where>;`  
     b. Execute via `hive -e` (or `beeline`).  
     c. Stream results to HDFS path `${MNAAS_Adhoc_Queries_for_users_output_dir}/${*_filename}` using `INSERT OVERWRITE DIRECTORY` or local redirection.  
     d. Append execution details to `${MNAAS_Adhoc_Queries_for_users_LogPath}`.

3. **Outputs**  
   - CSV files (semicolon‑delimited, header included) in the backup directory.  
   - Log file with timestamps and Hive execution status.  
   - Process‑status flag file (`*_ProcessStatus_FileName`) set to `SUCCESS` or `FAILURE`.  

4. **Side Effects**  
   - Automatic cleanup of files older than 2 days in the output directory (as described in header comments).  
   - Potential temporary script (`temp.sh`) generation for dynamic query execution.

# Integrations
- **Common property source** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` supplies base paths (`MNAASLocalLogPath`, `BackupDir`, `MNAASConfPath`, `MNAASPropertiesPath`).
- **Driver script** – `MNAAS_Adhoc_Queries_for_users.sh` reads this file; the driver may be invoked by a scheduler (e.g., Oozie, Airflow, cron).
- **HDFS** – Output directory and status flag reside on HDFS; cleanup relies on HDFS `dfs -rm` commands.
- **Hive/Impala** – Query execution engine; column list must match the table schema.
- **Logging framework** – Simple file appends; downstream monitoring may tail the log path.

# Operational Risks
- **Schema drift** – Adding/removing columns in `traffic_details_raw_daily_with_no_dups` without updating `*_column_names` leads to query failures. *Mitigation*: automated schema validation step before execution.
- **Date range mis‑configuration** – Future dates produce empty result sets; past dates may cause large scans. *Mitigation*: enforce date sanity checks in the driver script.
- **Path/permission errors** – Incorrect HDFS paths or insufficient write permissions stop file generation. *Mitigation*: pre‑run permission audit and fail‑fast checks.
- **Large result sets** – Unbounded `func_1` can generate massive CSVs, exhausting disk/HDFS quota. *Mitigation*: enforce size thresholds or require explicit `*_should_run_or_not="N"` for full dumps in production.
- **Stale status flag** – If the driver crashes before updating the status file, downstream jobs may misinterpret job state. *Mitigation*: use atomic rename (`mv temp_status success`) and timeout watchdog.

# Usage
```bash
# 1. Ensure environment points to the correct Hadoop/Hive binaries.
export HADOOP_CONF_DIR=/etc/hadoop/conf
export HIVE_HOME=/opt/hive
export PATH=$PATH:$HIVE_HOME/bin

# 2. Source common properties (optional, already done by the driver):
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# 3. Execute the driver script (normally via scheduler):
bash /app/hadoop_users/MNAAS/MNAAS_Adhoc_Queries_for_users.sh

# 4. Debug –