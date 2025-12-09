# Summary
`usage_overage_charge.properties` is a Bash‑style configuration wrapper for the Move mediation pipeline’s “usage overage charge” job. It sources the global `MNAAS_CommonProperties.properties` file and defines all environment‑specific variables required by the driver script `usage_overage_charge.sh`, including log path, process‑status file, Hadoop class names, Hive table names, refresh statements, and the script name. The file enables a single source of truth for job parameters across environments.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – imports shared properties (e.g., `$MNAASLocalLogPath`, `$dbname`).
- **`setparameter='set -x'`** – optional Bash debug flag; comment out to disable tracing.
- **`MNAAS_usage_overage_charge_LogPath`** – full path for the job log file, timestamped per run.
- **`usage_overage_charge_ProcessStatusFileName`** – HDFS/local file used to record job status (STARTED, SUCCESS, FAILURE).
- **`Dname_usage_overage_charge`** – logical job identifier.
- **`usage_overage_charge_temp_classname` / `usage_overage_charge_classname`** – fully‑qualified Java class names invoked by Sqoop/Hive.
- **`usage_overage_charge_temp_tblname` / `usage_overage_charge_tblname`** – Hive temporary and final table names.
- **`usage_overage_charge_temp_tblname_refresh` / `usage_overage_charge_tblname_refresh`** – Hive `REFRESH` commands for the respective tables.
- **`MNAAS_usage_overage_charge_tblname_ScriptName`** – driver script filename (`usage_overage_charge.sh`).

# Data Flow
| Stage | Input | Processing | Output / Side Effect |
|-------|-------|------------|----------------------|
| 1. Configuration load | `MNAAS_CommonProperties.properties` | Source variables into current shell | Populated environment variables |
| 2. Driver execution (`usage_overage_charge.sh`) | Hive tables, source data (billing view) | Java class (`usage_overage_charge_classname`) runs via Sqoop/Hive to compute overage charges | Populated Hive tables (`usage_overage_charge_table_temp`, `usage_overage_charge_table`) |
| 3. Table refresh | Hive tables | Execute `$usage_overage_charge_temp_tblname_refresh` and `$usage_overage_charge_tblname_refresh` | Hive metadata refreshed |
| 4. Logging & status | Job log path, status file | Append log entries, write status (`STARTED`, `COMPLETED`, `FAILED`) | Persistent log file, status file for monitoring |

External services: Hadoop HDFS, Hive Metastore, possibly Sqoop for data import, Java runtime for billing view classes.

# Integrations
- **Global properties**: `MNAAS_CommonProperties.properties` provides base paths, DB name, Hadoop environment settings.
- **Driver script**: `usage_overage_charge.sh` sources this file; uses defined variables to construct Sqoop/Hive commands.
- **Java classes**: `com.tcl.mnaas.billingviews.usage_overage_charge_temp` and `com.tcl.mnaas.billingviews.usage_overage_charge` are invoked by the driver.
- **Hive**: Temporary and final tables are refreshed via the generated `REFRESH` statements.
- **Monitoring**: Process status file is consumed by orchestration/alerting tools to track job health.

# Operational Risks
- **Missing or stale common properties** – job may fail to resolve `$MNAASLocalLogPath` or `$dbname`. *Mitigation*: Validate presence of common file before sourcing; version‑control both files together.
- **Debug flag left enabled in production** – `set -x` can flood logs and expose sensitive data. *Mitigation*: Enforce comment‑out of `setparameter` in production environments via CI lint rule.
- **Hard‑coded absolute paths** – any filesystem restructure breaks the job. *Mitigation*: Parameterize root directories in common properties; avoid absolute literals.
- **Hive refresh failures** – if tables are locked or corrupted, refresh commands may error, leaving downstream jobs with stale metadata. *Mitigation*: Add retry logic and explicit error handling in driver script.

# Usage
```bash
# Source configuration (usually done inside the driver script)
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
. /path/to/usage_overage_charge.properties

# Run driver with optional debug
set -x   # enable if setparameter is uncommented
./usage_overage_charge.sh   # driver script consumes the variables
```
To debug a single run:
```bash
setparameter='set -x'   # uncomment in the properties file
./usage_overage_charge.sh > /tmp/debug.log 2>&1
```

# Configuration
- **Environment variables** (populated by common properties):
  - `MNAASLocalLogPath` – base directory for logs.
  - `dbname` – Hive database name.
- **Referenced config files**:
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Local files created**:
  - `$MNAAS_usage_overage_charge_LogPath`
  - `$usage_overage_charge_ProcessStatusFileName`

# Improvements
1. **Externalize table names** – move `usage_overage_charge_temp_tblname` and `usage_overage_charge_tblname` into a central schema definition file to avoid duplication across jobs.
2. **Add validation block** – at the end of the properties file, insert a Bash function that checks required variables (e.g., `[[ -z $MNAASLocalLogPath ]] && echo "Missing log path" && exit 1`). This prevents silent misconfiguration.