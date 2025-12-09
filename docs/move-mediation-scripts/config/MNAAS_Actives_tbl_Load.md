# Summary
Defines environment‑specific variables for the **MNAAS Actives Table Load** job. It imports common MNAAS properties and sets paths, filenames, script names, intermediate directories, control files, and error handling locations used by the daily actives aggregation and Hive table load process in the Move‑Mediation pipeline.

# Key Components
- `MNAAS_Daily_Actives_Load_Aggr_ProcessStatusFileName` – HDFS path to the aggregation process‑status flag file.  
- `MNAAS_DailyActivesLoadAggrLogPath` – Local log file path with daily timestamp.  
- `MNAASDailyActivesAggregationScriptName` – Shell script (`MNAAS_Actives_tbl_Load.sh`) that performs aggregation and load.  
- `MNAASInterFilePath_Daily_Actives` / `MNAASInterFilePath_Daily_ActivesDump` – HDFS intermediate directories for raw actives and dump files.  
- `MNAAS_Dups_Handlingpath_Actives` – HDFS directory for duplicate‑handling sorted files.  
- `MNAAS_Daily_Rawtablesload_Actives_PathName` / `MNAAS_Daily_Rawtablesload_Actives_dumpOfLastLoad_PathName` – HDFS locations for daily raw actives tables and previous‑load dump.  
- `MNAAS_Daily_Actives_AggregationCntrlFileName` – Path to control‑file properties governing aggregation logic.  
- `MNAAS_Actives_Error_FileName` – Path to error‑report file.  
- `Dname_*` variables – Hive table names used during temporary staging and final insertion.  
- `Actives_diff_filepath` / `Actives_diff_filepath_global` – Paths for diff files used in validation/comparison.  
- `Insert_Part_Daily_table_actives_only_last_file_oftheday` – Java class reference for loading only the last file of the day.

# Data Flow
| Stage | Input | Process | Output | Side Effects |
|-------|-------|---------|--------|--------------|
| 1 | Raw actives files (HDFS) | `MNAAS_Actives_tbl_Load.sh` reads from `MNAASInterFilePath_Daily_Actives` | Aggregated actives data | Writes process‑status flag, logs |
| 2 | Aggregated data | Duplicate handling via `MNAAS_Dups_Handlingpath_Actives` | Cleaned dataset | Generates duplicate‑sorted files |
| 3 | Cleaned data | Load into temporary Hive tables (`Dname_MNAAS_Load_Daily_actives_tbl_temp`, `..._at_eod_temp`) | Staging tables | Hive DDL/DML executed |
| 4 | Staging tables | Insert into final Hive tables (`Dname_MNAAS_Insert_Daily_Aggr_actives_tbl`, `..._at_eod`) | Production tables | Updates Hive metastore |
| 5 | Diff generation | Compare current vs. previous load using diff files | `Actives_diff_filepath*` files | May trigger alerts on mismatches |
| 6 | Errors | Written to `MNAAS_Actives_Error_FileName` | Error log | Alerting/monitoring hooks |

# Integrations
- **Common Properties**: Sources `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` for base paths (`$MNAASConfPath`, `$MNAASLocalLogPath`, etc.).  
- **Shell Script**: `MNAAS_Actives_tbl_Load.sh` consumes all variables defined here.  
- **Hive**: Uses Hive CLI/Beeline to create/insert temporary and final tables.  
- **Java Loader**: `com.tcl.mnass.tableloading.raw_table_loading_only_last_file_oftheday` invoked from the shell script for final load.  
- **Monitoring**: Process‑status flag file and log paths are polled by cron‑based watchdogs.  

# Operational Risks
- **Missing/Corrupt Control File** – Aggregation may skip required steps. *Mitigation*: Validate existence of `$MNAAS_Daily_Actives_AggregationCntrlFileName` at script start.  
- **Duplicate File Accumulation** – Unhandled duplicates can cause data inflation. *Mitigation*: Enforce cleanup of `$MNAAS_Dups_Handlingpath_Actives` after each run.  
- **Path Misconfiguration** – Incorrect `$MNAASConfPath` leads to file not found errors. *Mitigation*: Include sanity checks and unit tests for path resolution.  
- **Log Overwrite** – Daily log file name uses `date +_%F`; if script runs multiple times per day, logs may be overwritten. *Mitigation*: Append timestamp with hour/minute or use unique run ID.  

# Usage
```bash
# Load environment variables
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source /path/to/MNAAS_Actives_tbl_Load.properties

# Execute aggregation and load
bash $MNAASDailyActivesAggregationScriptName   # resolves to MNAAS_Actives_tbl_Load.sh
```
For debugging, set `set -x` inside `MNAAS_Actives_tbl_Load.sh` and monitor `$MNAAS_DailyActivesLoadAggrLogPath`.

# Configuration
- **Referenced Files**  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` (base paths)  
  - `$MNAASConfPath/MNAAS_Daily_Actives_AggregationCntrlFile.properties` (control parameters)  
  - `$MNAASConfPath/MNAAS_Actives_Error_File` (error capture)  

- **Environment Variables** (populated by common properties)  
  - `MNAASConfPath`  
  - `MNAASLocalLogPath`  
  - `MNAASInterFilePath_Daily`  
  - `MNAAS_Dups_Handlingpath`  
  - `MNAAS_Daily_RAWtable_PathName`  

# Improvements
1. **Parameter Validation Layer** – Add a pre‑execution validation script that checks existence and permissions of all referenced paths and files; exit with clear error codes.  
2. **Idempotent Log Naming** – Replace static daily log filename with `MNAAS_Actives_tbl_Load.log$(date +_%F_%H%M%S)` to prevent overwriting on multiple runs.