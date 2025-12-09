# Summary
`MNAAS_TapErrors_tbl_Load.properties` is a Bash‑sourced configuration fragment for the **TapErrors table load** step of the MNAAS daily mediation pipeline. It defines runtime constants used by `MNAAS_TapErrors_tbl_Load.sh` to track job status, write logs, locate intermediate files, reference Hive/Impala tables, and manage backup/cleanup of TAP‑Errors CSV extracts. The file pulls shared defaults from `MNAAS_CommonProperties.properties`.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – imports common paths, Hadoop/Hive settings, and global variables.  
- **Process‑status file** – `MNAAS_Daily_TapErrors_Load_Aggr_ProcessStatusFileName` (tracks success/failure of the aggregation job).  
- **Log path** – `MNAASDailyTapErrorsLoadAggrLogPath` (daily log file with date suffix).  
- **Script identifier** – `MNAASDailyTapErrorsAggregationScriptName` (name of the invoking shell script).  
- **Intermediate HDFS directories** – `MNAASInterFilePath_Daily_TapErrors` and `MNAASInterFilePath_Daily_TapErrors_rm_header` (raw vs header‑stripped data).  
- **Hive table identifiers** – `Dname_MNAAS_Insert_Daily_TapErrors_tbl`, `Dname_MNAAS_Load_Daily_TapErrors_tbl_temp`, `TapErrors_daily_inter_tblname`, `TapErrors_daily_tblname`.  
- **Refresh statements** – `TapErrors_daily_inter_tblname_refresh`, `TapErrors_daily_tblname_refresh` (Hive `REFRESH` commands).  
- **Error handling file** – `MNAAS_TapErrors_Error_File`.  
- **Raw source path** – `MNAAS_Daily_Rawtablesload_TapErrors_PathName`.  
- **Merged file name** – `Merge_TapErrors_file` (date‑suffixed merged CSV).  
- **File pattern** – `TapErrors_extn=*TAPErrors*.csv`.  
- **Backup directory** – `Daily_TapErrors_BackupDir`.  
- **Process name** – `processname_TapErrors`.  

# Data Flow
| Stage | Input | Transformation | Output / Side‑Effect |
|-------|-------|----------------|----------------------|
| 1. Ingest | CSV files matching `*TAPErrors*.csv` in `MNAAS_Daily_Rawtablesload_TapErrors_PathName` | Header removal → `MNAASInterFilePath_Daily_TapErrors_rm_header` | Header‑stripped files |
| 2. Merge | Header‑stripped files | Concatenate → `Merge_TapErrors_file` (date‑suffixed) | Single merged CSV |
| 3. Load to Hive (temp) | `Merge_TapErrors_file` | `INSERT OVERWRITE` into `Dname_MNAAS_Load_Daily_TapErrors_tbl_temp` | Temp Hive table |
| 4. Refresh temp table | – | Execute `TapErrors_daily_inter_tblname_refresh` | Hive metadata refreshed |
| 5. Insert into final table | Temp table | `INSERT INTO` `TapErrors_daily_tblname` | Production Hive table |
| 6. Refresh final table | – | Execute `TapErrors_daily_tblname_refresh` | Hive metadata refreshed |
| 7. Archive | Processed CSVs | Move to `Daily_TapErrors_BackupDir` | Backup retained |
| 8. Status & Logging | – | Write status to `MNAAS_Daily_TapErrors_Load_Aggr_ProcessStatusFileName`; log to `MNAASDailyTapErrorsLoadAggrLogPath` | Monitoring artifacts |

External services: Hadoop HDFS, Hive/Impala, Linux filesystem, optional alerting via status file.

# Integrations
- **MNAAS_CommonProperties.properties** – supplies base paths (`MNAASConfPath`, `MNAASLocalLogPath`, `MNAASInterFilePath_Daily`, `Files_BackupDir`, etc.).  
- **MNAAS_TapErrors_tbl_Load.sh** – consumes all variables defined herein; invoked by the daily scheduler (e.g., Oozie, Airflow, cron).  
- **Hive/Impala** – tables referenced for insert/refresh operations.  
- **Backup/Retention scripts** – may reference `Daily_TapErrors_BackupDir`.  
- **Monitoring framework** – reads the process‑status file to raise alerts.

# Operational Risks
- **Missing or malformed CSV files** – leads to empty merges; mitigate with pre‑run validation of `TapErrors_extn`.  
- **Date suffix mismatch** (`date +_%F`) causing log or merge file name collisions if the script runs across midnight; mitigate by using a fixed timestamp variable passed from the scheduler.  
- **Path variable mis‑configuration** (e.g., `MNAASConfPath` undefined) → job failure; mitigate with startup sanity checks.  
- **Hive refresh failures** (permissions or metadata lag) → downstream queries see stale data; mitigate by capturing Hive command exit codes and writing to the status file.  
- **Concurrent executions** – race on status file or intermediate directories; mitigate by using lock files or scheduler‑enforced singleton.

# Usage
```bash
# Source common properties first
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Source this file to load TapErrors constants
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_TapErrors_tbl_Load.properties

# Execute the load script (typically invoked by the scheduler)
bash $MNAASLocalScriptPath/$MNAASDailyTapErrorsAggregationScriptName
```
*Debug*: `set -x` before sourcing to echo variable expansions; inspect `$MNAASDailyTapErrorsLoadAggrLogPath` for runtime logs.

# Configuration
- **Referenced files**: `MNAAS_CommonProperties.properties`.  
- **Environment variables expected from common file**: `MNAASConfPath`, `MNAASLocalLogPath`, `MNAASInterFilePath_Daily`, `MNAASLocalScriptPath`, `Files_BackupDir`, `MNAAS_D