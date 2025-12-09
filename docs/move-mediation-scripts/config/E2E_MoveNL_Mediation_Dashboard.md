# Summary
Defines environment‑specific parameters for the **End‑to‑End Move‑NL Mediation Dashboard** nightly job. It sources common MNAAS properties and sets the shell script name, process‑status file location, and log file name (date‑stamped) used by the dashboard loading pipeline.

# Key Components
- **Source statement** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  Loads shared configuration (e.g., `MNAASConfPath`, `MNAASLocalLogPath`).
- **E2E_MoveNL_Mediation_Dashboard_ScriptName** – `End_to_End_MOVENL_Mediation_Dashboard.sh`  
  Wrapper script executed by the scheduler.
- **E2E_MoveNL_Mediation_Dashboard_ProcessStatusFileName** – `$MNAASConfPath/E2E_MoveNL_Mediation_Dashboard_ProcessStatusFile`  
  File used to record success/failure status.
- **E2E_MoveNL_Mediation_Dashboard_LogName** – `$MNAASLocalLogPath/E2E_MoveNL_Mediation_Dashboard.log$(date +_%F)`  
  Daily log file with ISO‑date suffix.

# Data Flow
| Stage | Input | Transformation | Output | Side Effects |
|-------|-------|----------------|--------|--------------|
| Load | `MNAAS_CommonProperties.properties` | Shell `.` sources variables | Environment variables (`MNAASConfPath`, `MNAASLocalLogPath`, etc.) | None |
| Execute script | No direct input (script reads DB/HDFS) | Dashboard job runs Hive/Hadoop queries, aggregates mediation data | Dashboard artifacts (Hive tables, reports) | Writes to process‑status file and log file |
| Logging | `date` command | Generates timestamped log filename | `E2E_MoveNL_Mediation_Dashboard.log_YYYY-MM-DD` | Log rotation handled by external log‑management |

# Integrations
- **MNAAS_CommonProperties.properties** – central configuration repository for all Move‑Mediation jobs.
- **End_to_End_MOVENL_Mediation_Dashboard.sh** – invoked by cron/ODI scheduler; uses the defined variables.
- **Hadoop/Hive** – underlying data processing performed by the wrapper script.
- **Process‑status monitoring** – external watchdog reads the status file to trigger alerts or downstream jobs.

# Operational Risks
- **Missing common properties file** → job fails at source step. *Mitigation*: verify file existence and permissions before scheduler run.
- **Incorrect `MNAASConfPath`/`MNAASLocalLogPath`** → status or log files written to wrong location. *Mitigation*: include sanity check in wrapper script.
- **Date command format change** → log filename may not match retention policies. *Mitigation*: lock to `date +_%F` or make format configurable.
- **Log file growth** → uncontrolled disk usage. *Mitigation*: integrate log rotation (e.g., logrotate) or size‑based truncation.

# Usage
```bash
# Ensure common properties are accessible
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Export variables defined in the properties file (optional)
export $(grep -v '^#' move-mediation-scripts/config/E2E_MoveNL_Mediation_Dashboard.properties | xargs)

# Run the dashboard job manually
./End_to_End_MOVENL_Mediation_Dashboard.sh
```
To debug, inspect `$MNAASLocalLogPath/E2E_MoveNL_Mediation_Dashboard.log_$(date +%F)` and the process‑status file.

# Configuration
- **Referenced file**: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Environment variables** (populated by common properties):
  - `MNAASConfPath` – base directory for process‑status files.
  - `MNAASLocalLogPath` – base directory for log files.
- **Local properties** (defined in this file):
  - `E2E_MoveNL_Mediation_Dashboard_ScriptName`
  - `E2E_MoveNL_Mediation_Dashboard_ProcessStatusFileName`
  - `E2E_MoveNL_Mediation_Dashboard_LogName`

# Improvements
1. **Add validation block** to confirm that `MNAASConfPath` and `MNAASLocalLogPath` exist and are writable before job execution.  
2. **Externalize the date format** into a variable (e.g., `E2E_LogDatePattern`) to allow flexible log‑naming without script modification.