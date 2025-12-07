**High‑Level Documentation – `move-mediation-scripts/bin/runBARep.sh`**

---

### 1. Purpose (One‑Paragraph Summary)
`runBARep.sh` is a thin wrapper script that launches the **BARepLoader** Java application. It loads the Billing Account (BA) report for a specific month‑year into the MNAAS data‑warehouse, using configuration values defined in the central `MNAAS_ShellScript.properties` file. The script is typically invoked by a cron job or manually by an operator to trigger the BA report ingestion pipeline.

---

### 2. Key Components

| Component | Type | Responsibility |
|-----------|------|----------------|
| `MNAAS_ShellScript.properties` | External properties file | Supplies environment variables (e.g., DB connection strings, Hadoop paths, log directories) used by the Java loader and other Move‑Mediation scripts. |
| `runBARep.sh` | Bash wrapper | Sources the properties file, then executes `BARepLoader.jar` with three arguments: **period**, **properties file path**, and **log file path**. |
| `BARepLoader.jar` | Java executable (located under `/app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars/`) | Performs the actual extraction, transformation, and loading (ETL) of the BA report data for the supplied period. Writes operational logs to the supplied log file and persists data to target tables / HDFS. |
| `MoveBARep.log` | Log file (`/app/hadoop_users/MNAAS/MNAASCronLogs/MoveBARep.log`) | Captures stdout/stderr from the Java process for audit and troubleshooting. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | 1. `MNAAS_ShellScript.properties` (sourced). <br>2. Hard‑coded period argument (`09-2019`). <br>3. Log file path (`/app/hadoop_users/MNAAS/MNAASCronLogs/MoveBARep.log`). |
| **Outputs** | 1. Log entries written to `MoveBARep.log`. <br>2. Data persisted by `BARepLoader.jar` (typically DB tables, Hive/Impala tables, or HDFS directories). |
| **Side Effects** | - Database writes (billing‑account tables). <br>- Potential HDFS/Hive updates. <br>- May trigger downstream processes that depend on the BA data (e.g., revenue reconciliation, reporting). |
| **Assumptions** | - Java 1.8+ is installed and on the PATH. <br>- The properties file exists and contains all required variables (e.g., DB URLs, credentials). <br>- The `BARepLoader.jar` version is compatible with the current schema. <br>- The executing user has read access to the properties file and write access to the log directory. |

---

### 4. Interaction with Other Scripts / Components

| Connected Script / Component | Relationship |
|------------------------------|--------------|
| `mnaas_seq_check_for_feed.sh` | May run **before** `runBARep.sh` to verify that the BA feed file is present in the inbound directory. |
| `mnaas_tbl_load.sh` / `mnaas_tbl_load_generic.sh` | Load generic tables; `runBARep.sh` focuses on the BA‑specific table set. Both share the same properties file and may be orchestrated sequentially in a nightly batch. |
| `move_table_compute_stats.sh` | Consumes the data loaded by `BARepLoader.jar` to compute aggregates and statistics after the BA report load completes. |
| Cron scheduler (`crontab`) | Typically schedules `runBARep.sh` (e.g., `0 2 * * * /path/runBARep.sh`). |
| Monitoring / Alerting system (e.g., Nagios, Splunk) | Watches `MoveBARep.log` for error patterns or non‑zero exit codes. |
| External data source (FTP/SFTP drop zone) | The BA report file that `BARepLoader.jar` processes is usually placed there by an upstream system; the wrapper does not handle the transfer itself. |

---

### 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Hard‑coded period (`09-2019`)** | Loads stale data; requires manual edit each month. | Parameterize the period (e.g., read from env var or compute `$(date +%m-%Y)` for previous month). |
| **Missing or outdated `MNAAS_ShellScript.properties`** | Loader may fail due to missing DB credentials or path settings. | Add a pre‑run validation step that checks required variables are defined; fail fast with a clear message. |
| **Jar not found / corrupted** | Job aborts, no data loaded. | Verify jar checksum at deployment; include a fallback to an alternate path; log explicit error if `java -jar` returns non‑zero. |
| **Log file growth** | Disk exhaustion on the log directory. | Implement log rotation (e.g., `logrotate` config) and size checks before execution. |
| **Insufficient permissions** | Script cannot read properties or write logs, causing silent failures. | Run the script under a dedicated service account with documented ACLs; test permissions during CI. |
| **Java process hangs** | Batch window missed, downstream jobs blocked. | Wrap the `java -jar` call with a timeout utility (e.g., `timeout 2h java -jar …`). |

---

### 6. Running & Debugging Guide

1. **Standard Execution**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_ShellScript/bin
   ./runBARep.sh
   ```

2. **Check Exit Status**  
   ```bash
   ./runBARep.sh
   echo $?   # 0 = success, non‑zero = failure
   ```

3. **View Runtime Log**  
   ```bash
   tail -f /app/hadoop_users/MNAAS/MNAASCronLogs/MoveBARep.log
   ```

4. **Debug Mode (trace Bash commands)**  
   ```bash
   set -x   # enable tracing
   ./runBARep.sh
   set +x   # disable tracing
   ```

5. **Validate Properties**  
   ```bash
   grep -E 'DB|HADOOP|LOG' /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties
   ```

6. **Force a Different Period (temporary)**  
   ```bash
   java -jar /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars/BARepLoader.jar 08-2024 \
        /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties \
        /app/hadoop_users/MNAAS/MNAASCronLogs/MoveBARep.log
   ```

---

### 7. External Configuration & Environment Variables

| Variable / File | Usage |
|-----------------|-------|
| `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties` | Sourced at the top of the script; defines paths (`BARepLoaderJarPath` – currently commented out), DB credentials, Hadoop configuration, and log directories. |
| `BARepLoaderJarPath` (commented) | Intended to hold the jar location; the script currently uses a hard‑coded absolute path. |
| `MNAASPropertiesPath` (if defined in properties) | Could be used to build the properties file argument; not used in the current hard‑coded call. |
| `BARepLoaderJavaLogPath` (if defined) | Intended log path; overridden by the explicit log argument. |

*If any of these variables are missing or changed, the script will still run because it uses absolute paths, but future maintainability depends on aligning the script with the property definitions.*

---

### 8. Suggested Improvements (TODO)

1. **Parameterize the Period Argument**  
   Replace the hard‑coded `09-2019` with a dynamic value (e.g., previous month) or accept a command‑line argument:  
   ```bash
   PERIOD=${1:-$(date -d "$(date +%Y-%m-01) -1 month" +"%m-%Y")}
   java -jar "$BARepLoaderJarPath" "$PERIOD" "$MNAASPropertiesPath/MNAAS_ShellScript.properties" "$BARepLoaderJavaLogPath"
   ```

2. **Add Robust Error Handling**  
   Capture the Java process exit code, log a concise success/failure message, and exit with the same code to allow upstream orchestration tools to react appropriately:
   ```bash
   java -jar ... 
   RC=$?
   if [ $RC -ne 0 ]; then
       echo "$(date) ERROR: BARepLoader failed with code $RC" >> "$BARepLoaderJavaLogPath"
   else
       echo "$(date) INFO: BARepLoader completed successfully" >> "$BARepLoaderJavaLogPath"
   fi
   exit $RC
   ```

Implementing these changes will make the script more maintainable, reduce manual intervention, and improve observability in production.