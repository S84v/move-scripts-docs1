# Summary
Defines Bash‑sourced runtime constants for the **MNAAS_TrafficDetails_tbl_Load** job, which loads daily raw traffic‑detail records into Hive, manages intermediate file locations, logs, status tracking, and lock coordination for the telecom mediation pipeline.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – sources shared environment defaults (paths, DB names, etc.).
- **Process status file** – `MNAAS_Daily_Traffictable_Load_Raw_ProcessStatusFile`.
- **Log file** – `MNAASDailyTrafficDetailsLoadAggrLogPath` (date‑suffixed).
- **Shell script name** – `MNAASDailyTrafficDetailsLoadAggrSriptName` (`MNAAS_TrafficDetails_tbl_Load.sh`).
- **Backup directory** – `Daily_TrafficDetails_BackupDir`.
- **Hive table identifiers** – `Dname_MNAAS_Insert_Daily_Reject_TrafficDetails_tbl`, `Dname_MNAAS_Insert_Daily_TrafficDetails_tbl`, `Dname_MNAAS_Load_Daily_TrafficDetails_tbl_temp`.
- **HDFS raw data path** – `MNAAS_Daily_Rawtablesload_TrafficDetails_PathName`.
- **Intermediate file paths** – `MNAASInterFilePath_Daily_Traffic_Details_Merge`, `MNAASInterFilePath_Daily_Traffic_Details_MergeFiles`, `MNAASInterFilePath_Daily_Traffic_Details`.
- **Error file** – `MNAAS_Traffic_Error_File`.
- **Duplicate‑removal file locations** – `MNASS_Intermediatefiles_removedups_withdups_filepath`, `MNASS_Intermediatefiles_removedups_withoutdups_filepath`.
- **Merge lock file** – `MergeLockName`.
- **Auxiliary script** – `api_med_data_oracle_loadingSriptName`.
- **File extension constant** – `SEM_EXTN=.SEM`.

# Data Flow
1. **Input**: Raw traffic‑detail files located at `MNAAS_Daily_Rawtablesload_TrafficDetails_PathName` (HDFS).
2. **Processing**: 
   - Load into temporary Hive table `Dname_MNAAS_Load_Daily_TrafficDetails_tbl_temp`.
   - Insert into final tables (`Insert_Daily_TrafficDetails_tbl`, `Insert_Daily_Reject_TrafficDetails_tbl`) after validation.
   - Remove duplicates using paths under `MNASS_Intermediatefiles_removedups_*`.
3. **Output**: 
   - Populated Hive tables.
   - Log file at `MNAASDailyTrafficDetailsLoadAggrLogPath`.
   - Status file updated at `MNAAS_Daily_Traffictable_Load_Raw_ProcessStatusFile`.
   - Backup of processed files in `Daily_TrafficDetails_BackupDir`.
4. **Side Effects**: Creation of lock file `MergeLockName` to signal downstream merge jobs; possible Oracle load via `api_med_data_oracle_loading.sh`.

# Integrations
- **MNAAS_CommonProperties.properties** – provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`, `Files_BackupDir`, etc.).
- **Downstream merge scripts** – consume `MergeLockName` to trigger daily traffic‑details merge.
- **Oracle loading script** – `api_med_data_oracle_loading.sh` may be invoked after Hive load.
- **Hive metastore** – accessed via temporary and final table names.
- **HDFS** – all file paths are HDFS locations.

# Operational Risks
- **Lock contention**: Stale `MergeLockName` can block downstream merges. *Mitigation*: Implement lock expiration and cleanup.
- **Duplicate handling failure**: Incorrect paths for duplicate removal may cause data loss. *Mitigation*: Validate existence of both with‑dups and without‑dups directories before processing.
- **Log growth**: Unbounded log file size due to daily suffix. *Mitigation*: Rotate/compress logs via cron.
- **Path misconfiguration**: Errors in common properties propagate. *Mitigation*: Unit‑test property sourcing and enforce CI lint.

# Usage
```bash
# Source common properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Source this configuration
. /path/to/MNAAS_TrafficDetails_tbl_Load.properties

# Execute the load script (debug mode)
bash -x $MNAASDailyTrafficDetailsLoadAggrSriptName
```
Check `$MNAASDailyTrafficDetailsLoadAggrLogPath` for execution details and `$MNAAS_Daily_Traffictable_Load_Raw_ProcessStatusFile` for status.

# Configuration
- **Environment variables** (provided by `MNAAS_CommonProperties.properties`):
  - `MNAASConfPath`
  - `MNAASLocalLogPath`
  - `Files_BackupDir`
  - `MNAAS_Daily_RAWtable_PathName`
  - `MNAASInterFilePath_Daily`
- **Local file**: `move-mediation-scripts/config/MNAAS_TrafficDetails_tbl_Load.properties` (the file being documented).

# Improvements
1. **Externalize lock handling** – replace plain text lock file with ZooKeeper or HDFS lease to avoid stale locks.
2. **Parameterize date suffix** – allow overriding `date` format via an env var to support back‑fill runs.