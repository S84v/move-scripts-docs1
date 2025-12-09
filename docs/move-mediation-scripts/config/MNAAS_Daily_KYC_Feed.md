# Summary
Defines runtime constants for the **MNAAS_Daily_KYC_Feed** Bash pipeline, including script names, status‑file locations, log paths, intermediate file directories, and a delay timeout. Also adds analogous constants for the weekly KYC feed processing.

# Key Components
- `MNAAS_Daily_KYC_Feed_ScriptName` – name of the daily KYC loading shell script.  
- `MNAAS_Daily_KYC_Feed_ProcessStatusFileName` – path to the daily KYC process‑status file.  
- `MNAAS_Daily_KYC_Feed_LogPath` – daily KYC log file location with date suffix.  
- `MNAASInterFilePath_Daily_KYC_Feed_Details` – HDFS/local intermediate directory for daily KYC feed files.  
- `MNAASInterFilePath_Dups_Check` – path to duplicate‑check master file for daily KYC.  
- `Max_ipvprobe_files_delay_time` – maximum allowed delay (seconds) for IPV probe file arrival.  
- `MNAAS_Weekly_KYC_Feed_ProcessStatusFileName` – path to the weekly KYC process‑status file.  
- `MNAAS_Weekly_KYC_Feed_LogPath` – weekly KYC log file location with date suffix.  
- `MNAAS_Weekly_KYC_Feed_ScriptName` – name of the weekly KYC loading shell script.

# Data Flow
- **Inputs**: Daily/weekly KYC source files placed in `$MNAASInterFilePath_Daily/KYC_Feed` (daily) or equivalent weekly directory.  
- **Processing**: Scripts referenced by `*_ScriptName` read source files, perform validation, duplicate checks using `$MNAASInterFilePath_Dups_Check`, and load into Hive/HDFS.  
- **Outputs**: Log files (`*_LogPath`), process‑status files (`*_ProcessStatusFileName`), transformed intermediate files in `$MNAASInterFilePath_Daily_KYC_Feed_Details`.  
- **Side Effects**: Updates of process‑status flags, creation of duplicate‑check master file, possible HDFS writes.  
- **External Services**: HDFS for intermediate storage, Hive for final tables, optional IPV probe monitoring (controlled by `Max_ipvprobe_files_delay_time`).

# Integrations
- Sources common properties from `MNAAS_CommonProperties.properties`.  
- Consumed by the daily and weekly KYC loading scripts (`MNAAS_Daily_KYC_Feed_Loading.sh`, `MNAAS_Weekly_KYC_Feed_Loading.sh`).  
- Interacts with downstream aggregation pipelines that read the generated Hive tables.  
- Shares `$MNAASConfPath`, `$MNAASLocalLogPath`, `$MNAASInterFilePath_Daily` variables defined in the common properties file.

# Operational Risks
- **Stale status files**: Failure to clean old status files may cause false‑positive run detection. *Mitigation*: Include cleanup step at script start.  
- **Log path overflow**: Daily log files accumulate; disk exhaustion possible. *Mitigation*: Implement log rotation/compression.  
- **Duplicate‑check file corruption**: Concurrent runs could corrupt `Daily_KYC_Master_File.txt`. *Mitigation*: Use file locking or move to a transactional store.  
- **Delay timeout misconfiguration**: `Max_ipvprobe_files_delay_time` set too low may abort valid runs. *Mitigation*: Align with SLA for file arrival.

# Usage
```bash
# Source the properties
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source /path/to/MNAAS_Daily_KYC_Feed.properties

# Run daily KYC feed loading
bash $MNAAS_Daily_KYC_Feed_ScriptName

# Debug (enable trace)
set -x
bash $MNAAS_Daily_KYC_Feed_ScriptName
```

# Configuration
- **Referenced config file**: `MNAAS_CommonProperties.properties` (defines `$MNAASConfPath`, `$MNAASLocalLogPath`, `$MNAASInterFilePath_Daily`, etc.).  
- **Environment variables**: Inherited from common properties; no additional env vars required.  

# Improvements
1. Consolidate daily and weekly KYC constants into a single parameter block with a mode flag to reduce duplication.  
2. Replace flat file duplicate‑check (`Daily_KYC_Master_File.txt`) with a Hive/Impala table to enable atomic upserts and better concurrency handling.