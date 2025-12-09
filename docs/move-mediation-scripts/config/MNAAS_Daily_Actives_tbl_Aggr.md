# Summary
Defines runtime variables for the **MNAAS_Daily_Actives_tbl_Aggr** Bash pipeline, including status‑file path, log file location, script name, and control‑file reference. These variables are sourced by the aggregation script during daily active‑subscriber table aggregation in a network‑move production workflow.

# Key Components
- `MNAAS_Daily_Actives_Aggr_ProcessStatusFileName` – path to the process‑status file used to signal start/completion.
- `MNAAS_DailyActivesAggrLogPath` – full log file path, timestamped per execution.
- `MNAASDailyActivesAggrScriptName` – name of the Bash driver script invoked by the scheduler.
- `MNAAS_Daily_Actives_AggrCntrlFileName` – path to the control‑properties file that supplies additional parameters to the aggregation job.

# Data Flow
- **Inputs**: Control file (`MNAAS_Daily_Actives_AggrCntrlFile.properties`), common properties (`MNAAS_CommonProperties.properties`), source tables in Hive/HDFS.
- **Outputs**: Aggregated Hive table (daily actives), log file (`MNAAS_Daily_Actives_tbl_Aggr.log_YYYY-MM-DD`), process‑status file (updated with SUCCESS/FAIL).
- **Side Effects**: Writes to HDFS log directory, updates status file, may trigger downstream jobs via status flag.

# Integrations
- Sourced by `MNAAS_Daily_Actives_tbl_Aggr.sh` (driver script) which runs Sqoop/Hive commands.
- Shares common variables from `MNAAS_CommonProperties.properties`.
- Consumed by downstream reporting scripts that poll the process‑status file.

# Operational Risks
- **Stale status file**: If previous run fails to clean up, downstream jobs may misinterpret state. *Mitigation*: Ensure script always rewrites status file at start and end.
- **Log path collision**: Concurrent runs could overwrite logs if date format not unique. *Mitigation*: Include timestamp or PID in log filename.
- **Missing control file**: Job aborts with unclear error. *Mitigation*: Add pre‑flight check for control file existence.

# Usage
```bash
# Source common and this properties file
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
. /path/to/MNAAS_Daily_Actives_tbl_Aggr.properties

# Execute the driver script
bash $MNAASDailyActivesAggrScriptName
```
For debugging, enable Bash tracing:
```bash
set -x
bash $MNAASDailyActivesAggrScriptName
```

# Configuration
- Environment variable `MNAASConfPath` – base directory for configuration files (defined in common properties).
- Environment variable `MNAASLocalLogPath` – base directory for local logs (defined in common properties).
- External file: `MNAAS_Daily_Actives_AggrCntrlFile.properties` (control parameters for the aggregation job).

# Improvements
1. Append process ID (`$$`) to `MNAAS_DailyActivesAggrLogPath` to guarantee uniqueness across parallel executions.
2. Add a validation function that checks existence and readability of the control file and status file before proceeding, exiting with a clear error code.