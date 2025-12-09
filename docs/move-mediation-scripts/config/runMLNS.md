# Summary
`runMLNS.properties` is a lightweight configuration wrapper used by the Move mediation pipeline to inject environment‑specific settings for the **MLNS** (Mobile LAN Statistics) loader job. It sources the global `MNAAS_CommonProperties.properties` file and defines the paths to the job‑specific properties file, executable JAR, and log file. The script that includes this file reads the variables to launch the `MLNS.jar` Hadoop job with the appropriate configuration and logging.

# Key Components
- **Source directive** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  *Loads shared Hadoop, credential, and logging settings for all Move jobs.*
- `runMLNS_properties_file` – Path to the job‑specific properties file (`runMLNS.properties`).  
  *Allows the loader to read additional parameters (e.g., date range, source tables).*
- `runMLNS_jar_file` – Absolute path to the executable JAR (`MLNS.jar`).  
  *Contains the MapReduce/ Spark driver that processes LAN statistics.*
- `runMLNS_log_file` – Destination for stdout/stderr of the job (`MoveMLNS.log`).  
  *Centralized log for operational monitoring and troubleshooting.*

# Data Flow
| Stage | Input | Process | Output | Side Effects |
|-------|-------|---------|--------|--------------|
| 1. Sourcing | `MNAAS_CommonProperties.properties` | Bash `.` sources environment variables (HADOOP_CONF_DIR, JAVA_HOME, credential tokens). | Environment variables available to the invoking script. | None |
| 2. Job launch | `runMLNS_properties_file` (optional key‑value pairs), `runMLNS_jar_file` | Invocation of `hadoop jar $runMLNS_jar_file -conf $runMLNS_properties_file …` (or Spark submit). | Processed LAN statistics written to HDFS/ Hive tables. | HDFS writes, Hive metastore updates. |
| 3. Logging | Job stdout/stderr | Redirected to `runMLNS_log_file`. | Log file persisted under `/app/hadoop_users/MNAAS/MNAASCronLogs/`. | Disk consumption; log rotation required. |

# Integrations
- **MNAAS_CommonProperties.properties** – Provides cluster configuration, Kerberos principals, and common log locations.
- **MLNS.jar** – Built from the `move-mediation-scripts/src/main/java/com/mnaas/mlns/` codebase; registers as a Sqoop/Hadoop job.
- **Cron / Scheduler** – Typically invoked by a cron entry (`MoveMLNS.sh`) that sources this file before executing the loader.
- **Hive / HDFS** – Target data stores for the processed LAN statistics; accessed via the job’s internal JDBC/HDFS APIs.
- **Credential Store** – May reference keytab files defined in the common properties for secure Hadoop access.

# Operational Risks
- **Path drift** – Hard‑coded absolute paths can become stale after filesystem re‑layout. *Mitigation*: Use symlinks or environment variables defined in the common properties.
- **Missing source file** – If `MNAAS_CommonProperties.properties` is unavailable, the job inherits no Hadoop configuration, causing submission failures. *Mitigation*: Add existence check and fail‑fast logic in the wrapper script.
- **Log file growth** – Unbounded log accumulation may fill the log partition. *Mitigation*: Implement log rotation (logrotate) and retention policies.
- **Version mismatch** – Updating `MLNS.jar` without updating corresponding property schema may cause runtime errors. *Mitigation*: Enforce version tagging and CI validation.

# Usage
```bash
# Example wrapper script snippet
#!/bin/bash
set -euo pipefail

# Load shared and job‑specific properties
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/runMLNS.properties

# Optional: verify required files exist
[[ -f "$runMLNS_jar_file" ]] || { echo "JAR not found"; exit 1; }

# Execute the loader
hadoop jar "$runMLNS_jar_file" \
    -D mapreduce.job.name=MLNS_Load \
    -conf "$runMLNS_properties_file" \
    >"$runMLNS_log_file" 2>&1
```
To debug, run the script with `bash -x` to trace variable expansion and verify that the sourced common properties are applied.

# Configuration
- **Environment variables** (inherited from `MNAAS_CommonProperties.properties`):  
  `HADOOP_CONF_DIR`, `JAVA_HOME`, `HADOOP_USER_NAME`, `KRB5CCNAME`, etc.
- **Referenced files**:  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  - `/app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars/MLNS.jar`  
  - `/app/hadoop_users/MNAAS/MNAASCronLogs/MoveMLNS.log`  
  - Optional job‑specific properties file (`runMLNS.properties`) for dynamic parameters.

# Improvements
1. **Parameterize paths** – Replace hard‑coded absolute paths with variables defined in `MNAAS_CommonProperties.properties` (e.g., `MNAAS_JAR_DIR`, `MNAAS_LOG_DIR`) to simplify environment migrations.
2. **Add validation block** – Include a Bash function that checks existence and readability of all referenced files and exits with a clear error code before job submission. This reduces silent failures in production.