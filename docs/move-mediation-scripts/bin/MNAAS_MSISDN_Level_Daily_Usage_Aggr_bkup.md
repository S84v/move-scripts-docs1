**File:** `move-mediation-scripts/bin/MNAAS_MSISDN_Level_Daily_Usage_Aggr_bkup.sh`

---

## 1. High‑Level Summary
This Bash script orchestrates the daily load and aggregation of MSISDN‑level usage data into a Hive/Impala table. It first truncates the target table, runs a Java aggregation job, then applies a retention policy that drops old partitions. The script maintains a lightweight process‑status file to coordinate with other daily jobs, logs all actions, and on failure sends an email and creates an SDP ticket. It is the “backup” variant of the primary `MNAAS_MSISDN_Level_Daily_Usage_Aggr.sh` and is typically invoked by a cron schedule when the primary script is unavailable or as a safety net.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **MNAAS_insert_into_aggr_table** | * Truncates the Hive aggregation table.<br>* Launches the Java class `$Insert_Daily_Aggr_table` to populate the table.<br>* Refreshes the Impala metadata on success.<br>* Updates the process‑status file to flag “insert” stage.<br>* Handles duplicate‑run detection and error logging. |
| **MNAAS_retention_for_msisdn_level_daily_usage_aggr_daily** | * Executes the generic Java retention JAR (`$DropNthOlderPartition`) to drop partitions older than the configured retention period.<br>* Refreshes or invalidates Impala metadata as needed.<br>* Updates the process‑status file to flag “retention” stage. |
| **terminateCron_successful_completion** | * Resets the status file flags to “Success”.<br>* Writes final timestamps and logs a clean exit. |
| **terminateCron_Unsuccessful_completion** | * Records failure timestamp, logs the error, and invokes the email/SDP ticket routine before exiting with a non‑zero status. |
| **email_and_SDP_ticket_triggering_step** | * Sets the job status to “Failure” in the status file.<br>* Sends a pre‑formatted email to the configured distribution list (via `mail`).<br>* Ensures a ticket is created only once per run. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_MSISDN_Level_Daily_Usage_Aggr.properties` – defines all environment variables used (log paths, DB/table names, jar locations, class names, retention period, mail recipients, etc.). |
| **Process‑Status File** | `$MNAAS_MSISDN_level_daily_usage_aggr_ProcessStatusFileName` – a simple key‑value file tracking PID, flags, timestamps, email‑sent flag, etc. Updated throughout the script. |
| **External Services** | - **Hive** (`hive -S -e "TRUNCATE TABLE …"`).<br>- **Impala** (`impala-shell`).<br>- **Java** (aggregation and retention JARs).<br>- **Mail** (`mail` command).<br>- **System logger** (`logger`). |
| **Outputs** | - Populated Hive/Impala table `${dbname}.${msisdn_level_daily_usage_aggr_tblname}`.<br>- Log file at `$MNAAS_MSISDN_level_daily_usage_aggrLogPath` (stderr redirected).<br>- Updated status file.<br>- Email notification on failure. |
| **Assumptions** | - All variables referenced in the properties file are defined and point to reachable resources.<br>- Hive, Impala, and Java runtimes are installed and accessible to the user running the script.<br>- The `mail` command is correctly configured (SMTP relay, recipient list).<br>- No other instance of the same Java process is running (checked via `ps`). |
| **Side Effects** | - Truncates the target Hive table (data loss if run out‑of‑order).<br>- Drops old partitions (irreversible).<br>- Sends external email and may trigger an SDP ticket. |

---

## 4. Interaction with Other Scripts / Components

| Connected Component | How the Connection Occurs |
|---------------------|----------------------------|
| **MNAAS_MSISDN_Level_Daily_Usage_Aggr.sh** (primary script) | Shares the same properties file and status file. The backup script reads the same flag values, ensuring only one of the two runs at a time. |
| **Cron Scheduler** | Typically invoked via a daily cron entry (e.g., `0 2 * * * /app/hadoop_users/.../MNAAS_MSISDN_Level_Daily_Usage_Aggr_bkup.sh`). |
| **Retention Scripts** (`MNAAS_*_Retention_*.sh`) | May use the same generic Java retention JAR (`$DropNthOlderPartition`). |
| **Monitoring / Alerting** | The status file is read by external health‑check scripts to surface job health in dashboards. |
| **SDP Ticketing System** | Email sent by `email_and_SDP_ticket_triggering_step` is routed to an automated ticketing pipeline (outside the script). |
| **Other Daily Load Scripts** (e.g., `MNAAS_GBS_Load.sh`, `MNAAS_IMEIChange_tbl_Load.sh`) | Run earlier in the same nightly window; they may populate source tables that this aggregation script consumes. Coordination is achieved via the shared status file and/or orchestrator (e.g., Oozie/Airflow) that enforces ordering. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Duplicate execution** (PID file stale, script thinks previous run finished) | Data truncation & double load → corrupted aggregates | Implement a lock file with a timeout; verify Hive table row count after load; add a health‑check that aborts if table already contains today’s data. |
| **Java job failure** (non‑zero exit) | No data loaded, downstream reports missing | Capture Java stdout/stderr to a separate log; add retry logic with exponential back‑off; alert on first failure. |
| **Impala metadata refresh failure** | Queries see stale data | After refresh, run a simple `SELECT COUNT(*)` to verify visibility; if fails, run `INVALIDATE METADATA` automatically. |
| **Retention job dropping wrong partitions** | Unexpected data loss | Validate the partition list before dropping; log the partitions that will be removed; keep a backup of the table for 24 h. |
| **Email/SDP spam** (multiple failures in short period) | Alert fatigue | Throttle email notifications (e.g., only first failure per day) and ensure the “email_sent” flag works correctly. |
| **Missing/incorrect properties** | Script aborts early, unclear error | Add a pre‑flight check that verifies all required variables are set and files exist; fail fast with a clear message. |
| **Insufficient disk space on HDFS** (during truncate/load) | Job stalls or fails | Monitor HDFS usage; add a pre‑run check; configure alerts for low space. |

---

## 6. Running / Debugging the Script

### Normal Execution (Cron)

```bash
# Example cron entry (run at 02:15 daily)
15 2 * * * /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_MSISDN_Level_Daily_Usage_Aggr_bkup.sh >> /dev/null 2>&1
```

### Manual Run (for testing)

```bash
# Switch to the Hadoop user that owns the scripts
sudo -u hadoop_user bash -x /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_MSISDN_Level_Daily_Usage_Aggr_bkup.sh
```

* `-x` enables shell tracing (already set via `set -x` in the script) and prints each command to stdout, useful for debugging.

### Common Debug Steps

1. **Verify property file** – `cat $MNAAS_MSISDN_Level_Daily_Usage_Aggr_ScriptName` to ensure all variables resolve.
2. **Check status file** – `grep -E 'MNAAS_.*' $MNAAS_MSISDN_level_daily_usage_aggr_ProcessStatusFileName`.
3. **Confirm no stray Java processes** – `ps -ef | grep $Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl`.
4. **Run Java class manually** (if needed) to capture detailed stack traces:
   ```bash
   java -Dname=$Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl \
        -cp $CLASSPATHVAR:$MNAAS_Main_JarPath \
        $Insert_Daily_Aggr_table $MNAAS_Property_filename \
        $MNAAS_MSISDN_level_daily_usage_aggrCntrlFileName \
        $processname_msisdn_level_daily_usage_aggr null \
        $MNAAS_MSISDN_level_daily_usage_aggrLogPath
   ```
5. **Validate Impala refresh** – after script finishes, run:
   ```bash
   impala-shell -i $IMPALAD_HOST -q "SELECT COUNT(*) FROM ${dbname}.${msisdn_level_daily_usage_aggr_tblname} LIMIT 1;"
   ```

---

## 7. External Configuration & Environment Variables

| Variable (populated by properties file) | Purpose |
|------------------------------------------|---------|
| `MNAAS_MSISDN_level_daily_usage_aggrLogPath` | Path to the script’s log file (stderr redirected). |
| `MNAAS_MSISDN_level_daily_usage_aggr_ProcessStatusFileName` | Shared status/lock file. |
| `dbname` | Hive database name. |
| `msisdn_level_daily_usage_aggr_tblname` | Target Hive/Impala table. |
| `Insert_Daily_Aggr_table` | Fully‑qualified Java class name for the aggregation job. |
| `MNAAS_Main_JarPath` / `Generic_Jar_Names` | Locations of the JAR files containing the Java classes. |
| `Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl` | Identifier used for process‑duplication checks (`ps`). |
| `DropNthOlderPartition` | Java class that implements partition retention. |
| `mnaas_retention_period_MSISDN_level_daily_usage_aggr` | Number of days to retain. |
| `IMPALAD_HOST`, `HIVE_HOST`, `HIVE_JDBC_PORT` | Connection endpoints for Impala/Hive. |
| `MailId`, `ccList` | Email recipients for failure notifications. |
| `MNAAS_MSISDN_Level_Daily_Usage_Aggr_ScriptName` | Human‑readable script identifier used in logs/emails. |
| `CLASSPATHVAR` | Additional classpath entries required by the Java jobs. |

*All of the above are defined in `MNAAS_MSISDN_Level_Daily_Usage_Aggr.properties`. If any are missing, the script will abort with a “command not found” or “null pointer” error.*

---

## 8. Suggested Improvements (TODO)

1. **Lock‑File with TTL** – Replace the ad‑hoc PID check with a robust lock file that expires after a configurable timeout to avoid dead‑locks caused by orphaned PID entries.
2. **Centralised Logging Framework** – Pipe all `logger` and `echo` output to a structured logging system (e.g., Logstash) with JSON fields for `job`, `status`, `timestamp`, enabling easier monitoring and alerting.

---