**High‑Level Documentation – `move-mediation-scripts/bin/processNotifBackup.sh`**

---

### 1. Summary
`processNotifBackup.sh` is a thin wrapper that initiates the **Notification** data‑load job for the MNAAS mediation platform. It loads runtime properties, then executes the Java program `Notification.jar`, passing the same properties file and a dedicated log file. In production the script is typically scheduled (via cron) to process nightly notification backup files and insert the transformed data into downstream Hadoop/DB tables.

---

### 2. Important Components & Responsibilities
| Component | Responsibility |
|-----------|-----------------|
| **`processNotifBackup.sh`** (this file) | *Shell driver*: sources environment‑wide properties, launches the Java loader, and redirects output to a persistent log. |
| **`MNAAS_ShellScript.properties`** (sourced) | Central configuration: Hadoop paths, DB connection strings, S3/SFTP endpoints, authentication tokens, and any job‑specific flags used by the Java loader. |
| **`Notification.jar`** (Java application) | Core ETL engine for notification data: reads raw backup files (likely Parquet/CSV), performs validation & transformation, and writes results to target tables or HDFS locations. |
| **`MoveBNotification.log`** | Append‑only execution log capturing INFO/ERROR messages from the Java process; used for monitoring and troubleshooting. |

---

### 3. Inputs, Outputs, Side Effects & Assumptions
| Category | Details |
|----------|---------|
| **Inputs** | • `MNAAS_ShellScript.properties` (absolute path).<br>• Implicit input files referenced inside the JAR (e.g., notification backup files located under paths defined in the properties). |
| **Outputs** | • Processed data persisted by the JAR (HDFS tables, Hive partitions, or relational DB – defined in the properties).<br>• Log entries written to `/app/hadoop_users/MNAAS/MNAASCronLogs/MoveBNotification.log`. |
| **Side Effects** | • May create temporary staging directories on HDFS.<br>• May trigger downstream jobs (e.g., compute‑stats scripts) via Hive/Impala triggers defined in the JAR. |
| **Assumptions** | • Java 1.8+ is installed and `java` is on the PATH.<br>• The user executing the script has read access to the properties file and write access to the log directory.<br>• All external services referenced in the properties (HDFS, Hive, DB, SFTP) are reachable and credentials are valid.<br>• The JAR is built for the current schema version; no version mismatch. |

---

### 4. Integration with Other Scripts / Components
| Connected Script / Component | Relationship |
|------------------------------|--------------|
| **`processBalBackup.sh`**, **`processAsyncBackup.sh`** | Sibling backup‑processing jobs; typically run sequentially or in parallel by a master cron schedule. |
| **`move_table_compute_stats.sh`** | Consumes the tables populated by `Notification.jar` to generate statistics; may be triggered downstream after this script completes successfully. |
| **`mnaas_tbl_load*.sh`** family | Alternative loaders for other domains (e.g., usage, balance). The notification loader follows the same pattern (properties → JAR → log). |
| **Cron Scheduler** | `processNotifBackup.sh` is usually invoked from a crontab entry (e.g., `0 2 * * * /path/processNotifBackup.sh`). |
| **Monitoring / Alerting** | Log file is tailed by log‑aggregation tools (Splunk, ELK) and alerts are raised on ERROR patterns. |

---

### 5. Operational Risks & Recommended Mitigations
| Risk | Mitigation |
|------|------------|
| **Missing or corrupted properties file** | Validate existence & readability at script start; exit with clear error code. |
| **Java process OOM / failure** | Configure JVM options (e.g., `-Xmx`) via a wrapper or inside the JAR; monitor exit status and send alerts. |
| **Log file growth** | Implement log rotation (e.g., `logrotate` daily, keep 30 days) or size‑based rotation in the cron. |
| **Stale input data** | Add a pre‑check in the JAR (or wrapper) to verify that expected backup files exist and are newer than the last successful run. |
| **Permission changes on HDFS/DB** | Run a periodic “dry‑run” validation script that attempts a minimal write/read using the same credentials. |
| **Uncontrolled parallel runs** | Use a lock file (`flock`) around the script to prevent overlapping executions. |

---

### 6. Running & Debugging the Script
1. **Standard execution** (as scheduled):  
   ```bash
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/processNotifBackup.sh
   ```
2. **Manual run with verbose output** (helps debugging):  
   ```bash
   set -x   # enable shell tracing
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/processNotifBackup.sh
   set +x
   ```
3. **Check exit status**:  
   ```bash
   echo $?   # 0 = success, non‑zero = failure
   ```
4. **Inspect log** (real‑time):  
   ```bash
   tail -f /app/hadoop_users/MNAAS/MNAASCronLogs/MoveBNotification.log
   ```
5. **Validate properties**:  
   ```bash
   grep -E '^(HDFS|DB|NOTIF)' /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties
   ```
6. **Re‑run the Java JAR directly** (if deeper inspection needed):  
   ```bash
   java -jar /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars/Notification.jar \
        File /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties \
        /tmp/debug.log
   ```
   Review `debug.log` for stack traces.

---

### 7. External Configuration / Environment Variables
| File / Variable | Purpose |
|-----------------|---------|
| `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties` | Holds all runtime parameters (HDFS paths, DB URLs, credentials, feature flags). The script sources it, making its variables available to the Java process. |
| `JAVA_HOME` (implicitly) | Determines which Java runtime is used; must point to a compatible JDK/JRE. |
| `PATH` (implicitly) | Must include the `java` executable. |
| **Potential future env vars** (not currently used but common in this suite): `MNAAS_ENV`, `MNAAS_LOG_LEVEL`. If added, the script should forward them to the JAR via additional command‑line arguments. |

---

### 8. Suggested TODO / Improvements
1. **Add robust error handling** – capture the Java exit code, write a concise status line to a separate “status” file, and exit with a non‑zero code to allow the scheduler to trigger alerts.
2. **Introduce a lock mechanism** – wrap the execution with `flock -n /tmp/processNotifBackup.lock` to prevent concurrent runs that could corrupt target tables or overload the cluster.