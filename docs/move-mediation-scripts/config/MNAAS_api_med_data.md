# Summary
Defines environment‑specific variables for the **MNAAS api_med_data Sqoop import** job. It sources common MNAAS properties and sets HDFS paths, log filenames, script name, source table name, and the Hive/Impala query template used by the `api_med_data.sh` driver to extract `api_med_data` records via Sqoop into the Move‑Mediation pipeline.

# Key Components
- `api_med_data_Sqoop_ProcessStatusFile` – HDFS flag file indicating Sqoop job status.  
- `api_med_data_table_SqoopLogName` – Local log file for the Sqoop run (timestamped).  
- `MNAAS_Sqoop_api_med_data_Scriptname` – Shell script invoked to execute the Sqoop import (`api_med_data.sh`).  
- `api_med_data_table_Dir` – HDFS target directory for the imported data (`$SqoopPath/api_med_data`).  
- `api_med_data_table_name` – Source relational table name (`api_med_data`).  
- `api_med_data_table_Query` – Sqoop free‑form query template (`select * from api_med_data where $CONDITIONS`).  

# Data Flow
1. **Input**: Relational source table `api_med_data` accessed via JDBC (configured in the driver script).  
2. **Process**: Sqoop executes `api_med_data_table_Query` with `$CONDITIONS` for split‑by parallelism, writes records to `api_med_data_table_Dir` on HDFS.  
3. **Outputs**:  
   - HDFS data files under `$SqoopPath/api_med_data`.  
   - Process‑status flag file (`api_med_data_Sqoop_ProcessStatusFile`).  
   - Local log (`api_med_data_table_SqoopLogName`).  
4. **Side Effects**: Updates status flag for downstream Hive load jobs; may trigger alerts on failure.

# Integrations
- **Common Properties**: `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` supplies `$MNAASConfPath`, `$MNAASLocalLogPath`, `$SqoopPath`, JDBC credentials, and Hadoop environment variables.  
- **Driver Script**: `api_med_data.sh` consumes all variables defined here to construct the Sqoop command.  
- **Downstream Hive Load**: Status flag consumed by Hive table load scripts (e.g., `MNAAS_Actives_tbl_load_driver.properties`).  
- **Monitoring**: Status file read by orchestration tools (Oozie/Airflow) to determine job success.

# Operational Risks
- **Stale Status Flag**: Failure to delete/overwrite the process‑status file may cause downstream jobs to skip processing. *Mitigation*: Ensure driver script atomically creates/cleans flag.  
- **Incorrect JDBC Credentials**: Propagated from common properties; leads to authentication failures. *Mitigation*: Centralize credential rotation and validate before execution.  
- **Data Skew / Split‑by Misconfiguration**: Large table may cause uneven mapper load. *Mitigation*: Define explicit `--split-by` column in driver script.  
- **Log File Growth**: Timestamped logs accumulate. *Mitigation*: Implement log rotation or archival policy.

# Usage
```bash
# Source environment
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source move-mediation-scripts/config/MNAAS_api_med_data.properties

# Execute driver (example)
bash $MNAAS_Sqoop_api_med_data_Scriptname   # runs Sqoop import using variables above
```
To debug, set `set -x` in `api_med_data.sh` and inspect `$api_med_data_table_SqoopLogName`.

# Configuration
- **Referenced Files**: `MNAAS_CommonProperties.properties` (defines base paths, Hadoop/DB configs).  
- **Environment Variables**: `$MNAASConfPath`, `$MNAASLocalLogPath`, `$SqoopPath`, `$HADOOP_CONF_DIR`, JDBC URL/user/password (from common file).  
- **File Paths**: All variables are absolute; ensure HDFS directories exist and have proper permissions.

# Improvements
1. **Parameterize Split‑by Column** – Add a property (e.g., `api_med_data_split_by=id`) and modify the driver to pass `--split-by $api_med_data_split_by` for better parallelism.  
2. **Idempotent Status Handling** – Implement a wrapper that creates a temporary status file and renames it atomically on success, reducing race conditions with downstream consumers.