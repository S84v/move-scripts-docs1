# Summary
`MNAAS_Tolling_seq_check.properties` is a Bash‑sourced configuration fragment that defines all runtime constants required by the **MNAAS_Tolling_seq_check.sh** job. It sets paths for status tracking, logging, HDFS staging, local directories, and a pattern map (`FEED_FILE_PATTERN`) used to locate tolling feed files for sequence‑number validation in the telecom mediation pipeline.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – imports shared environment variables and base paths.  
- **Variable definitions** – `MNASS_SeqNo_Check_tolling_foldername`, `MNAAS_Tolling_seq_check_ProcessStatusFilename`, `MNAAS_seq_check_logpath_Tolling`, `MNAAS_Tolling_seq_check_Scriptname`, `MNASS_SeqNo_Check_tolling_current_missing_files`, `Dname_MNAAS_Tolling_seqno_check`, `MNASS_SeqNo_Check_tolling_history_of_files`, `MNASS_SeqNo_Check_tolling_missing_history_files`.  
- **Associative array `FEED_FILE_PATTERN`** – maps logical feed identifiers (`SNG_01`, `HOL_01`) to filename glob patterns that include the tolling‑specific extension (`${MNAAS_dtail_extn['tolling']}`).

# Data Flow
| Element | Direction | Description |
|---------|-----------|-------------|
| **Input files** | Read | Files matching patterns in `FEED_FILE_PATTERN` located under `$MNASS_SeqNo_Check_tolling_foldername` (e.g., `SNG*01*.tolling`). |
| **Process status file** | Write | `$MNAAS_Tolling_seq_check_ProcessStatusFilename` – indicates job success/failure for downstream orchestration. |
| **Log file** | Append | `$MNAAS_seq_check_logpath_Tolling` – timestamped log capturing validation steps and errors. |
| **Missing‑files reports** | Write | `$MNASS_SeqNo_Check_tolling_current_missing_files`, `$MNASS_SeqNo_Check_tolling_missing_history_files` – lists of files absent in the current run and historical gaps. |
| **External services** | – | None directly; relies on HDFS/Hive via downstream scripts if they ingest validated data. |

# Integrations
- **MNAAS_Tolling_seq_check.sh** – consumes all variables defined here; invoked by the orchestration layer (e.g., Oozie, Airflow).  
- **MNAAS_CommonProperties.properties** – provides base paths (`$MNAASConfPath`, `$MNAASLocalLogPath`, `$MNAAS_dtail_extn`) and common Hadoop/Hive configuration.  
- **Downstream validation/reporting jobs** – read the status file and missing‑files reports to trigger alerts or corrective actions.

# Operational Risks
- **Missing base variables** – if `MNAAS_CommonProperties.properties` fails to load, all derived paths become empty, causing file‑not‑found errors. *Mitigation*: verify source file existence and exit on error.  
- **Log filename includes `$(date +_%F)` at definition time** – static timestamp leads to a single log file per job start; subsequent runs within the same day overwrite or append unexpectedly. *Mitigation*: compute log filename at runtime inside the script.  
- **Pattern mismatch** – changes to tolling file naming conventions require updates to `FEED_FILE_PATTERN`; otherwise files are ignored. *Mitigation*: implement a validation step that warns when no files match a pattern.  
- **Hard‑coded folder names** – any relocation of tolling directories requires manual config changes. *Mitigation*: externalize folder roots to a higher‑level property file.

# Usage
```bash
# Source the configuration
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Tolling_seq_check.properties

# Run the processing script (example)
bash $MNAAS_Tolling_seq_check_Scriptname
```
For debugging, echo key variables after sourcing:
```bash
echo "Log path: $MNAAS_seq_check_logpath_Tolling"
declare -p FEED_FILE_PATTERN
```

# Configuration
- **Environment variables / files referenced**
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` – base configuration.  
  - `$MNAASConfPath` – directory for process status files.  
  - `$MNAASLocalLogPath` – root for local log files.  
  - `$MNAAS_dtail_extn['tolling']` – file extension for tolling feeds (e.g., `.tolling`).  

# Improvements
1. **Dynamic log filename** – move `$(date +_%F)` evaluation into `MNAAS_Tolling_seq_check.sh` to ensure a fresh log per execution.  
2. **Pattern validation utility** – add a function that scans `$MNASS_SeqNo_Check_tolling_foldername` for each pattern and logs a warning if zero matches, preventing silent data loss.