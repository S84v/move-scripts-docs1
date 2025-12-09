# Summary
Defines runtime constants for the **MNAAS_Daily_KYC_Feed_tbl_Load** and **MNAAS_Weekly_KYC_Feed_tbl_Load** Bash pipelines. The properties supply file‑system locations, status‑file names, log paths, intermediate directories, backup locations, and Hive table identifiers used during daily and weekly KYC feed ingestion, deduplication, and loading into MNAAS data warehouse.

# Key Components
- **Import of common properties** – sources `MNAAS_CommonProperties.properties` for base paths (`MNAASConfPath`, `MNAASLocalLogPath`, etc.).
- **Process status files** – `MNAAS_Daily_KYC_Feed_Load_ProcessStatusFileName`, `MNAAS_Weekly_KYC_Feed_Load_ProcessStatusFileName`.
- **Log paths** – `MNAAS_Daily_KYC_Feed_Load_LogPath`, `MNAAS_Weekly_KYC_Feed_Load_LogPath`.
- **Intermediate data directories** – daily/weekly duplicate and non‑duplicate folders, merge file locations, dups‑check master files.
- **Backup directories** – `Daily_KYC_BackupDir`, `Weekly_KYC_BackupDir`.
- **Hive table identifiers** – insert and load table names for daily (`Dname_MNAAS_Insert_Daily_KYC_tbl`, `Dname_MNAAS_Load_Daily_KYC_tbl_temp`) and weekly (`Dname_MNAAS_Insert_Weekly_KYC_tbl`, `Dname_MNAAS_Load_Weekly_KYC_tbl_temp`).
- **Script names** – `MNAAS_Daily_KYC_Feed_LoadScriptName`, `MNAAS_Weekly_KYC_Feed_LoadScriptName`.
- **Error and header‑check files** – `MNAAS_Daily_KYC_Error_FileName`, `MNNAS_Daily_KYC_OnlyHeaderFile`, `MNAAS_Weekly_KYC_Error_FileName`, `MNNAS_Weekly_KYC_OnlyHeaderFile`.
- **One‑time snapshot files** – `KYC_OneTimeSnapshot`, `KYC_OneTimeSnapshotWeekly`.

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| **Raw ingestion** | Raw KYC files in `$MNAAS_Daily_RAWtable_PathName/KYC` (daily) or `$MNAAS_Weekly_RAWtable_PathName/KYC` (weekly) | Bash loader script (`*_tbl_loading.sh`) reads status file, moves files to intermediate dirs (`Daily_Increment_KYC_Feed_*` / `Weekly_Increment_KYC_Feed_*`) | Populated intermediate folders, status file updated to *COMPLETED* | Creation of log entries, possible file moves to backup dirs |
| **Deduplication** | Files in `.../Files_withDups` and `.../Files_withoutDups` | Dedup logic (external script) generates master file (`*_Dups_Check/Daily_KYC_Master_File.txt` or `Weekly_KYC_Master_File.txt`) | Dedup‑checked master file, duplicate‑free dataset | Writes to merge files (`MNAASKYCDailyMergeFiles`, `MNAASKYCWeeklyMergeFiles`) |
| **Load to Hive** | Dedup‑checked CSV (`KYC_OneTimeSnapshot*`) | Hive `INSERT` using table identifiers (`Dname_MNAAS_Insert_*`) and temporary load tables (`Dname_MNAAS_Load_*_temp`) | Data persisted in MNAAS KYC tables | Logs written, error file populated on failure |
| **Backup** | Processed raw files | `cp`/`mv` to `$Daily_KYC_BackupDir` or `$Weekly_KYC_BackupDir` | Archived raw files | Disk usage increase |

External services: Hadoop/HDFS for file storage, Hive/Impala for table loads, OS file system for logs and backups.

# Integrations
- **MNAAS_CommonProperties.properties** – provides base path variables referenced throughout.
- **MNAAS_Daily_KYC_Feed_tbl_loading.sh** / **MNAAS_Weekly_KYC_Feed_tbl_loading.sh** – consume these properties via `source` and execute the pipeline steps.
- **Downstream analytics jobs** – rely on populated Hive KYC tables.
- **Backup retention scripts** – clean `$Daily_KYC_BackupDir` and `$Weekly_KYC_BackupDir`.
- **Error monitoring** – reads `$MNAAS_Daily_KYC_Error_FileName` and `$MNAAS_Weekly_KYC_Error_FileName`.

# Operational Risks
- **Path mis‑configuration** – incorrect base paths cause file not found errors. *Mitigation*: validate existence of all derived directories at script start.
- **Log file growth** – log names include date but no rotation; logs may fill disk. *Mitigation*: implement logrotate or size‑based cleanup.
- **Duplicate handling failure** – if master file generation fails, downstream load may ingest duplicates. *Mitigation*: add checksum validation and abort on duplicate count > 0.
- **Backup directory saturation** – unlimited accumulation of raw files. *Mitigation*: enforce retention policy (e.g., 30 days) via cron cleanup.

# Usage
```bash
# Source the properties
source /app/hadoop_users/MNAAS/config/MNAAS_Daily_KYC_Feed_tbl_Load.properties

# Run daily loader (debug mode)
bash $MNAAS_Daily_KYC_Feed_LoadScriptName -d   # -d enables verbose logging

# Verify status
cat $MNAAS_Daily_KYC_Feed_Load_ProcessStatusFileName
```
Replace script name with weekly variant for weekly runs.

# Configuration
- **Environment variables**: None required beyond those defined in `MNAAS_CommonProperties.properties`.
- **Referenced config files**:
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
  - Status files (`*_ProcessStatusFileName`)
  - Error files (`*_Error_FileName`)
- **Directory variables** (populated from common properties):
  - `MNAASConfPath`
  - `MNAASLocalLogPath`
  - `MNAASInterFilePath_Daily`, `MNAASInterFilePath_Weekly`
  - `MNAAS_Daily_RAWtable_PathName`, `MNAAS_Weekly_RAWtable_PathName`

# Improvements
1. **Parameterize dates** – replace inline `$(date +_%F)` with a variable passed from the orchestrator to ensure consistent naming across steps.
2. **Consolidate duplicate‑check logic** – create a shared function/library for generating master files and snapshots to reduce duplication between daily and weekly sections.