# Summary
`MNAAS_IMEIChange_seq_check.properties` defines runtime constants for the **IMEI‑Change Sequence Check** batch job. The constants are sourced by `MNAAS_IMEIChange_seq_check.sh` and drive status‑file handling, logging, HDFS directory references, and the logical name of the check operation in the Move‑Mediation production environment.

# Key Components
- **MNAAS_IMEIChange_seq_check_ProcessStatusFilename** – full path to the process‑status file for the job.  
- **MNAAS_seq_check_logpath_imeichange** – local log file path, includes a daily date suffix.  
- **MNAAS_IMEIChange_seq_check_Scriptname** – script identifier used for notifications and audit trails.  
- **MNASS_SeqNo_Check_imeichange_current_missing_files** – HDFS folder tracking files missing in the current run.  
- **MNASS_SeqNo_Check_imeichange_missing_history_files** – HDFS folder tracking historically missing files.  
- **MNASS_SeqNo_Check_imeichange_history_of_files** – HDFS folder containing the full history of processed files.  
- **Dname_MNAAS_IMEIChange_seqno_check** – logical job name used in monitoring/alerting systems.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| 1. Load constants | `MNAAS_CommonProperties.properties` (via `.` source) | Variable expansion (e.g., `$MNAASConfPath`, `$MNAASLocalLogPath`) | In‑memory constants for the script |
| 2. Read status | `$MNAAS_IMEIChange_seq_check_ProcessStatusFilename` | Determines last successful run, decides if re‑run is needed | Updated status file after successful execution |
| 3. Scan HDFS | `$MNASS_SeqNo_Check_imeichange_foldername/*` (derived folders) | Identify missing sequence numbers for IMEI change files | Files listed in `current_missing_files`, `missing_history_files` |
| 4. Log | `$MNAAS_seq_check_logpath_imeichange` | Append timestamped entries, errors, and summary | Persistent log file per day |
| 5. Notify / monitor | `Dname_MNAAS_IMEIChange_seqno_check` | Send email or push to monitoring system (outside this file) | Alerts on failures or anomalies |

External services: HDFS (file system), local filesystem for logs, optional email/monitoring agents.

# Integrations
- **`MNAAS_CommonProperties.properties`** – provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`, `MNASS_SeqNo_Check_imeichange_foldername`).  
- **`MNAAS_IMEIChange_seq_check.sh`** – consumes all variables defined here; orchestrates HDFS commands (`hdfs dfs -ls`, `-cat`), status updates, and logging.  
- **Monitoring/Alerting** – uses `Dname_MNAAS_IMEIChange_seqno_check` as the identifier for downstream alert pipelines.  
- **Potential downstream Hive jobs** – not defined in this file but typical for Move‑Mediation; may read the same HDFS folders.

# Operational Risks
- **Path mis‑configuration** – incorrect base paths cause HDFS command failures; mitigate with validation at script start.  
- **Date‑suffix log collision** – if the script runs multiple times per day, logs may be overwritten; mitigate by adding a run‑id suffix.  
- **Missing status file** – first‑run scenario may abort; mitigate by initializing the status file if absent.  
- **HDFS latency** – delayed file visibility can produce false “missing” entries; mitigate by adding a configurable grace period.

# Usage
```bash
# Source the property file (normally done inside the .sh script)
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_IMEIChange_seq_check.properties

# Run the job
bash /app/hadoop_users/MNAAS/MNAAS_Scripts/MNAAS_IMEIChange_seq_check.sh

# Debug: print variables
echo "Log path: $MNAAS_seq_check_logpath_imeichange"
echo "Current missing folder: $MNASS_SeqNo_Check_imeichange_current_missing_files"
```

# Configuration
- **Referenced config file**: `MNAAS_CommonProperties.properties` (must be present at the path shown).  
- **Environment variables required by the common file**:  
  - `MNAASConfPath` – directory for process‑status files.  
  - `MNAASLocalLogPath` – base directory for local logs.  
  - `MNASS_SeqNo_Check_imeichange_foldername` – root HDFS folder for IMEI‑change sequence checks.  
- No additional command‑line parameters; all behavior is driven by the constants above.

# Improvements
1. **Add sanity checks** – script should verify that all derived paths exist (both HDFS and local) before proceeding and exit with a clear error code if not.  
2. **Parameterize log suffix** – replace static `$(date +_%F)` with a configurable pattern (e.g., `${LOG_DATE_FORMAT:-%Y%m%d}`) to support multiple runs per day without collision.