# Summary
Defines Bash‑sourced runtime constants for the **MNAAS_TrafficDetails_Mergefile_triggerScript** job. It imports common properties, sets the script’s log file path (including a date suffix), and specifies the HDFS lock file used to coordinate daily traffic‑details merge triggers in the mediation pipeline.

# Key Components
- `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` – sources shared environment variables (e.g., `MNAASLocalLogPath`).
- `MNAAS_TrafficDetails_Mergefile_triggerScript_log` – full path to the script’s log file, appended with the current date (`YYYY-MM-DD`).
- `MergeLockName` – absolute HDFS path to the merge‑trigger lock file (`Mergetriggerfile.txt`) used for inter‑process synchronization.

# Data Flow
- **Input:** None directly; relies on environment variables from `MNAAS_CommonProperties.properties`.
- **Output:** Log file written to `$MNAASLocalLogPath/MNAAS_TrafficDetails_Mergefile_triggerScript.log_YYYY-MM-DD`.
- **Side Effects:** Creation/check of the lock file at `MergeLockName` to signal or block downstream merge processes.
- **External Services:** HDFS (for lock file), local filesystem (for log).

# Integrations
- Consumed by `MNAAS_TrafficDetails_Mergefile_triggerScript.sh` (driver script) which reads these variables.
- Lock file is monitored by downstream aggregation scripts (e.g., hourly/daily traffic‑details aggregation jobs) to coordinate execution order.
- Log path integrates with central logging/monitoring infrastructure.

# Operational Risks
- **Lock file contention** – multiple instances may overwrite or delete the lock, causing race conditions. *Mitigation:* enforce single‑instance execution via a PID lock or Hadoop job scheduler.
- **Log path misconfiguration** – missing `MNAASLocalLogPath` leads to script failure. *Mitigation:* validate environment variables at script start.
- **Date suffix format** – if system date format changes, log naming may break downstream log parsers. *Mitigation:* enforce ISO‑8601 date format (`%F`).

# Usage
```bash
# Source common properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Source this config
. /app/hadoop_users/MNAAS/move-mediation-scripts/config/MNAAS_TrafficDetails_Mergefile_triggerScript.properties

# Run the driver script (example)
bash /app/hadoop_users/MNAAS/move-mediation-scripts/MNAAS_TrafficDetails_Mergefile_triggerScript.sh
```
To debug, echo the variables after sourcing:
```bash
echo $MNAAS_TrafficDetails_Mergefile_triggerScript_log
echo $MergeLockName
```

# Configuration
- **Referenced Config File:** `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Environment Variables (provided by common properties):**
  - `MNAASLocalLogPath` – base directory for script logs.
- **Local Variables Defined Here:**
  - `MNAAS_TrafficDetails_Mergefile_triggerScript_log`
  - `MergeLockName`

# Improvements
1. **Add validation block** to verify that `MNAASLocalLogPath` exists and is writable; exit with error if not.
2. **Encapsulate lock handling** in a reusable function (create, check, release) to standardize synchronization across all merge‑trigger scripts.