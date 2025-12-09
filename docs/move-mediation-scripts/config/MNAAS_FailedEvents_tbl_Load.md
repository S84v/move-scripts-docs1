# Summary
Defines runtime constants for the **MNAAS_FailedEvents_tbl_Load** batch job. Provides paths, filenames, Hive table identifiers, and processing parameters used by `MNAAS_FailedEvents_tbl_Load.sh` to ingest, transform, and load daily failed‑event CDR files into the Move production data lake.

# Key Components
- **MNAAS_Daily_FailedEvents_Load_Aggr_ProcessStatusFileName** – absolute path to the process‑status flag file.  
- **MNAAS_DailyFailedEventsLoadAggrLogPath** – daily log file location (includes date suffix).  
- **MNAASDailyFailedEventsAggregationScriptName** – name of the executing shell script.  
- **MNAASInterFilePath_Daily_FailedEvents*** – set of intermediate HDFS directories for raw input, header‑removed files, column‑mapping files, and temporary feed files.  
- **Dname_MNAAS_Insert_Daily_FailedEvents_tbl** – Hive stored procedure name for final insert.  
- **Dname_MNAAS_Load_Daily_FailedEvents_tbl_temp** – Hive temporary load table name.  
- **MNAAS_FailedEvents_Error_FileName** – path to error‑capture file.  
- **MNAAS_Daily_Rawtablesload_FailedEvents_PathName** – HDFS path for raw failed‑event tables.  
- **FailedEvents_extn** – glob pattern for source CDR files.  
- **Daily_FailedEvents_BackupDir** – HDFS backup directory for processed files.  
- **processname_FailedEvents** – logical name used in monitoring/alerts.  
- **FailedEvents_daily_inter_tblname / FailedEvents_daily_tblname** – Hive table names for intermediate and final datasets.  
- **FailedEvents_daily_inter_tblname_refresh / FailedEvents_daily_tblname_refresh** – Hive `refresh` commands for metadata sync.

# Data Flow
1. **Input**: CDR files matching `*FailedEvents*.cdr` located in `$MNAASInterFilePath_Daily_FailedEvents`.  
2. **Pre‑processing**: Header removal, column mapping, and temporary feed generation using the intermediate paths.  
3. **Load**: Data staged into `move_cdrs_raw_failed_events_inter` (temp Hive table).  
4. **Insert**: Stored procedure `MNAAS_Insert_Daily_FailedEvents_tbl` moves data to `move_cdrs_raw_failed_events`.  
5. **Output**: Log file (`MNAAS_FailedEvents_tbl_Load.log_YYYY-MM-DD`), status flag file, error file (if any), and backup of raw files in `$Daily_FailedEvents_BackupDir`.  
6. **Side Effects**: Hive metadata refresh commands executed; HDFS directories updated; process‑status file toggled.

# Integrations
- Sources **MNAAS_CommonProperties.properties** for shared base paths (`$MNAASConfPath`, `$MNAASLocalLogPath`, `$MNAASInterFilePath_Daily`, etc.).  
- Invoked by **MNAAS_FailedEvents_tbl_Load.sh** which orchestrates the ETL steps.  
- Interacts with Hive/Impala via stored procedure `MNAAS_Insert_Daily_FailedEvents_tbl` and `refresh` commands.  
- Utilises common email/alerting lists defined in `MNAAS_Environment.properties` (not shown but referenced by the shell script).  

# Operational Risks
- **Missing or corrupted CDR files** → job aborts; mitigate with pre‑run file existence check and retry logic.  
- **Stale process‑status flag** → downstream jobs may start prematurely; enforce atomic write and cleanup at job end.  
- **Hive procedure failure** → data inconsistency; add transaction‑style rollback or idempotent load pattern.  
- **Insufficient HDFS storage** for backups → monitor `$Files_BackupDir` usage; implement retention policy.  

# Usage
```bash
# Source common properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Run the batch job (normally scheduled via cron/oozie)
bash /app/hadoop_users/MNAAS/move-mediation-scripts/config/MNAAS_FailedEvents_tbl_Load.sh
```
To debug, export `set -x` before invoking the script and inspect the generated log file at `$MNAASLocalLogPath`.

# Configuration
- **Env vars**: Inherited from `MNAAS_CommonProperties.properties` (e.g., `MNAASConfPath`, `MNAASLocalLogPath`, `MNAASInterFilePath_Daily`, `Files_BackupDir`).  
- **Referenced config files**: `MNAAS_CommonProperties.properties`, `MNAAS_Environment.properties` (for global settings).  

# Improvements
1. **Parameterise file pattern** – expose `FailedEvents_extn` as a configurable variable to support future naming changes without script edit.  
2. **Add explicit exit codes and alert hooks** – integrate with monitoring system to raise alerts on non‑zero exit status or missing status flag.