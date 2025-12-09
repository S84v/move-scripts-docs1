# Summary
Defines runtime constants for the **MNAAS_move_files_from_staging_ipvprobe** batch job. The properties are sourced by `MNAAS_move_files_from_staging_ipvprobe.sh`, which moves staged IPV‑Probe detail files to the production ingestion pipeline, records execution status, and writes a dated log file.

# Key Components
- **MNAAS_move_files_from_staging_ipvprobe_ScriptName** – name of the executable shell script.  
- **MNAAS_move_files_from_staging_ipvprobe_ProcessStatusFileName** – absolute path to the job’s process‑status file.  
- **MNAAS_move_files_from_staging_ipvprobe_LogPath** – full path of the dated log file (`…/MNAAS_move_files_from_staging_ipvprobe.log_YYYY‑MM‑DD`).  
- **MNAASInterFilePath_Daily_IPVProbe_Details** – directory containing daily IPV‑Probe detail files to be moved.  
- **MNAASInterFilePath_Dups_Check** – path to the duplicate‑check master file used for de‑duplication logic.  
- **Max_ipvprobe_files_delay_time** – maximum allowed age (seconds) of a file before it is considered stale (default 86 400 s = 24 h).

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| 1 | Files in `MNAASInterFilePath_Daily_IPVProbe_Details` | Shell script validates file age ≤ `Max_ipvprobe_files_delay_time`; optionally checks against `MNAASInterFilePath_Dups_Check` | Files copied/moved to real‑time ingestion directory (path defined in the script) | Updated process‑status file, log entry |
| 2 | Process‑status file path | Script writes success/failure flag and timestamp | `MNAAS_move_files_from_staging_ipvprobe_ProcessStatusFile` | None |
| 3 | Log path | Script appends detailed execution log | `MNAAS_move_files_from_staging_ipvprobe.log_YYYY‑MM‑DD` | None |

External services: Hadoop/HDFS for source and target directories; optional email/alerting handled by downstream monitoring (not defined in this file).

# Integrations
- **MNAAS_CommonProperties.properties** – sourced at the top; provides base variables (`MNAASConfPath`, `MNAASLocalLogPath`, `MNAASInterFilePath_Daily`, etc.).  
- **MNAAS_move_files_from_staging_ipvprobe.sh** – consumes all properties defined here.  
- **Ingestion pipeline** – receives the moved IPV‑Probe files; downstream Hive/Hadoop jobs process them.  
- **Duplicate‑check subsystem** – reads `MNAASInterFilePath_Dups_Check` to avoid re‑processing already‑ingested records.

# Operational Risks
- **Stale file ingestion** – files older than `Max_ipvprobe_files_delay_time` may be ignored, causing data gaps. *Mitigation*: monitor the status file and alert on “no files processed” conditions.  
- **Path mis‑configuration** – incorrect base variables from `MNAAS_CommonProperties` lead to missing source/target directories. *Mitigation*: validate paths at script start; fail fast with clear error messages.  
- **Duplicate processing** – if `MNAASInterFilePath_Dups_Check` is out‑of‑sync, duplicates may be ingested. *Mitigation*: schedule periodic cleanup of the master file and add checksum verification.  
- **Log file rotation** – unbounded log growth if old logs are not purged. *Mitigation*: implement log‑retention policy (e.g., delete logs > 30 days).

# Usage
```bash
# Source the property file (ensures common properties are loaded)
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
. move-mediation-scripts/config/MNAAS_move_files_from_staging_ipvprobe.properties

# Execute the job
bash $MNAAS_move_files_from_staging_ipvprobe_ScriptName

# Debug – print resolved variables
echo "Script: $MNAAS_move_files_from_staging_ipvprobe_ScriptName"
echo "Status file: $MNAAS_move_files_from_staging_ipvprobe_ProcessStatusFileName"
echo "Log: $MNAAS_move_files_from_staging_ipvprobe_LogPath"
```

# Configuration
- **Environment variables / base properties** (provided by `MNAAS_CommonProperties.properties`):  
  - `MNAASConfPath` – root directory for configuration files.  
  - `MNAASLocalLogPath` – base directory for log files.  
  - `MNAASInterFilePath_Daily` – base directory for daily intermediate files.  
- **Referenced files**:  
  - `MNAAS_move_files_from_staging_ipvprobe_ProcessStatusFile` (created at runtime).  
  - `IPVProbe_Details` directory (source).  
  - `IPVProbe_Details_Dups_Check/IPVProbe_Master_File.txt` (duplicate‑check reference).  

# Improvements
1. **Parameterize `Max_ipvprobe_files_delay_time`** via an external configuration service (e.g., ZooKeeper) to allow dynamic tuning without file edits.  
2. **Add pre‑execution validation** in the shell script to verify existence and permissions of all referenced directories/files; exit with a distinct error code if any check fails.