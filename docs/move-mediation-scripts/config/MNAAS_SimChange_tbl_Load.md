# Summary
`MNAAS_SimChange_tbl_Load.properties` is a Bash‑sourced configuration fragment for the “TrafficDetails” table load step of the MNAAS daily‑processing pipeline. It imports the shared `MNAAS_CommonProperties.properties` file and defines all runtime constants required by the `MNAAS_TrafficDetails_tbl_Load.sh` script: process‑status file, log file name, script name, HDFS source/target directories, intermediate merge paths, error file, backup location, and lock‑file for merge coordination.

# Key Components
- **Source statement** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` (injects common variables).  
- **Process‑status file** – `MNAAS_Daily_Traffictable_Load_Raw_ProcessStatusFileName`.  
- **Log file** – `MNAAS_DailyTrafficDetailsLoadAggrLogPath` (includes current date).  
- **Shell script name** – `MNAASDailyTrafficDetailsLoadAggrSriptName`.  
- **Backup directory** – `Daily_TrafficDetails_BackupDir`.  
- **Hive table identifiers** – `Dname_MNAAS_Insert_Daily_Reject_TrafficDetails_tbl`, `Dname_MNAAS_Insert_Daily_TrafficDetails_tbl`, `Dname_MNAAS_Load_Daily_TrafficDetails_tbl_temp`.  
- **HDFS raw‑table path** – `MNAAS_Daily_Rawtablesload_TrafficDetails_PathName`.  
- **Intermediate merge directories** – `MNAASInterFilePath_Daily_Traffic_Details_Merge`, `..._MergeFiles`, `..._Traffic_Details`.  
- **Error file** – `MNAAS_Traffic_Error_FileName`.  
- **Duplicate‑removal paths** – `MNASS_Intermediatefiles_removedups_withdups_filepath`, `..._withoutdups_filepath`.  
- **Merge trigger lock file** – `MergeLockName`.

# Data Flow
| Stage | Input | Transformation | Output | Side Effects |
|-------|-------|----------------|--------|--------------|
| 1. Load Raw | HDFS path `MNAAS_Daily_Rawtablesload_TrafficDetails_PathName` | Parsed by `MNAAS_TrafficDetails_tbl_Load.sh` | Intermediate files under `MNAASInterFilePath_Daily_Traffic_Details*` | Status file updated |
| 2. Deduplication | Intermediate files | Remove duplicates → `withdups` / `withoutdups` directories | Cleaned files for merge | Logs written to `MNAAS_DailyTrafficDetailsLoadAggrLogPath` |
| 3. Merge | Files in `..._MergeFiles` | Consolidate into final dataset | Final HDFS load to Hive table `Dname_MNAAS_Load_Daily_TrafficDetails_tbl_temp` | Merge lock file `MergeLockName` created/removed |
| 4. Hive Insert | Final dataset | Insert into `Dname_MNAAS_Insert_Daily_TrafficDetails_tbl` (success) or reject table `..._Reject_...` (failure) | Hive tables populated | Errors recorded in `MNAAS_Traffic_Error_FileName` |
| 5. Backup | Processed files | Copy to `Daily_TrafficDetails_BackupDir` | Archived raw files | None |

External services: HDFS, Hive/Impala, Linux filesystem, cron scheduler.

# Integrations
- **Common properties** (`MNAAS_CommonProperties.properties`) – provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`, `Files_BackupDir`, etc.).  
- **Shell driver** – `MNAAS_TrafficDetails_tbl_Load.sh` reads all variables defined here.  
- **Hive scripts** – referenced via table name variables; executed by the driver script via `beeline`/`hive`.  
- **Sqoop jobs** – not directly referenced but may consume the same raw HDFS location.  
- **Monitoring/Alerting** – status file and error file are consumed by downstream health‑check scripts.

# Operational Risks
- **Missing common properties** – pipeline fails at source step; mitigate with existence check and fail‑fast.  
- **Date‑dependent log filename** – if `date` command fails, log path becomes invalid; guard with fallback naming.  
- **Lock‑file race condition** – concurrent runs could overwrite `MergeLockName`; enforce single‑instance execution via `flock`.  
- **Hard‑coded table names** – schema changes require manual updates; consider externalizing to a metadata service.  
- **Backup directory permission issues** – could cause data loss; validate write access before processing.

# Usage
```bash
# Load configuration into current shell
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_SimChange_tbl_Load.properties

# Execute the driver script (typically invoked by cron)
bash "$MNAASDailyTrafficDetailsLoadAggrSriptName"
```
For debugging, export `set -x` before sourcing to trace variable expansion, and inspect `$MNAAS_DailyTrafficDetailsLoadAggrLogPath` after execution.

# Configuration
- **File**: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_SimChange_tbl_Load.properties`  
- **Imported file**: `MNAAS_CommonProperties.properties` (must be present at the same directory).  
- **Environment variables referenced** (populated by common properties):  
  - `MNAASConfPath`  
  - `MNAASLocalLogPath`  
  - `Files_BackupDir`  
  - `MNAAS_Daily_RAWtable_PathName`  
  - `MNAASInterFilePath_Daily`  
  - `MNASS_Intermediatefiles_removedups_filepath`  

# Improvements
1. **Parameter validation** – add a Bash function that verifies all required variables are non‑empty and directories exist before the driver script runs.  
2. **Dynamic table metadata** – replace hard‑coded Hive table names with look‑ups from a central metadata store (e.g., a JSON file or Hive metastore API) to reduce maintenance overhead.