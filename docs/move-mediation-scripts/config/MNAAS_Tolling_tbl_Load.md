# Summary
Defines Bash‑sourced runtime constants for the **MNAAS_Tolling_tbl_Load** aggregation job. The variables supply status‑file locations, log file naming, script reference, HDFS staging paths, intermediate directories, duplicate‑handling locations, Hive table identifiers, and raw‑table base paths used by `MNAAS_Tolling_tbl_Load.sh` in the daily tolling‑data load pipeline.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – imports shared environment definitions (e.g., `MNAASConfPath`, `MNAASLocalLogPath`, `MNAASInterFilePath_Daily`, `MNAAS_Dups_Handlingpath`).
- **Status & Log Variables**
  - `MNAAS_Daily_Tolling_Load_Aggr_ProcessStatusFileName` – status‑file path.
  - `MNAAS_DailyTollingLoadAggrLogPath` – daily log file (date‑suffixed).
  - `MNAASDailyTollingAggregationScriptName` – script name invoked by the scheduler.
- **Diff & Error Files**
  - `Tolling_diff_filepath` / `Tolling_diff_filepath_global` – local diff files for change detection.
  - `MNAAS_Tolling_Error_FileName` – consolidated error dump.
- **Intermediate & Duplicate‑Handling Paths**
  - `MNAASInterFilePath_Daily_Tolling` – HDFS staging directory for tolling feeds.
  - `MNAAS_Dups_Handlingpath_Tolling` – HDFS location for sorted duplicate files.
- **Hive Table Identifiers**
  - `Dname_MNAAS_Insert_Daily_Tolling_tbl` – target Hive insert table.
  - `Dname_MNAAS_Load_Daily_Tolling_tbl_temp` – temporary load table.
- **Raw‑Table Base Path**
  - `MNAAS_Daily_Rawtablesload_Tolling_PathName` – root HDFS path for raw tolling tables.

# Data Flow
| Stage | Input | Output | Side Effects |
|-------|-------|--------|--------------|
| **Pre‑load** | Raw tolling files in `MNAASInterFilePath_Daily_Tolling` (HDFS) | Diff files (`Tolling_diff_filepath*`) | Generates diff logs, updates status file |
| **Aggregation** | Diff files, error file path | Populates `Dname_MNAAS_Insert_Daily_Tolling_tbl` (Hive) via temporary `Dname_MNAAS_Load_Daily_Tolling_tbl_temp` | Writes to Hive, moves processed files to duplicate‑handling path |
| **Logging** | Script execution context | `MNAAS_DailyTollingLoadAggrLogPath` (local) | Appends runtime logs |
| **Status Tracking** | Job start/completion | `MNAAS_Daily_Tolling_Load_Aggr_ProcessStatusFileName` | Updated by script to reflect SUCCESS/FAILURE |

External services: HDFS (staging & duplicate dirs), Hive/Impala (tables), local filesystem (logs, diff files).

# Integrations
- **Common Properties** – inherits base paths and Hadoop environment from `MNAAS_CommonProperties.properties`.
- **MNAAS_Tolling_tbl_Load.sh** – consumes all variables defined herein; invoked by scheduler (e.g., Oozie, cron).
- **Downstream Jobs** – downstream reporting or billing jobs read from `Dname_MNAAS_Insert_Daily_Tolling_tbl`.
- **Error Handling** – `MNAAS_Tolling_Error_FileName` is consumed by alerting/monitoring components.

# Operational Risks
- **Path Misconfiguration** – missing or incorrect base variables cause file not found errors. *Mitigation*: validate existence of all derived directories at script start.
- **Date Suffix Collision** – log file name uses `date +_%F`; if script runs multiple times per day, logs may be overwritten. *Mitigation*: include timestamp (`%H%M%S`) or unique identifier.
- **Diff File Overwrite** – global diff file may be overwritten by concurrent runs. *Mitigation*: lock file or use per‑run naming scheme.
- **Hive Table Schema Drift** – changes to target tables not reflected in script may cause load failures. *Mitigation*: schema version check before load.

# Usage
```bash
# Source configuration
source /app/hadoop_users/MNAAS/move-mediation-scripts/config/MNAAS_Tolling_tbl_Load.properties

# Run aggregation (normally invoked by scheduler)
bash $MNAASDailyTollingAggregationScriptName   # resolves to MNAAS_Tolling_tbl_Load.sh
```
For debugging, export `set -x` after sourcing to trace variable expansion.

# Configuration
- **Referenced Env/Config Files**
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Required Environment Variables (provided by common properties)**
  - `MNAASConfPath`
  - `MNAASLocalLogPath`
  - `MNAASInterFilePath_Daily`
  - `MNAAS_Dups_Handlingpath`
  - `MNAAS_Daily_RAWtable_PathName`
- **Local Files Created**
  - `MNAAS_Daily_Tolling_Load_Aggr_ProcessStatusFile`
  - `MNAAS_Tolling_tbl_Load.log_YYYY-MM-DD`
  - `Tolling_diff_file`, `Tolling_diff_file_global`
  - `MNAAS_Tolling_Error_File`

# Improvements
1. **Add Validation Block** – script should verify that all derived directories exist and are writable; exit with clear error codes if not.
2. **Parameterize Log Naming** – replace static date suffix with `date +_%F_%H%M%S` or accept an optional run‑id argument to avoid log collisions in multi‑run scenarios.