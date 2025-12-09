# Summary
Shell‑sourced configuration file for the **MNAAS_business_trend_Aggr** job. It imports shared constants and defines the status‑file path, dated log file location, and the driver script name used in the telecom move‑mediation pipeline to aggregate business‑trend metrics.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – sources common environment constants (e.g., `$MNAASConfPath`, `$MNAASLocalLogPath`).
- **`MNAAS_business_trend_Aggr_ProcessStatusFileName`** – full path to the job’s process‑status file.
- **`MNAAS_business_trend_AggrLogPath`** – log file path with a `YYYY-MM-DD` suffix generated at runtime.
- **`MNAASbusinessTrendAggrScriptName`** – name of the executable shell script (`MNAAS_business_trend_Aggr.sh`).

# Data Flow
- **Inputs:** Environment variables from `MNAAS_CommonProperties.properties` (e.g., base config and log directories).
- **Outputs:** 
  - Process status file written by the driver script to `$MNAASConfPath`.
  - Log file written to `$MNAASLocalLogPath` with date suffix.
- **Side Effects:** Creation/overwrite of status and log files; potential email/alert triggers inside the driver script.
- **External Services/DBs:** None defined here; downstream scripts may interact with Hive/HDFS.

# Integrations
- Consumed by `MNAAS_business_trend_Aggr.sh` (referenced via `MNAASbusinessTrendAggrScriptName`).
- Shares constants with all other MNAAS jobs through `MNAAS_CommonProperties.properties`.
- May be invoked by higher‑level orchestration (e.g., Oozie, Airflow) that sets the execution environment.

# Operational Risks
- **Missing common properties file** → job fails at source step. *Mitigation:* Verify file existence and permissions before execution.
- **Log path date formatting error** → log not created, loss of diagnostics. *Mitigation:* Ensure `date` command is available and locale‑independent.
- **Stale status file** → downstream jobs misinterpret job state. *Mitigation:* Clean or archive status files at job start.

# Usage
```bash
# Load configuration
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_business_trend_Aggr.properties

# Run the driver script (debug mode optional via common properties)
bash "$MNAASbusinessTrendAggrScriptName"
```
To debug, enable Bash tracing in `MNAAS_CommonProperties.properties` (e.g., `set -x`).

# Configuration
- **Referenced file:** `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Environment variables required:** `$MNAASConfPath`, `$MNAASLocalLogPath` (provided by the common file).

# Improvements
1. **Parameterize log date format** – expose a variable for the `date` pattern to allow locale‑specific adjustments without editing the file.
2. **Add validation block** – after sourcing, verify that `$MNAASConfPath` and `$MNAASLocalLogPath` are non‑empty and writable; exit with a clear error if not.