# Summary
`MNAAS_HDFSSpace_Checks.properties` supplies runtime constants for the **MNAAS_HDFSSpace_Checks** batch job. It imports shared common properties and defines the script name, the location of the process‑status file, and the HDFS‑space‑check log file (including a date suffix). The constants are consumed by `MNAAS_HDFSSpace_Checks.sh` to perform HDFS capacity validation in the Move‑Mediation production environment.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – source of shared environment variables (e.g., `MNAASConfPath`, `MNAASLocalLogPath`).
- **`MNAAS_HDFSSpace_Checks_ScriptName`** – identifier of the executing shell script (`MNAAS_HDFSSpace_Checks.sh`).
- **`MNAAS_HDFSSpace_Checks_ProcessStatusFileName`** – full path to the status‑tracking file used by the script (`$MNAASConfPath/MNAAS_HDFSSpace_Checks_ProcessStatusFile`).
- **`MNAAS_HDFSSpace_Checks_LogPath`** – full path to the log file, appended with the current date (`$MNAASLocalLogPath/MNAAS_HDFSSpace_Checks_Log.log_YYYY-MM-DD`).

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| **Load** | `MNAAS_CommonProperties.properties` | Bash `.` (source) imports variables | Populates environment with `MNAASConfPath`, `MNAASLocalLogPath`, etc. |
| **Script Execution** (`MNAAS_HDFSSpace_Checks.sh`) | Uses `MNAAS_HDFSSpace_Checks_ProcessStatusFileName` to read/write job status; reads HDFS metrics via `hdfs dfsadmin -report` (or similar). | Performs space‑availability checks, updates status file, writes diagnostic messages. | Writes log entries to `MNAAS_HDFSSpace_Checks_LogPath`; may emit alerts (email, monitoring). |

External services:
- HDFS NameNode (for space metrics)
- Local filesystem (status file, log file)
- Optional alerting system (email/monitoring) invoked by the shell script.

# Integrations
- **Shared Property File**: `MNAAS_CommonProperties.properties` (defines base paths, environment flags).
- **Shell Script**: `MNAAS_HDFSSpace_Checks.sh` (consumes all variables defined herein).
- **HDFS CLI**: invoked by the script to query cluster capacity.
- **Monitoring/Alerting**: downstream components that parse the log or status file for SLA enforcement.

# Operational Risks
- **Missing/Corrupt Common Properties** → script fails to resolve paths. *Mitigation*: health‑check for source file at script start.
- **Incorrect Date Substitution** (`$(date +_%F)`) may produce unexpected filenames if locale changes. *Mitigation*: enforce `LC_ALL=C` or use ISO‑8601 format explicitly.
- **Log File Growth** → unbounded log accumulation. *Mitigation*: implement log rotation or retention policy.
- **Permission Issues** on `$MNAASConfPath` or `$MNAASLocalLogPath`. *Mitigation*: enforce correct Unix ACLs during deployment.

# Usage
```bash
# Ensure common properties are present and sourced
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Run the check (normally invoked by cron or workflow manager)
bash /app/hadoop_users/MNAAS/MNAAS_Scripts/MNAAS_HDFSSpace_Checks.sh

# Debug: print resolved variables
echo "Script: $MNAAS_HDFSSpace_Checks_ScriptName"
echo "Status file: $MNAAS_HDFSSpace_Checks_ProcessStatusFileName"
echo "Log file: $MNAAS_HDFSSpace_Checks_LogPath"
```

# Configuration
- **Environment Variables (from common properties)**
  - `MNAASConfPath` – directory for process‑status files.
  - `MNAASLocalLogPath` – base directory for local logs.
- **Referenced Files**
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
  - `MNAAS_HDFSSpace_Checks_ProcessStatusFile` (created/updated by the script).
- **Derived Paths**
  - `MNAAS_HDFSSpace_Checks_LogPath` includes a date suffix generated at runtime.

# Improvements
1. **Parameterize Date Format** – replace inline `$(date +_%F)` with a configurable variable (e.g., `LogDateFormat`) to allow locale‑independent formatting and easier testing.
2. **Add Validation Block** – include a sanity‑check section that verifies existence and write permission of `$MNAASConfPath` and `$MNAASLocalLogPath` before the script runs, aborting with a clear error if any check fails.