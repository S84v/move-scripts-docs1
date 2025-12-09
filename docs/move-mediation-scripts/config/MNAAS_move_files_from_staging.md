# Summary  
Defines runtime constants for the **MNAAS_move_files_from_staging** batch job. The job moves staged CDR and TAP‑error files from the staging area to the real‑time ingestion directory, records its execution status, and writes a dated log file.  

# Key Components  
- **MNAAS_move_files_from_staging_ScriptName** – name of the executable shell script (`MNAAS_move_files_from_staging.sh`).  
- **MNAAS_move_files_from_staging_ProcessStatusFileName** – absolute path to the process‑status file used for monitoring job health.  
- **MNAAS_move_files_from_staging_LogPath** – full path of the log file; includes a date suffix (`$(date +_%F)`).  
- **Holland01_tap_error_extn / Singapore01_tap_error_extn** – filename patterns for TAP‑error CSVs per site.  
- **Holland01_failed_events_extn / Singapore01_failed_events_extn** – filename patterns for failed‑event CDRs per site.  
- **SSH_MOVE_REALTIME_PATH** – target directory on the real‑time CDR ingestion server.  
- **SEM_EXTN** – file extension used to flag “SEM” control files.  
- **FEED_CHECK_RAW_FILE** – sample input file path for feed‑validation utilities.  

# Data Flow  
1. **Input** – Files matching the defined extensions are read from the staging directory (`/Input_Data_Staging/MNAAS_DailyFiles/`).  
2. **Processing** – The shell script copies/moves each matching file to `SSH_MOVE_REALTIME_PATH` via SCP/SSH.  
3. **Output** –  
   - Moved files appear in the real‑time ingestion path.  
   - Execution log written to `MNAAS_move_files_from_staging.log_YYYY-MM-DD`.  
   - Process‑status file updated to indicate success/failure.  
4. **Side Effects** – Potential creation of “.SEM” marker files; removal of source files from staging after successful transfer.  

# Integrations  
- **MNAAS_CommonProperties.properties** – imported at the top; provides base variables (`MNAASConfPath`, `MNAASLocalLogPath`, etc.).  
- **MNAAS_move_files_from_staging.sh** – the actual script that consumes these properties.  
- **SSH/ SCP** – external service used to transfer files to the real‑time server.  
- **Monitoring/Orchestration** – process‑status file is polled by the Move‑Mediation scheduler to trigger alerts or downstream jobs.  

# Operational Risks  
- **Hard‑coded date in log filename** may cause log rotation issues if the script runs multiple times per day.  
- **Missing or mismatched file extensions** leads to unprocessed files and data loss.  
- **SSH connectivity failures** result in incomplete transfers; no retry logic in properties.  
- **Process‑status file path misconfiguration** prevents monitoring systems from detecting job state.  

# Usage  
```bash
# Load common properties
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Export the job‑specific properties (the file being documented)
source /path/to/MNAAS_move_files_from_staging.properties

# Execute the job
bash $MNAAS_move_files_from_staging_ScriptName
```
For debugging, tail the generated log file:
```bash
tail -f $MNAAS_move_files_from_staging_LogPath
```  

# Configuration  
- **Environment variables** supplied by `MNAAS_CommonProperties.properties`:  
  - `MNAASConfPath` – base directory for process‑status files.  
  - `MNAASLocalLogPath` – base directory for log files.  
- **Referenced config files**: `MNAAS_CommonProperties.properties`.  

# Improvements  
1. **Parameterize the log date** (e.g., allow optional timestamp format) and implement log rotation to avoid unbounded file growth.  
2. **Add retry/back‑off logic** for SSH transfers and externalize file‑extension patterns to a separate, version‑controlled dictionary to simplify updates per site.