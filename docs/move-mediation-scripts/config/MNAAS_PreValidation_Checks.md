# Summary
Defines runtime constants for the **MNAAS_PreValidation_Checks** batch job. The script `MNAAS_PreValidation_Checks.sh` uses these properties to locate common configuration, set its own script name, process‑status file, log file (dated), and parameters for a validation class that checks for missing Telena files. The job is part of the daily mediation move pipeline and records execution status for monitoring and idempotency.

# Key Components
- **MNAAS_CommonProperties.properties** – sourced common configuration (paths, Hadoop settings, etc.).
- `MNAAS_PreValidation_Checks_ScriptName` – name of the executable shell script.
- `MNAAS_PreValidation_Checks_ProcessStatusFileName` – absolute path to the process‑status file used for idempotency and health‑check.
- `MNAAS_PreValidation_Checks_LogPath` – log file path with date suffix.
- `Dname_check_telena_files` – logical name for the Telena‑file‑check step.
- `MNAAS_check_telena_files_classname` – fully‑qualified Java class (`com.tcl.mnass.validation.check_for_missing_telena_files`) that implements the validation logic.

# Data Flow
| Stage | Input | Processing | Output / Side Effect |
|-------|-------|------------|----------------------|
| 1. Load common props | `MNAAS_CommonProperties.properties` | Environment variable expansion (`$MNAASConfPath`, `$MNAASLocalLogPath`) | Populates shared paths |
| 2. Execute script | N/A (triggered by scheduler) | `MNAAS_PreValidation_Checks.sh` reads this properties file | Uses defined constants |
| 3. Validation class | Telena file locations (derived from shared config) | Java class `check_for_missing_telena_files` scans HDFS / local FS for expected Telena files | Returns success/failure flag |
| 4. Status update | Validation result | Writes to `MNAAS_PreValidation_Checks_ProcessStatusFile` (e.g., SUCCESS/FAIL) | Enables downstream jobs to check readiness |
| 5. Logging | Log statements from script & Java class | Appends to `MNAAS_PreValidation_Checks.log_YYYY-MM-DD` | Auditable trace |

External services: Hadoop HDFS (file presence checks), possibly Hive metastore if class queries metadata.

# Integrations
- **Common property file** (`MNAAS_CommonProperties.properties`) provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`) used across all MNAAS jobs.
- **Downstream jobs** read the process‑status file to determine if pre‑validation succeeded before proceeding (e.g., ingestion, aggregation jobs).
- **Java validation library** (`com.tcl.mnass.validation.*`) is shared across multiple validation scripts.
- **Scheduler** (e.g., Oozie, Airflow, cron) invokes `MNAAS_PreValidation_Checks.sh` and monitors the status file.

# Operational Risks
- **Missing common property file** → script fails early. *Mitigation*: verify file existence at start, abort with clear error.
- **Incorrect path expansion** (`$MNAASConfPath`/`$MNAASLocalLogPath`) → status or log files written to wrong location. *Mitigation*: validate resolved paths before use.
- **Date suffix in log path** evaluated at source time, not at execution → stale log name if script sourced multiple times. *Mitigation*: compute log filename inside the script, not in properties.
- **Java class not on classpath** → validation step fails silently. *Mitigation*: enforce classpath check and exit on error.
- **Process‑status file not cleaned** → false‑positive downstream runs. *Mitigation*: script should truncate or overwrite status file each run.

# Usage
```bash
# Source properties (usually done inside the shell script)
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
. move-mediation-scripts/config/MNAAS_PreValidation_Checks.properties

# Run the validation script
bash $MNAASConfPath/../MNAAS_PreValidation_Checks.sh

# Debug: force verbose logging
export LOG_LEVEL=DEBUG
bash $MNAASConfPath/../MNAAS_PreValidation_Checks.sh
```

# Configuration
- **Environment variables** (provided by `MNAAS_CommonProperties.properties`):
  - `MNAASConfPath` – directory containing configuration files.
  - `MNAASLocalLogPath` – base directory for log files.
- **Referenced files**:
  - `MNAAS_CommonProperties.properties` (shared across all jobs).
  - Java class JAR containing `com.tcl.mnass.validation.check_for_missing_telena_files`.
- **Properties defined in this file** (see *Key Components*).

# Improvements
1. **Dynamic log filename** – move `$(date +_%F)` evaluation into the shell script to guarantee correct date per execution.
2. **Add validation of required variables** – script should exit with a non‑zero code if `MNAASConfPath` or `MNAASLocalLogPath` are undefined or unwritable.