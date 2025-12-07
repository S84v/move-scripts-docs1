**File:** `move-mediation-scripts/bin/MNAAS_Billing_Export_02-Nov-20.sh`

---

### 1. High‑Level Summary
This shell script is a one‑off driver that launches the `BillingExport.jar` Java batch job for the October 2020 billing period. It first loads a shared Java‑batch properties file, then executes the jar in *daily* mode (`D`). A commented line shows how the same jar could be run in *monthly* mode (`M`). The script is intended to be invoked by a cron entry or a higher‑level driver (e.g., `MNAAS_Billing_Export.sh`) as part of the nightly MNAAS billing export pipeline.

---

### 2. Key Components & Responsibilities  

| Component | Type | Responsibility |
|-----------|------|----------------|
| `MNAAS_Java_Batch.properties` | External properties file | Supplies environment‑specific configuration (DB URLs, Hadoop paths, credentials, log levels, etc.) used by the Java batch job. |
| `BillingExport.jar` | Java executable (contains `com.mnaas.billing.ExportMain` or similar) | Implements the actual extraction, transformation, and loading of billing data for a given period. Accepts four CLI arguments: `<period> <propertiesFile> <logFile> <mode>`. |
| `Export.log` | Log file (written by the Java process) | Captures stdout/stderr of the Java job for audit and troubleshooting. |
| This shell script (`MNAAS_Billing_Export_02-Nov-20.sh`) | Bash driver | Sources the shared properties, builds the command line, and launches the Java batch job in daily mode. |

*No functions or classes are defined inside the script itself; it is a linear driver.*

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | 1. Hard‑coded period argument: `2020-10` (October 2020).<br>2. Path to the shared properties file: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Java_Batch.properties`.<br>3. Path to the log file: `/app/hadoop_users/MNAAS/MNAASCronLogs/Export.log`.<br>4. Mode flag: `D` (daily). |
| **Outputs** | - The Java job writes its own output files (likely CSV/Parquet) to HDFS or a staging directory defined in the properties file.<br>- Execution details are appended to `Export.log`. |
| **Side‑Effects** | - Consumes Hadoop resources (YARN containers, HDFS I/O).<br>- May open DB connections (source/target) as defined in the properties file.<br>- May trigger downstream processes that watch the export directory (e.g., reconciliation dashboards). |
| **Assumptions** | - The script runs under the `MNAAS` Hadoop user with appropriate OS and Hadoop permissions.<br>- Java 1.8+ is installed and `java` is on the PATH.<br>- `BillingExport.jar` is compatible with the supplied properties file version.<br>- The properties file contains all required keys; missing keys will cause the Java job to fail.<br>- The log directory exists and is writable. |

---

### 4. Integration Points  

| Connected Component | How the Connection Occurs |
|---------------------|---------------------------|
| **`MNAAS_Billing_Export.sh`** (parent driver) | Likely calls this script (or a similar dated variant) from a master cron schedule. |
| **Cron Scheduler** | A nightly cron entry (e.g., `0 2 * * * /app/hadoop_users/MNAAS/.../MNAAS_Billing_Export_02-Nov-20.sh`) triggers the script. |
| **Downstream Reconciliation Scripts** (e.g., `End_to_End_MOVENL_Mediation_Dashboard.sh`) | Consume the exported billing files produced by `BillingExport.jar` for validation and reporting. |
| **Hadoop/YARN** | The Java jar submits a MapReduce or Spark job; YARN allocates containers. |
| **Source/Target Databases** | Connection details are read from the properties file; the Java job may read from the Mediation DB and write to the Billing DB or a data lake. |
| **Monitoring/Alerting** | Log file (`Export.log`) is typically tailed by a monitoring system (e.g., Splunk, ELK) to raise alerts on failures. |

---

### 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Hard‑coded period (`2020-10`)** – script must be edited each month. | Human error → missed runs or wrong data. | Parameterize the period (e.g., `$1` argument) and use a wrapper that injects the current month. |
| **No error handling** – script always returns success even if Java exits with non‑zero code. | Silent failures, downstream jobs consume incomplete data. | Capture `$?` after the `java -jar` call; exit with that code and write an explicit error entry to the log. |
| **Log file growth** – `Export.log` is never rotated. | Disk exhaustion on the Hadoop node. | Configure logrotate or append a timestamp to the log filename per run. |
| **Dependency on absolute paths** – changes in deployment layout break the script. | Deployment failures. | Use environment variables (e.g., `$MNAAS_HOME`) defined in a central config file. |
| **Commented monthly mode** – operator may forget to switch mode. | Incorrect export granularity. | Add a command‑line flag (`-m` for monthly) that selects the mode automatically. |
| **Missing properties file or jar** – script will fail early. | Immediate job failure. | Pre‑flight checks: `[ -f "$PROPS" ] && [ -f "$JAR" ]` before execution, with clear error messages. |

---

### 6. Typical Execution & Debugging Steps  

1. **Run Manually**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_Billing_Export_02-Nov-20.sh
   ```
2. **Verify Log**  
   ```bash
   tail -f /app/hadoop_users/MNAAS/MNAASCronLogs/Export.log
   ```
   Look for a line containing `INFO`/`ERROR` and the final exit status.  
3. **Check Hadoop/YARN**  
   ```bash
   yarn application -list | grep BillingExport
   ```
   Ensure the job started and completed successfully.  
4. **Validate Output Files** (path defined in properties, e.g., `/data/billing/export/2020-10/`)  
   ```bash
   hdfs dfs -ls /data/billing/export/2020-10/
   ```
5. **Debugging**  
   - Add `set -x` at the top of the script to echo each command.  
   - If the Java job fails, run it directly with `java -jar ...` and add `-Dlog4j.debug` (if supported) to increase verbosity.  
   - Confirm that the properties file contains correct DB credentials and HDFS paths.  

---

### 7. External Configuration & Environment Variables  

| Item | Usage |
|------|-------|
| `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Java_Batch.properties` | Sourced by the script; also passed as an argument to the Java jar. Contains DB URLs, Hadoop namenode addresses, credential aliases, and job‑specific parameters. |
| `JAVA_HOME` (implicit) | Determines which Java runtime is used by `java -jar`. |
| `PATH` (implicit) | Must include the `java` executable. |
| Potential env vars (not shown but common) | `MNAAS_HOME`, `HADOOP_CONF_DIR`, `YARN_CONF_DIR` – may be referenced inside the properties file. |

---

### 8. Suggested Improvements (TODO)

1. **Parameterize the period and mode** – modify the script to accept command‑line arguments (`$1` = period, `$2` = mode) and default to the current month if omitted.  
2. **Add robust error handling** – capture the Java process exit code, log a clear success/failure message, and exit the script with the same status to allow upstream cron monitoring to detect failures.  

---