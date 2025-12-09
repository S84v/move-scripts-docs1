# Summary
Bash‑sourced configuration fragment that defines runtime constants for the **Telena SNG file statistics** step of the MNAAS daily‑processing pipeline. It supplies paths for status tracking, logging, HDFS staging, Hive/Impala table identifiers, and local KPI measurement directories used by `MNAAS_Telena_SNG_files_Stat_process.sh`.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – imports shared environment variables (e.g., `MNAASConfPath`, `MNAASLocalLogPath`, `dbname`).
- **`setparameter='set -x'`** – optional Bash debug flag; comment out to enable verbose tracing.
- **`MNAAS_Telena_files_backup_ProcessStatusFile`** – full path to the status‑file written by the process.
- **`MNAAS_backup_files_logpath_Telena`** – log file path; includes current date (`%_F`).
- **`MNAAS_telena_SNG_files_Stat_Scriptname`** – name of the executable script.
- **`MNAAS_sng_filename_hdfs_Telena`** – HDFS directory where raw Telena SNG feed files are staged.
- **`telena_sng_file_record_count_inter_tblname`** – Hive/Impala intermediate table for per‑file record counts.
- **`telena_sng_file_record_count_tblname`** – final Hive/Impala table that stores aggregated SNG file statistics.
- **`telena_sng_file_record_count_refresh`** – Hive `REFRESH` command string for the final table.
- **`MNAAS_Customer_SNG_Stat_file_path`** – local filesystem directory where KPI measurement files are written.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|-----------------------|
| **Script start** | Environment from `MNAAS_CommonProperties.properties` | Source this file to populate variables | Variables available to `MNAAS_Telena_SNG_files_Stat_process.sh` |
| **File discovery** | Files in HDFS path `$MNAAS_sng_filename_hdfs_Telena` | Count records per file, write intermediate rows to `$telena_sng_file_record_count_inter_tblname` | Intermediate Hive table populated |
| **Aggregation** | Intermediate table | Compute per‑customer aggregates | Final Hive table `$telena_sng_file_record_count_tblname` populated |
| **Refresh** | Final table | Execute `$telena_sng_file_record_count_refresh` | Hive metadata refreshed for downstream queries |
| **Status & Logging** | N/A | Write job status to `$MNAAS_Telena_files_backup_ProcessStatusFile`; append logs to `$MNAAS_backup_files_logpath_Telena` | Status file & log file created/updated |
| **KPI export** | Aggregated data | Export KPI files to `$MNAAS_Customer_SNG_Stat_file_path` | Local KPI measurement files written |

External services: HDFS, Hive/Impala, local filesystem, optional logging infrastructure.

# Integrations
- **`MNAAS_Telena_SNG_files_Stat_process.sh`** – consumes all variables defined here.
- **`MNAAS_CommonProperties.properties`** – provides base paths, DB name, Hadoop user, etc.
- **Hive/Impala** – tables referenced by `telena_sng_file_record_count_inter_tblname` and `telena_sng_file_record_count_tblname`.
- **HDFS** – staging directory `$MNAAS_sng_filename_hdfs_Telena`.
- **Downstream KPI consumers** – read files from `$MNAAS_Customer_SNG_Stat_file_path`.

# Operational Risks
- **Missing common properties** – script fails if `MNAASConfPath` or `dbname` undefined. *Mitigation*: validate required vars at script start.
- **Hard‑coded date in log filename** – creates a new log file each run; old logs may be orphaned. *Mitigation*: implement log rotation or retain a symbolic link to the latest log.
- **Permission issues on HDFS/local paths** – job aborts during read/write. *Mitigation*: enforce ACLs and run as the dedicated Hadoop user.
- **Hive `REFRESH` failure** – stale metadata can cause downstream query errors. *Mitigation*: capture exit code and retry or alert.
- **Debug flag left enabled** – verbose output may fill logs. *Mitigation*: ensure `setparameter` is commented out in production.

# Usage
```bash
# Source configuration
source /app/hadoop_users/MNAAS/config/MNAAS_Telena_SNG_files_Stat_process.properties

# Optional: enable Bash tracing for debugging
# set -x   # uncomment if you need step‑by‑step trace

# Execute the processing script
bash "$MNAAS_telena_SNG_files_Stat_Scriptname"
```

# Configuration
- **Environment variables imported from `MNAAS_CommonProperties.properties`**  
  - `MNAASConfPath` – base directory for status files.  
  - `MNAASLocalLogPath` – base directory for log files.  
  - `dbname` – Hive/Impala database name.  
- **Local paths** – defined directly in this file (see Key Components).  
- **HDFS path** – `/user/MNAAS/FileCount/telena_sng_feed_data`.  

# Improvements
1. **Parameter validation block** – add a function that checks existence and write permissions of all paths before the main script runs.  
2. **Externalize the Hive refresh command** – move `$telena_sng_file_record_count_refresh` to a dedicated SQL file to simplify maintenance and allow version control of DDL/DML statements.