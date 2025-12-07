**File:** `move-mediation-scripts/bin/MNAAS_Traffic_tbl_with_nodups_sms_only_loading.sh`

---

## 1. High‑Level Summary
This Bash script implements the daily “SMS‑only traffic” load for the MNAAS mediation pipeline. It orchestrates two Java‑based stages – an *intermediate* load that writes raw, de‑duplicated records into a temporary table, followed by a *final* load that moves the data into the production Impala table and refreshes the corresponding view. Execution status, PID tracking, and error handling (including SDP ticket creation via `mailx`) are persisted in a shared process‑status file that other mediation jobs read/write, enabling coordinated, idempotent runs across the whole mediation suite.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **`MNAAS_Traffic_tbl_with_nodups_sms_only_inter_table_loading`** | Updates status flag to **1**, checks that the intermediate Java job is not already running, launches the Java class that loads de‑duplicated SMS traffic into a staging table, logs success/failure, and aborts on error. |
| **`MNAAS_Traffic_tbl_with_nodups_sms_only_table_loading`** | Updates status flag to **2**, ensures the final Java job is not already running, launches the Java class that moves data from staging to the production Impala table, runs an `impala-shell` `REFRESH` on the target table, logs outcome, and aborts on error. |
| **`terminateCron_successful_completion`** | Resets the status file to “idle” (`flag=0`, `job_status=Success`), logs script termination, and exits with code 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, invokes `email_and_SDP_ticket_triggering_step`, and exits with code 1. |
| **`email_and_SDP_ticket_triggering_step`** | Marks the job as `Failure` in the status file, checks whether an SDP ticket has already been raised, and if not, sends a templated email to the support mailbox (via `mailx`) and flips the `email_sdp_created` flag. |
| **Main block** | Reads the previous PID from the status file, prevents concurrent runs, decides which loading steps to invoke based on the persisted `MNAAS_Daily_ProcessStatusFlag`, and finally calls the appropriate termination routine. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_Traffic_tbl_with_nodups_sms_only_loading.properties` – defines all runtime variables (log path, status‑file path, Java class names, JAR locations, Impala host, partition file, etc.). |
| **Environment variables** | `CLASSPATHVAR`, `IMPALAD_HOST`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` – expected to be exported in the shell or set in the properties file. |
| **External services** | • Java runtime (JAR `MNAAS_Main_JarPath`) <br>• Impala cluster (`impala-shell`) <br>• SMTP/mailx for SDP ticket email <br>• System logger (`logger`) |
| **Files written** | • Log file (`$MNAAS_Traffic_tbl_with_nodups_sms_only_loadingLogPath`) <br>• Process‑status file (`$MNAAS_Traffic_tbl_with_nodups_sms_only_loading_ProcessStatusFileName`) – holds flags, PID, job status, email flag, etc. |
| **Tables affected** | • Temporary staging table (populated by the first Java job) <br>• Production Impala table `${traffic_details_raw_daily_with_no_dups_sms_only_tblname}` – refreshed after the second Java job. |
| **Assumptions** | • The status file exists and is writable. <br>• Java class names and JAR are correct and compatible with the current schema. <br>• Impala host is reachable and the user has `REFRESH` privileges. <br>• `mailx` is configured to send external mail. |
| **Side effects** | • Updates a shared status file used by other mediation scripts (e.g., `MNAAS_TrafficDetails_tbl_Load_new.sh`). <br>• May trigger an SDP ticket on failure, which downstream support processes consume. |

---

## 4. Interaction with Other Scripts / Components

| Interaction | Description |
|-------------|-------------|
| **Process‑status file** | Shared with all MNAAS daily jobs (e.g., `MNAAS_TrafficDetails_tbl_Load_new.sh`). The flag values (`0`, `1`, `2`) indicate the current stage of the SMS‑only load, allowing downstream scripts to wait or skip execution. |
| **Java loader classes** | The same JAR (`$MNAAS_Main_JarPath`) is used by multiple scripts; class names (`$MNAAS_Traffic_tbl_with_nodups_sms_only_inter_loading_load_classname`, `$MNAAS_Traffic_tbl_with_nodups_sms_only_loading_load_classname`) are defined in the properties file. |
| **Impala refresh** | After the final load, the script runs `impala-shell -i $IMPALAD_HOST -q "<refresh‑stmt>"`. Other scripts that query the refreshed table rely on this step completing successfully. |
| **Cron scheduler** | Typically invoked by a daily cron entry; the script itself guards against overlapping runs using the PID stored in the status file. |
| **SDP ticketing** | Failure handling sends an email to `insdp@tatacommunications.com`. The ticketing system may be monitored by the same team that maintains other mediation jobs. |
| **Logging** | All scripts write to a common log directory (path defined in the properties file), enabling centralized log aggregation and alerting. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** – stale PID or missing status file could allow two instances to run simultaneously, causing duplicate loads. | Data corruption / duplicate rows. | Ensure the status file is created atomically at deployment; add a lockfile (`flock`) as an extra safeguard. |
| **Java job failure** – non‑zero exit code not captured (e.g., OOM, classpath issues). | Incomplete data load, downstream jobs stall. | Verify Java exit codes, add memory limits, and monitor Java logs separately. |
| **Impala refresh failure** – network glitch or permission issue. | Table not refreshed, downstream analytics see stale data. | Capture `impala-shell` output, retry on transient errors, and alert on non‑zero exit status. |
| **Email/SDP ticket not sent** – `mailx` misconfiguration. | Failure not escalated, SLA breach. | Add a fallback to write a ticket file locally; monitor the `email_sdp_created` flag. |
| **Properties file drift** – missing or malformed variable leads to script crash. | Immediate job abort, no data load. | Validate required variables at script start; fail fast with clear messages. |
| **Log file growth** – unbounded log size over time. | Disk exhaustion. | Rotate logs via `logrotate` and enforce size limits. |

---

## 6. Running & Debugging the Script

### Normal Execution (Cron)

```bash
# Ensure the properties file is present and readable
# Cron entry (example, runs at 02:30 daily)
30 2 * * * /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Traffic_tbl_with_nodups_sms_only_loading.properties && \
    /path/to/MNAAS_Traffic_tbl_with_nodups_sms_only_loading.sh >> /dev/null 2>&1
```

### Manual Run (for testing)

```bash
# Export any missing env vars (if not already in the properties file)
export CLASSPATHVAR=/opt/java/lib/* 
export IMPALAD_HOST=impala01.tatacommunications.com
export SDP_ticket_from_email=cloudera.support@tatacommunications.com
export MOVE_DEV_TEAM=devteam@tatacommunications.com

# Run the script interactively
bash -x /app/hadoop_users/MNAAS/MNAAS_Traffic_tbl_with_nodups_sms_only_loading.sh
```

### Debug Checklist

1. **Check status file** – `cat $MNAAS_Traffic_tbl_with_nodups_sms_only_loading_ProcessStatusFileName`  
   Verify `MNAAS_Daily_ProcessStatusFlag` and `MNAAS_Script_Process_Id`.
2. **Inspect logs** – `tail -f $MNAAS_Traffic_tbl_with_nodups_sms_only_loadingLogPath`.
3. **Validate Java class** – Run the Java command manually with `-verbose:class` to ensure the class loads.
4. **Test Impala refresh** – `impala-shell -i $IMPALAD_HOST -q "SHOW TABLES;"` and then the refresh statement.
5. **Force failure** – Temporarily change a variable (e.g., wrong JAR path) to confirm the failure path creates an SDP ticket.
6. **PID check** – `ps -ef | grep $Dname_MNAAS_Load_Daily_Traffic_tbl_with_nodups_sms_only` should show only the current process.

---

## 7. External Configuration & Environment Dependencies

| Item | Source | Usage |
|------|--------|-------|
| `MNAAS_Traffic_tbl_with_nodups_sms_only_loading.properties` | Sourced at top of script | Provides all `$MNAAS_*` variables (log path, status file, class names, JAR path, partition file, etc.). |
| `CLASSPATHVAR` | Exported in environment or properties file | Extends Java classpath for the loader JARs. |
| `IMPALAD_HOST` | Exported or in properties file | Target Impala daemon for the `REFRESH` command. |
| `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Exported or in properties file | Email “From” address and CC list for SDP ticket notifications. |
| `mailx` | System binary | Sends failure notification email. |
| `logger` | System binary | Writes to syslog and the script‑specific log file. |
| `java` | System binary | Executes the loader classes. |
| `impala-shell` | System binary | Refreshes the target Impala table. |

---

## 8. Suggested Improvements (TODO)

1. **Add a lockfile (`flock`)** around the entire main block to guarantee single‑instance execution even if the status file becomes corrupted or the PID is stale.
2. **Centralize exit‑code handling**: wrap Java and Impala commands in a helper function that logs stdout/stderr, retries on transient failures, and returns a uniform status code for the termination routines.

---