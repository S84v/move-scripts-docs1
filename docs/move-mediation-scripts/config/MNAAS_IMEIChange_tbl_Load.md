# Summary
`MNAAS_IMEIChange_tbl_Load.properties` defines runtime constants for the **IMEI‑Change Table Load** batch job in the Move‑Mediation production environment. The properties are sourced by `MNAAS_IMEIChange_tbl_Load.sh` and provide paths for process‑status tracking, logging, error handling, HDFS staging, backup, and Hive table identifiers used to load IMEI‑change data from raw HDFS files into Hive.

# Key Components
- **MNAAS_IMEIChange_tbl_Load_ProcessStatusFileName** – Full path to the job’s process‑status file.  
- **MNAAS_IMEIChange_tbl_Load_log_path** – Local log file path with daily date suffix.  
- **MNAAS_IMEIChange_tbl_Load_SriptName** – Name of the executable shell script.  
- **MNAAS_IMEIChange_tbl_Load_Error_FileName** – Path to the error‑capture file.  
- **MNAASInterFilePath_IMEIChange** – HDFS intermediate directory for daily IMEI‑change files.  
- **Daily_IMEIChange_BackupDir** – Local backup directory for processed files.  
- **MNAAS_Daily_Rawtablesload_IMEIChange_hdfsPathName** – HDFS raw‑table directory for IMEI‑change data.  
- **Dname_MNAAS_Load_Daily_IMEIChange_tbl_temp** – Temporary Hive table name used during load.  
- **Dname_MNAAS_Load_Daily_IMEIChange_tbl** – Target Hive table name for final data.  
- **IMEIChange_feed_inter_tblname** – Hive “intermediate” table identifier.  
- **IMEIChange_feed_tblname** – Hive “final” table identifier.  
- **IMEIChange_feed_inter_tblname_refresh** – Hive `REFRESH` command string for the intermediate table.  
- **IMEIChange_feed_tblname_refresh** – Hive `REFRESH` command string for the final table.  
- **processname_IMEIChange** – Logical name used for monitoring/alerting.

# Data Flow
| Stage | Input | Transformation | Output | Side Effects |
|-------|-------|----------------|--------|--------------|
| 1. Status Init | None | Script creates/updates `$MNAAS_IMEIChange_tbl_Load_ProcessStatusFileName` | Process‑status file | Enables downstream orchestration |
| 2. Log Init | None | Log file opened at `$MNAAS_IMEIChange_tbl_Load_log_path` | Log entries | Auditing |
| 3. HDFS Staging | Raw IMEI‑change files in `$MNAASInterFilePath_IMEIChange` | Files copied to `$MNAAS_Daily_Rawtablesload_IMEIChange_hdfsPathName` | Staged HDFS data | May trigger HDFS quota usage |
| 4. Hive Load (Temp) | Staged HDFS data | `INSERT OVERWRITE` into `$Dname_MNAAS_Load_Daily_IMEIChange_tbl_temp` | Temp Hive table | Consumes Hive resources |
| 5. Refresh Temp Table | Temp table | Execute `$IMEIChange_feed_inter_tblname_refresh` | Refreshed metadata | Ensures downstream queries see latest data |
| 6. Hive Load (Final) | Temp table | `INSERT OVERWRITE` into `$Dname_MNAAS_Load_Daily_IMEIChange_tbl` | Final Hive table | Consumes Hive resources |
| 7. Refresh Final Table | Final table | Execute `$IMEIChange_feed_tblname_refresh` | Refreshed metadata | Same as step 5 |
| 8. Backup | Processed files | Move to `$Daily_IMEIChange_BackupDir` | Archived files | Retains data for audit/recovery |
| 9. Error Capture | Any failure | Write details to `$MNAAS_IMEIChange_tbl_Load_Error_FileName` | Error file | Supports troubleshooting |

External services: HDFS, Hive Metastore, local filesystem for logs/backups.

# Integrations
- **`MNAAS_CommonProperties.properties`** – Provides base variables (`MNAASConfPath`, `MNAASLocalLogPath`, `MNAASInterFilePath_Daily`, `Files_BackupDir`, `MNAAS_Daily_RAWtable_PathName`, `dbname`).  
- **`MNAAS_IMEIChange_tbl_Load.sh`** – Consumes all properties defined herein.  
- **Hive** – Tables referenced by `IMEIChange_feed_inter_tblname` and `IMEIChange_feed_tblname`.  
- **Monitoring/Orchestration** – Process‑status file and `processname_IMEIChange` are read by external job schedulers (e.g., Oozie, Airflow) to determine success/failure.

# Operational Risks
- **Missing or incorrect base variables** (e.g., `MNAASConfPath`) → job fails to locate status or error files. *Mitigation*: Validate existence of all base vars at script start.  
- **Date‑suffix log file collision** if the script runs multiple times per day. *Mitigation*: Include timestamp (`%H%M%S`) or unique identifier.  
- **HDFS quota exhaustion** on `$MNAASInterFilePath_IMEIChange` or raw table path. *Mitigation*: Monitor directory sizes; implement cleanup policy.  
- **Hive table refresh failures** due to metadata lock. *Mitigation*: Add retry logic with exponential back‑off.  
- **Backup directory permission issues** → loss of raw files. *Mitigation*: Pre‑flight permission check and alert.

# Usage
```bash
# Source the properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
. /path/to/move-mediation-scripts/config/MNAAS_IMEIChange_tbl_Load.properties

# Execute the batch job
bash $MNAAS_IMEIChange_tbl_Load_SriptName   # typically invoked by scheduler
```
For debugging, enable shell tracing:
```bash
set -x
bash $MNAAS_IMEIChange_tbl_Load_SriptName
set +x
```
Inspect `$MNAAS_IMEIChange_tbl_Load_log_path` and `$MNAAS_IMEIChange_tbl_Load_Error_FileName` for runtime details.

# Configuration
- **Referenced config files**: `MNAAS_CommonProperties.properties`.  
- **Environment variables required** (populated by common