**MNAAS_Sim_Status_Monthly.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)

`MNAAS_Sim_Status_Monthly.sh` is the nightly orchestration script that populates two Impala tables with monthly SIM‑status aggregates: the **SIM‑Status Monthly** table and the **SIM‑Inventory Status Monthly** table. It reads a property file for configuration, determines the appropriate month partition(s) based on the current calendar day, invokes Java “loader” classes (packaged in the main MNAAS JAR) to write the data, refreshes the Impala metadata, and maintains a shared process‑status file that other MNAAS jobs use to coordinate execution. The script also handles success/failure logging, email/SDP ticket generation on error, and prevents concurrent runs.

---

### 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|-----------------|
| **MNAAS_insert_into_table** | - Updates process‑status flag to *1* (first step). <br>- Determines month partition (current month or previous month if day ≤ 6). <br>- Calls Java class `$Insert_Monthly_Status_table` to load the SIM‑Status Monthly table. <br>- On success, runs `impala-shell` to refresh the table metadata; on failure, triggers termination. |
| **MNAAS_insert_into_sim_inventory_status_monthly_table** | - Updates process‑status flag to *2* (second step). <br>- Determines month partition (previous month only on day = 1, otherwise current month). <br>- Calls Java class `$Insert_sim_inventory_Monthly_Status_table` to load the SIM‑Inventory Status Monthly table. <br>- Refreshes Impala metadata on success; otherwise terminates. |
| **terminateCron_successful_completion** | - Resets process‑status flag to *0* (idle) and marks job status **Success** in the shared status file. <br>- Writes final log entries and exits with code 0. |
| **terminateCron_Unsuccessful_completion** | - Updates run‑time stamp, logs failure, invokes `email_and_SDP_ticket_triggering_step`, and exits with code 1. |
| **email_and_SDP_ticket_triggering_step** | - Sets job status **Failure** in the status file. <br>- Sends a single alert email (and records that an SDP ticket was created) unless an email has already been sent for this run. |
| **Main script block** | - Checks the shared status file for a previous PID; if none is running, writes its own PID. <br>- Reads the daily process flag and decides which loader(s) to invoke (both steps, or only the second step). <br>- Handles the “already running” case by logging and exiting. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (external file)** | `MNAAS_Sim_Status_Monthly.properties` – defines all variables referenced in the script (paths, jar names, class names, Impala host, email recipients, etc.). |
| **Environment variables** | None required beyond those set by the properties file; the script sources the file directly. |
| **Process‑status file** | `$MNAAS_Sim_Status_Monthly_ProcessStatusFileName` – a plain‑text key/value file used by many MNAAS jobs to coordinate execution and report status. The script reads and writes several keys (`MNAAS_Daily_ProcessStatusFlag`, `MNAAS_Daily_ProcessName`, `MNAAS_job_status`, `MNAAS_job_ran_time`, `MNAAS_email_sdp_created`, `MNAAS_Script_Process_Id`). |
| **Log file** | `$MNAAS_Sim_Status_MonthlyLogPath` – appended with all `logger` output and script `set -x` trace. |
| **Java loader classes** | `$Insert_Monthly_Status_table` and `$Insert_sim_inventory_Monthly_Status_table` – executed via `java -cp …`. They read the property file and the partition file to generate the data. |
| **Impala metadata refresh** | Two SQL statements (`$sim_status_monthly_tblname_refresh`, `$sim_inventory_status_monthly_tblname_refresh`) executed with `impala-shell`. |
| **Email/SDP ticket** | Sent via the system `mail` command to `$MailId` (CC list `${ccList}`) on failure, only once per run. |
| **Side effects** | - Inserts/overwrites data in the two monthly Impala tables. <br>- Updates the shared status file (affects downstream jobs). <br>- Generates a log file and possibly an email/SDP ticket. |
| **Assumptions** | - The properties file exists and contains valid values for all referenced variables. <br>- Java runtime, the MNAAS JAR, and Impala client are installed and reachable. <br>- The process‑status file is writable by the script user. <br>- No other job will manually edit the status file while this script runs. |

---

### 4. Integration Points (how it connects to other scripts/components)

| Component | Connection Detail |
|-----------|-------------------|
| **MNAAS_SimInventory_tbl_Load / MNAAS_SimInventory_seq_check** | Those scripts populate the raw SIM‑Inventory tables that the monthly loaders later aggregate. They run earlier in the nightly window. |
| **MNAAS_Sim_Status_Daily.sh (hypothetical)** | The daily status script sets `MNAAS_Daily_ProcessStatusFlag` to *0* after successful load; this monthly script reads that flag to decide whether to run both steps or only the inventory step. |
| **Shared Process‑Status File** | All MNAAS jobs (daily, weekly, monthly) read/write the same `$MNAAS_Sim_Status_Monthly_ProcessStatusFileName`. The flag values (0, 1, 2) orchestrate the sequence of loads across scripts. |
| **Impala Metastore** | The `impala-shell` refresh commands make the newly loaded partitions visible to downstream reporting jobs (e.g., RAReports, PreValidation). |
| **Alerting System** | Email sent by `mail` may be captured by the organization’s ticketing system (SDP) for escalation. |
| **Cron Scheduler** | Typically invoked by a nightly cron entry (e.g., `0 2 * * * /path/MNAAS_Sim_Status_Monthly.sh`). The script’s PID check prevents overlapping cron runs. |
| **MNAAS_Main_JarPath** | Shared JAR containing all loader classes; other scripts also reference this path. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** (PID check fails due to stale PID) | Duplicate loads, data corruption, table lock contention | Ensure the PID is cleared on abnormal termination (e.g., trap `EXIT`); add a timeout check that removes stale PID after a configurable period. |
| **Incorrect month partition** (date logic edge‑cases) | Data loaded into wrong partition, downstream reports inaccurate | Add unit tests for the date‑calculation logic; log the selected partition(s) explicitly; consider using `date -d "$(date +%Y-%m-15) -1 month"` for robust previous‑month calculation. |
| **Missing/invalid properties** | Script aborts early, no data loaded | Validate required variables after sourcing the properties file; fail fast with a clear error message. |
| **Java loader failure** (non‑zero exit) | No data loaded, job marked failure | Capture Java stdout/stderr to a separate log; implement retry logic for transient failures (e.g., network hiccups). |
| **Impala refresh failure** | New partitions invisible to downstream jobs | Verify `impala-shell` return code; optionally retry or fall back to `invalidate metadata`. |
| **Email flooding** (multiple failures in same window) | Spam, ticketing overload | The script already guards with `MNAAS_email_sdp_created`; ensure the flag file is cleared only after a successful run or manual reset. |
| **Log file growth** | Disk exhaustion over time | Rotate logs via `logrotate`; keep only recent N days. |
| **Hard‑coded paths** | Breakage after environment changes | Externalize all paths into the properties file; document required directory structure. |

---

### 6. Running / Debugging the Script

| Step | Command / Action |
|------|-------------------|
| **Manual execution** | ```bash /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Sim_Status_Monthly.properties && /path/to/MNAAS_Sim_Status_Monthly.sh``` (the script sources the properties file itself). |
| **Set debug mode** | The script already uses `set -x`; you can increase verbosity by adding `export BASH_XTRACEFD=1` before running. |
| **Check status before run** | ```grep -E 'MNAAS_Daily_ProcessStatusFlag|MNAAS_job_status' $MNAAS_Sim_Status_Monthly_ProcessStatusFileName``` |
| **View logs** | ```tail -f $MNAAS_Sim_Status_MonthlyLogPath``` |
| **Verify Java loader output** | The Java command writes to stdout/stderr; redirect to a temporary file if needed: `java … > /tmp/loader.out 2>&1`. |
| **Confirm Impala refresh** | Run the refresh query manually: `impala-shell -i $IMPALAD_HOST -q "$sim_status_monthly_tblname_refresh"` and check for errors. |
| **Force a clean start** | If a stale PID blocks execution, delete the line `MNAAS_Script_Process_Id=…` from the status file (after confirming no real process is running). |
| **Simulate failure** | Change the Java class name to a non‑existent one, run the script, and verify that an email is sent and the status file reflects `Failure`. |

---

### 7. External Configuration & Environment Dependencies

| Item | Source | Usage |
|------|--------|-------|
| **`MNAAS_Sim_Status_Monthly.properties`** | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/` | Provides all variable definitions (`$MNAAS_Sim_Status_MonthlyLogPath`, `$MNAAS_Sim_Status_Monthly_ProcessStatusFileName`, `$Insert_Monthly_Status_table`, `$Insert_sim_inventory_Monthly_Status_table`, `$CLASSPATHVAR`, `$MNAAS_Main_JarPath`, `$IMPALAD_HOST`, `$sim_status_monthly_tblname_refresh`, `$sim_inventory_status_monthly_tblname_refresh`, `$MailId`, `${ccList}`, etc.). |
| **Java runtime** | System PATH (`java`) | Executes the loader JARs. |
| **Impala client** | `impala-shell` binary | Refreshes table metadata. |
| **Mail utility** | `mail` command (POSIX) | Sends failure notifications. |
| **Logger** | `logger` (syslog) | Writes to system log and the script‑specific log file. |
| **Filesystem** | Writable directories for logs, partition files, status file. | Must be accessible by the script user (typically a Hadoop service account). |

---

### 8. Suggested Improvements (TODO)

1. **Add robust PID cleanup & timeout** – implement a trap that removes the PID entry from the status file on any exit (including signals) and introduce a configurable “stale‑PID” age check to avoid deadlocks.
2. **Externalize date‑partition logic into a reusable library** – create a small Bash function or Python helper that returns the correct month partition string(s) based on a configurable “early‑month” threshold, reducing duplication across monthly scripts. 

---