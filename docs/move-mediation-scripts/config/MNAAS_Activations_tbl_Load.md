# Summary
Defines environment‑specific variables for the **MNAAS Activations Table Load** job. It sources common MNAAS properties and sets paths, filenames, script name, and intermediate directories used by the daily activations aggregation and load process that populates the `MNAAS_Insert_Daily_activations_tbl` Hive table.

# Key Components
- **MNAAS_Daily_Activations_Load_Aggr_ProcessStatusFileName** – path to the process‑status flag file.
- **MNAASDailyActivationsLoadAggrLogPath** – timestamped log file location for the aggregation run.
- **MNAASDailyActivationsAggregationScriptName** – executable shell script that drives the load (`MNAAS_Activations_tbl_Load.sh`).
- **Activations_diff_filepath / Activations_diff_filepath_global** – locations of per‑run and global diff files used for change detection.
- **MNAASInterFilePath_Daily_Activations** – HDFS staging directory for intermediate activation files.
- **Dname_MNAAS_Insert_Daily_activations_tbl** – Hive table name for final insert.
- **Dname_MNAAS_Load_Daily_activations_tbl_temp** – Hive temporary table used during load.
- **MNAAS_Activations_Error_FileName** – file that captures parsing or validation errors.
- **MNAAS_Dups_Handlingpath_Activations** – directory where duplicate‑sorted activation files are stored.

# Data Flow
| Stage | Input | Processing | Output |
|------|-------|------------|--------|
| **Source Extraction** | Raw activation feed files (HDFS/local) | Diff detection using `Activations_diff_filepath*` | Filtered new/changed records |
| **Aggregation** | Filtered records | `MNAAS_Activations_tbl_Load.sh` aggregates daily activations | Intermediate files in `MNAASInterFilePath_Daily_Activations` |
| **Load to Hive** | Aggregated files | Hive `INSERT OVERWRITE` into `Dname_MNAAS_Load_Daily_activations_tbl_temp` then `INSERT INTO` `Dname_MNAAS_Insert_Daily_activations_tbl` | Populated production table |
| **Error Handling** | Any parsing/validation failures | Write to `MNAAS_Activations_Error_FileName` | Error log for downstream review |
| **Duplicate Handling** | Duplicate activation records | Sort and store in `MNAAS_Dups_Handlingpath_Activations` | Deduplicated dataset for load |

Side effects: creation/modification of status flag file, log file, error file, and duplicate‑handling directory. No external queues; interacts with HDFS and Hive metastore.

# Integrations
- **Common Properties**: sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` (provides base paths such as `$MNAASConfPath`, `$MNAASLocalLogPath`, `$MNAASInterFilePath_Daily`, `$MNAAS_Dups_Handlingpath`).
- **Shell Script**: `MNAAS_Activations_tbl_Load.sh` consumes the variables defined here to execute Sqoop/Hive commands.
- **Hive Metastore**: target tables `MNAAS_Insert_Daily_activations_tbl` and `MNAAS_Load_Daily_activations_tbl_temp`.
- **HDFS**: staging, diff, and duplicate directories referenced by the variables.

# Operational Risks
- **Stale Status Flag**: If `$MNAAS_Daily_Activations_Load_Aggr_ProcessStatusFileName` is not cleared, downstream jobs may skip execution. *Mitigation*: idempotent cleanup step at job start.
- **Log File Rotation**: Log path includes date; old logs accumulate. *Mitigation*: implement log retention policy.
- **Diff File Corruption**: Corrupted diff files cause missed records. *Mitigation*: checksum validation before processing.
- **Duplicate Handling Path Saturation**: Unlimited growth of sorted duplicate files. *Mitigation*: periodic purge or archiving.

# Usage
```bash
# Load common properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Source this job‑specific properties file
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Activations_tbl_Load.properties

# Execute the aggregation script
bash $MNAASDailyActivationsAggregationScriptName
```
To debug, export `set -x` before invoking the script and inspect `$MNAASDailyActivationsLoadAggrLogPath`.

# Configuration
- **External Config File**: `MNAAS_CommonProperties.properties` (defines `$MNAASConfPath`, `$MNAASLocalLogPath`, `$MNAASInterFilePath_Daily`, `$MNAAS_Dups_Handlingpath`).
- **Environment Variables**: None beyond those sourced; all paths are derived from common properties.

# Improvements
1. **Parameterize Date** – Replace hard‑coded `$(date +_%F)` with a configurable run‑date variable to enable back‑fills.
2. **Centralized Logging** – Redirect all job logs to a unified logging framework (e.g., Log4j via Hadoop) and implement automatic rotation/archival.