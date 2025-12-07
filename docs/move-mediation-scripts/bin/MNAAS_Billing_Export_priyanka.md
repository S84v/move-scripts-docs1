**File:** `move-mediation-scripts/bin/MNAAS_Billing_Export_priyanka.sh`

---

### 1. Summary
This shell wrapper executes the `BillingExport.jar` Java batch job for a **fixed August 2020** billing period in *daily* mode. It first loads a shared properties file (`MNAAS_Java_Batch.properties`) that defines environment‑specific variables (e.g., Hadoop paths, DB credentials, SFTP targets). The script is a lightweight, hard‑coded variant of the generic `MNAAS_Billing_Export.sh` used for ad‑hoc runs or testing.

---

### 2. Key Components & Responsibilities
| Component | Type | Responsibility |
|-----------|------|----------------|
| `MNAAS_Java_Batch.properties` | Config file (sourced) | Provides Java system properties, Hadoop classpaths, DB connection strings, and log locations used by the Java batch job. |
| `BillingExport.jar` | Java executable (batch) | Implements the actual billing export logic: extracts raw billing data, transforms it, writes output files, and possibly pushes them to downstream systems (e.g., SFTP, HDFS). |
| `Export.log` | Log file (written by Java) | Captures stdout/stderr of the Java process for audit and troubleshooting. |
| This shell script (`MNAAS_Billing_Export_priyanka.sh`) | Bash wrapper | Sources the properties, invokes the Java jar with hard‑coded arguments (date `2020-08`, mode `D` for daily), and directs logging. |

*No functions or classes are defined inside the script itself; its sole purpose is orchestration.*

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | • Fixed period argument: `2020-08` (hard‑coded). <br>• Mode flag: `D` (daily) – the script also contains a commented line for `M` (monthly). <br>• External properties file path: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Java_Batch.properties`. |
| **Outputs** | • Log file: `/app/hadoop_users/MNAAS/MNAASCronLogs/Export.log` (appended by the Java process). <br>• Any files produced by `BillingExport.jar` (location defined inside the properties file, typically an HDFS or SFTP target). |
| **Side‑Effects** | • Reads from source databases (likely Oracle/SQL Server) as configured in the properties file. <br>• Writes transformed billing files to downstream storage (HDFS, SFTP, or local FS). <br>• May trigger downstream downstream jobs that poll the output location. |
| **Assumptions** | • Java 1.8+ is installed and `java` is on the PATH. <br>• The properties file exists, is readable, and contains all required keys (e.g., `DB_URL`, `HADOOP_CONF_DIR`). <br>• The JAR file exists at the exact path and is executable. <br>• The user running the script has permission to write to the log directory and to access Hadoop/HDFS resources. <br>• No argument validation – the script trusts the hard‑coded values. |

---

### 4. Integration Points & Connectivity

| Connected Component | How the Connection Occurs |
|---------------------|---------------------------|
| **`MNAAS_Billing_Export.sh` (generic driver)** | The generic script likely builds the date and mode dynamically and then calls this or a similar wrapper. This file is a *static* copy used for a one‑off run; it does **not** receive parameters from other scripts. |
| **Hadoop / HDFS** | Configured via properties; the Java job uses Hadoop libraries to read/write data. |
| **Source Billing DB** | Connection details (JDBC URL, user, password) are supplied by the properties file. |
| **Downstream SFTP / FTP** | If the Java job pushes files externally, the target host/credentials are also defined in the properties file. |
| **Cron Scheduler** | In production, a cron entry (e.g., `MNAASCronLogs/Export.log`) may invoke the generic script; this ad‑hoc script is typically run manually. |
| **Monitoring / Alerting** | Logs written to `Export.log` are consumed by log‑aggregation tools (e.g., Splunk, ELK) for success/failure alerts. |

---

### 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Hard‑coded date (`2020-08`)** | Running the script after the intended period will re‑process stale data, causing duplicate exports. | Parameterize the date (e.g., read from `$1` or compute `$(date +%Y-%m)`). Add a sanity check that the date is not in the future. |
| **Missing/Corrupt Properties File** | Java job may fail silently or use default values, leading to data loss or security exposure. | Verify existence and readability of the properties file before invoking Java; exit with a clear error code if not found. |
| **No Error Handling / Exit Code Capture** | Failures are only visible in the log; the wrapper always returns success, confusing downstream orchestration. | Capture `$?` after the Java call, log success/failure, and exit with the same code. |
| **Log File Growth** | Unbounded appends to `Export.log` can fill disk. | Rotate logs (e.g., via `logrotate`) or truncate the file at start of each run. |
| **Dependency on Fixed Paths** | Changes in deployment layout break the script. | Use environment variables (e.g., `$MNAAS_HOME`) defined in the properties file to construct paths dynamically. |
| **Lack of Parallelism Control** | If multiple instances run concurrently, they may clash on output locations. | Implement a lock file or use a job scheduler that enforces single‑instance execution. |

---

### 6. Running & Debugging Guide

1. **Prerequisites**  
   - Ensure the user has read access to `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Java_Batch.properties`.  
   - Verify that `java -version` returns a compatible JRE.  
   - Confirm the JAR exists: `ls -l /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars/BillingExport.jar`.  

2. **Execute Manually**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars   # optional, just for context
   /path/to/move-mediation-scripts/bin/MNAAS_Billing_Export_priyanka.sh
   ```
   - The script will append output to `/app/hadoop_users/MNAAS/MNAASCronLogs/Export.log`.  

3. **Check Result**  
   - Inspect the log: `tail -f /app/hadoop_users/MNAAS/MNAASCronLogs/Export.log`.  
   - Look for a line containing `SUCCESS` or an explicit exit code from the Java program.  

4. **Debugging Tips**  
   - Add `set -x` at the top of the script to echo each command.  
   - Run the Java command directly to see full console output:  
     ```bash
     java -jar /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars/BillingExport.jar \
          2020-08 /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Java_Batch.properties \
          /app/hadoop_users/MNAAS/MNAASCronLogs/Export.log D
     ```  
   - Verify that the properties file contains the expected keys (`DB_URL`, `OUTPUT_PATH`, etc.) by grepping: `grep -E 'DB_URL|OUTPUT_PATH' /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Java_Batch.properties`.  

5. **Exit Code**  
   - After execution, run `echo $?` to see the script’s exit status (0 = success, non‑zero = failure).  

---

### 7. External Config / Environment Dependencies

| Item | Purpose | Typical Location |
|------|---------|------------------|
| `MNAAS_Java_Batch.properties` | Holds all runtime parameters for the Java batch (DB credentials, Hadoop config, output directories, SFTP targets). | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/` |
| `JAVA_HOME` (optional) | Determines which JRE is used. | Environment of the invoking user; not explicitly set in the script. |
| `PATH` (must include `java`) | Allows the script to invoke the Java binary. | System PATH. |
| `HADOOP_CONF_DIR` (often defined in properties) | Points to Hadoop configuration files needed by the Java job. | Inside properties or system env. |

If any of these are missing or mis‑configured, the Java job will fail early.

---

### 8. Suggested Improvements (TODO)

1. **Parameterize Date & Mode**  
   Replace the hard‑coded `2020-08` and `D` with command‑line arguments (e.g., `$1` for period, `$2` for mode) and add validation.  

2. **Add Robust Error Handling**  
   Capture the Java process exit code, log a concise success/failure message, and propagate the code to the caller. Also, implement a lock file to prevent concurrent executions.  

*(Implementing these two changes will make the script reusable, safer for production scheduling, and easier to troubleshoot.)*