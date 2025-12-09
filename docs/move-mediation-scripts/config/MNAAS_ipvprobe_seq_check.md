# Summary
Defines runtime constants for the **IPVProbe Sequence Check** batch job. The properties are sourced by `MNAAS_ipvprobe_seq_check.sh` and provide paths for process‑status tracking, logging, script identification, and locations of current, historical, and missing file inventories used to validate IPVProbe feed sequence integrity in the Move‑Mediation production environment.

# Key Components
- `MNAAS_ipvprobe_seq_check_ProcessStatusFilename` – full path to the process‑status file for the job.  
- `MNAAS_seq_check_logpath_ipvprobe` – date‑suffixed log file path.  
- `MNAAS_ipvprobe_seq_check_Scriptname` – name of the executing shell script.  
- `MNASS_SeqNo_Check_ipvprobe_current_missing_files` – directory holding the current missing‑file list.  
- `MNASS_SeqNo_Check_ipvprobe_missing_history_files` – directory holding historical missing‑file snapshots.  
- `MNASS_SeqNo_Check_ipvprobe_history_of_files` – directory holding the full history of processed files.  
- `Dname_MNAAS_ipvprobe_seqno_check` – Hive (or HDFS) temporary table name used for sequence validation.  
- `FEED_FILE_PATTERN` associative array – maps feed type (`ipvprobe`) to its file‑extension pattern sourced from `MNAAS_CommonProperties.properties`.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| 1 | Raw IPVProbe files on HDFS (path derived from `MNAAS_dtail_extn['ipvprobe']`) | `MNAAS_ipvprobe_seq_check.sh` reads file list, compares against expected sequence using `FEED_FILE_PATTERN`. | Updated missing‑file lists in the three `MNASS_SeqNo_Check_ipvprobe_*` directories. |
| 2 | Process‑status file path (`MNAAS_ipvprobe_seq_check_ProcessStatusFilename`) | Script writes job status (STARTED, SUCCESS, FAILURE). | Status visible to orchestration/monitoring. |
| 3 | Log path (`MNAAS_seq_check_logpath_ipvprobe`) | Script appends execution logs. | Log file persisted locally for audit. |
| 4 | Temporary Hive table (`Dname_MNAAS_ipvprobe_seqno_check`) | Optional load of file metadata for SQL‑based checks. | Table populated/cleaned per run. |

External services: HDFS (file enumeration), Hive (temporary table), local filesystem (logs, status files).

# Integrations
- Sources `MNAAS_CommonProperties.properties` for shared constants (`MNAASConfPath`, `MNAASLocalLogPath`, `MNAAS_dtail_extn`).  
- Invoked by the orchestrator that schedules `MNAAS_ipvprobe_seq_check.sh`.  
- Consumes/produces files in directories referenced by other IPVProbe loading jobs (`MNAAS_ipvprobe_daily_recon_loading`, `MNAAS_ipvprobe_monthly_recon_loading`).  
- May be referenced by monitoring dashboards that poll the process‑status file.

# Operational Risks
- **Stale missing‑file lists**: If cleanup fails, old entries may cause false alerts. *Mitigation*: Retain timestamps, purge > 7 days.  
- **Incorrect file‑extension pattern**: Mis‑configuration in `MNAAS_CommonProperties` leads to missed files. *Mitigation*: Validate pattern on script start.  
- **Log rotation overflow**: Daily log files accumulate. *Mitigation*: Implement logrotate or size‑based truncation.  
- **Hive table contention**: Concurrent jobs may attempt to use the same temporary table. *Mitigation*: Namespace tables per run (e.g., append timestamp).  

# Usage
```bash
# Load common properties
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Load IPVProbe sequence check properties
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ipvprobe_seq_check.properties

# Execute the job (debug mode adds -x)
bash $MNAAS_ipvprobe_seq_check_Scriptname
```
To debug, set `set -x` inside the script or run `bash -x $MNAAS_ipvprobe_seq_check_Scriptname`.

# Configuration
- **Environment variables** (in `MNAAS_CommonProperties.properties`):  
  - `MNAASConfPath` – base config directory.  
  - `MNAASLocalLogPath` – local log directory.  
  - `MNAAS_dtail_extn` – associative array of feed‑type to file‑extension regex.  
- **Local config file**: `move-mediation-scripts/config/MNAAS_ipvprobe_seq_check.properties` (the file under analysis).  
- **Directory prerequisites**:  
  - `$MNASS_SeqNo_Check_ipvprobe_foldername` must exist with sub‑folders `current_missing_files`, `missing_history_files`, `history_of_files`.  

# Improvements
1. **Parameterize folder root** – expose `MNASS_SeqNo_Check_ipvprobe_foldername` as a configurable property to avoid hard‑coding path dependencies.  
2. **Add validation block** – on source, verify that all referenced directories exist and are writable; abort with clear error if not.  