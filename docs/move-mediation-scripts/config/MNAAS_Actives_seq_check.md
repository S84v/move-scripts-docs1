# Summary
Defines environment‑specific variables for the **MNAAS Actives Sequence Check** job. Sources common MNAAS properties, sets log path, script name, status‑file location, HDFS folder references, and a map of expected feed file name patterns used by `MNAAS_Actives_seq_check.sh` to validate the presence and completeness of daily actives files in the Move‑Mediation pipeline.

# Key Components
- `MNAAS_seq_check_logpath_Actives` – full path to the timestamped log file.  
- `Dname_MNAAS_Actives_seqno_check` – directory name identifier for the seq‑no check.  
- `MNAAS_Actives_seq_check_ProcessStatusFilename` – path to the process‑status flag file.  
- `MNAAS_Actives_seq_check_Scriptname` – executable shell script name.  
- `MNASS_SeqNo_Check_actives_current_missing_files` – HDFS/local folder for current missing file list.  
- `MNASS_SeqNo_Check_actives_history_of_files` – folder for historical file list.  
- `MNASS_SeqNo_Check_actives_missing_history_files` – folder for missing‑history records.  
- `FEED_FILE_PATTERN` (associative array) – mapping of feed prefixes (`SNG_01`, `HOL_01`) to expected filename glob patterns, incorporating the `actives` extension from `MNAAS_dtail_extn`.

# Data Flow
| Stage | Input | Processing | Output / Side Effect |
|-------|-------|-------------|----------------------|
| 1 | Common properties (`MNAAS_CommonProperties.properties`) | Variable interpolation, path construction | Populated environment variables for the check job |
| 2 | HDFS/local actives folders (`MNASS_SeqNo_Check_actives_*`) | `MNAAS_Actives_seq_check.sh` reads these directories | Lists of missing/current files |
| 3 | `FEED_FILE_PATTERN` | Used by the script to match expected filenames | Validation report, status flag file update |
| 4 | Log path | Script writes execution details | `MNAAS_Actives_seq_check.log_YYYY-MM-DD` |

External services: HDFS (or local FS) for folder reads/writes; no direct DB or queue interaction.

# Integrations
- **Common Properties**: `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` – provides base paths (`MNAASLocalLogPath`, `MNAASConfPath`, `MNAAS_dtail_extn`).  
- **Sequence‑Check Script**: `MNAAS_Actives_seq_check.sh` consumes all variables defined herein.  
- **Downstream Jobs**: Status file (`MNAAS_Actives_seq_check_ProcessStatusFilename`) is polled by orchestration (cron/airflow) to trigger subsequent aggregation or alerting steps.

# Operational Risks
- **Missing Common Properties**: Failure to source `MNAAS_CommonProperties.properties` results in undefined paths → script aborts. *Mitigation*: Validate file existence at start of script.  
- **Pattern Mismatch**: Incorrect `FEED_FILE_PATTERN` values cause false‑negative missing‑file alerts. *Mitigation*: Unit‑test pattern generation against a sample file set.  
- **Log Path Permission**: Insufficient write permission on `MNAASLocalLogPath` prevents log creation. *Mitigation*: Enforce ACLs during deployment.  
- **Stale Status File**: If the status file is not cleaned, downstream jobs may misinterpret job state. *Mitigation*: Ensure script removes or overwrites the file on each run.

# Usage
```bash
# Load variables into the current shell (optional for debugging)
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source /path/to/MNAAS_Actives_seq_check.properties

# Execute the sequence‑check script
bash $MNAAS_Actives_seq_check_Scriptname   # typically invoked by cron
```
To debug, echo variables after sourcing and run the script with `-x` for trace.

# Configuration
- **Referenced Config File**: `MNAAS_CommonProperties.properties` (defines base paths and `MNAAS_dtail_extn`).  
- **Environment Variables Expected** (provided by common properties):  
  - `MNAASLocalLogPath`  
  - `MNAASConfPath`  
  - `MNAAS_dtail_extn` (associative array with key `'actives'`).  
- **Local/HDFS Folder Variables**: `MNASS_SeqNo_Check_actives_foldername` (defined in common properties) used to build folder paths.

# Improvements
1. **Validate Required Variables** – Add a function that checks existence and non‑emptiness of all critical variables (log path, status file, folder names) before script execution.  
2. **Externalize Feed Patterns** – Move `FEED_FILE_PATTERN` definitions to a separate JSON/YAML file to allow runtime updates without modifying the properties script.