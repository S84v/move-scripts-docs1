# Summary
Defines runtime constants for the **MNAAS_RawFileCount_Checks** batch job. The properties are sourced by `MNAAS_RawFileCount_Checks.sh`, which validates the count of raw input files, writes a dated log, and updates a process‑status file for idempotency and monitoring in the daily mediation move pipeline.

# Key Components
- **MNAAS_CommonProperties.properties** – central repository of shared paths (`MNAASConfPath`, `MNAASLocalLogPath`, etc.).
- **MNAAS_RawFileCount_Checks_ScriptName** – name of the executable shell script.
- **MNAAS_RawFileCount_Checks_ProcessStatusFileName** – absolute path to the process‑status file used to record success/failure.
- **MNAAS_RawFileCount_Checks_LogPath** – directory path for dated log files.

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| Shell script start | `MNAAS_RawFileCount_Checks.sh` (invoked manually or via scheduler) | Sources this properties file; resolves `$MNAASConfPath` and `$MNAASLocalLogPath` from common properties. | Resolved variables for script execution. | None |
| Validation logic (Python/Java class) | List of raw files in ingestion directories (derived from common config) | Counts files, compares against expected thresholds. | Exit code, status string. | Writes entry to `$MNAAS_RawFileCount_Checks_LogPath/<date>.log`. |
| Status update | Validation result | Writes `SUCCESS` or `FAILURE` to `$MNAAS_RawFileCount_Checks_ProcessStatusFileName`. | Process‑status file content. | Enables downstream jobs to check idempotency. |

External services: Hadoop/HDFS for raw file enumeration; local filesystem for logs and status file.

# Integrations
- **MNAAS_CommonProperties.properties** – provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`).
- **MNAAS_RawFileCount_Checks.sh** – consumes the variables defined here.
- Downstream jobs (e.g., `MNAAS_PreValidation_Checks`) read the process‑status file to decide whether to proceed.
- Monitoring/alerting tools poll the process‑status file and log directory.

# Operational Risks
- **Path misconfiguration** – incorrect `$MNAASConfPath` leads to missing status file; mitigate with validation at script start.
- **Log directory permission errors** – prevent log creation; mitigate by ensuring the Hadoop user owns `$MNAASLocalLogPath`.
- **Stale status file** – old success flag may cause downstream jobs to skip; mitigate by timestamping status entries or clearing file at job start.

# Usage
```bash
# Source common and job‑specific properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_RawFileCount_Checks.properties

# Execute the job (typically via scheduler)
bash $MNAASConfPath/MNAAS_RawFileCount_Checks.sh
```
To debug, echo resolved variables after sourcing:
```bash
echo "Script: $MNAAS_RawFileCount_Checks_ScriptName"
echo "Status file: $MNAAS_RawFileCount_Checks_ProcessStatusFileName"
echo "Log path: $MNAAS_RawFileCount_Checks_LogPath"
```

# Configuration
- **Environment variables**: None defined here; all derived from `MNAAS_CommonProperties.properties`.
- **Referenced config files**:  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` – must be present and readable.  

# Improvements
1. **Add validation block** in the properties file to assert that `$MNAASConfPath` and `$MNAASLocalLogPath` are non‑empty and writable.
2. **Version the process‑status file** (e.g., include run timestamp) to avoid stale status reuse across daily runs.