# Summary
Defines Bash‑sourced runtime constants for the **MNAAS_TrafficDetails_tbl_Load_xm** job. The job loads daily raw “xm” traffic‑detail records into Hive, manages intermediate file locations, logs, process‑status tracking, and HDFS lock coordination within the mediation pipeline.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – sources shared environment defaults (paths, DB names, log locations).  
- **`MNAAS_Daily_Traffictable_Load_Raw_ProcessStatusFileName`** – HDFS file that records the status of the raw‑load process.  
- **`MNAAS_DailyTrafficDetailsLoadAggrLogPath`** – local log file path with date suffix.  
- **`MNAASDailyTrafficDetailsLoadAggrSriptName`** – script name invoked by the scheduler.  
- **`Daily_TrafficDetails_BackupDir`** – HDFS backup directory for raw traffic‑detail files.  
- **`Dname_*` variables** – Hive table names for reject, final, and temporary traffic‑detail tables (xm variant).  
- **`MNAAS_Daily_Rawtablesload_TrafficDetails_PathName`** – HDFS directory containing raw “xm” traffic‑detail files.  
- **`MNAASInterFilePath_*`** – HDFS intermediate directories for merge, merge‑input, and raw traffic‑detail files.  
- **`MNAAS_Traffic_Error_FileName`** – HDFS file for persisting load errors.  
- **`MNASS_Intermediatefiles_removedups_*_filepath`** – HDFS locations for deduplication output (with/without duplicates).  
- **`MergeLockName`** – HDFS lock file used to serialize daily merge triggers.  
- **`api_med_data_oracle_loadingSriptName`** – downstream script that loads processed data into Oracle.  
- **`SEM_EXTN`** – file‑extension marker for semaphore files.

# Data Flow
1. **Input**: Raw “xm” traffic‑detail files located at `MNAAS_Daily_Rawtablesload_TrafficDetails_PathName`.  
2. **Processing**:  
   - Load into temporary Hive table (`Dname_MNAAS_Load_Daily_TrafficDetails_tbl_temp`).  
   - Perform deduplication, writing results to `MNASS_Intermediatefiles_removedups_*_filepath`.  
   - Insert clean rows into final Hive table (`Dname_MNAAS_Insert_Daily_TrafficDetails_tbl`).  
   - Reject rows written to `Dname_MNAAS_Insert_Daily_Reject_TrafficDetails_tbl`.  
3. **Output**:  
   - Populated Hive tables (final & reject).  
   - Log file at `MNAAS_DailyTrafficDetailsLoadAggrLogPath`.  
   - Process‑status file updated (`MNAAS_Daily_Traffictable_Load_Raw_ProcessStatusFileName`).  
   - Error file (`MNAAS_Traffic_Error_FileName`) if failures occur.  
4. **Side Effects**: Creation of semaphore files, HDFS lock acquisition via `MergeLockName`, backup of raw files to `Daily_TrafficDetails_BackupDir`.  
5. **External Services**: Hive metastore, HDFS, Oracle DB (via `api_med_data_oracle_loading.sh`).

# Integrations
- **Common Properties**: Inherits base paths and DB names from `MNAAS_CommonProperties.properties`.  
- **Trigger Script**: Coordinated with `MNAAS_TrafficDetails_Mergefile_triggerScript_xm` via `MergeLockName`.  
- **Oracle Load**: Calls `api_med_data_oracle_loading.sh` after successful Hive load.  
- **Scheduler**: Executed by Oozie/Airflow job that references `MNAASDailyTrafficDetailsLoadAggrSriptName`.  

# Operational Risks
- **Lock contention**: Stale `MergeLockName` can block downstream merges. *Mitigation*: Periodic lock cleanup, timeout logic in trigger script.  
- **Path misconfiguration**: Incorrect `MNAASConfPath` or `MNAASLocalLogPath` leads to missing logs/status files. *Mitigation*: Validate paths at script start.  
- **Deduplication failure**: Corrupt intermediate files cause job abort. *Mitigation*: Add checksum verification before processing.  
- **Hive table schema drift**: Table name variables may diverge from actual Hive schema. *Mitigation*: Schema version check in pre‑run validation.  

# Usage
```bash
# Source properties
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
# Export variables defined in this file
source move-mediation-scripts/config/MNAAS_TrafficDetails_tbl_Load_xm.properties

# Run the load script (typically invoked by scheduler)
bash $MNAASDailyTrafficDetailsLoadAggrSriptName
```
For debugging, enable `set -x` before invoking the script and inspect `$MNAAS_DailyTrafficDetailsLoadAggrLogPath`.

# Configuration
- **Environment Variables** (provided by common properties): `MNAASConfPath`, `MNAASLocalLogPath`, `Files_BackupDir`, `MNAAS_Daily_RAWtable_PathName`, `MNAASInterFilePath_Daily`, `MNASS_Intermediatefiles_removedups_filepath`.  
- **Config Files Referenced**: `MNAAS_CommonProperties.properties`.  

# Improvements
1. **Add lock timeout handling** – automatically release `MergeLockName` after a configurable TTL.  
2. **Introduce unit‑test harness** – mock HDFS/Hive interactions to validate variable resolution and path construction.