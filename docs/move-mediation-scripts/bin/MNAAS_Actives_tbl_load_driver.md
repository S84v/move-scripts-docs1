**File:** `move-mediation-scripts/bin/MNAAS_Actives_tbl_load_driver.sh`  

---

## 1. High‑level Summary
This driver orchestrates the daily “Actives” data pipeline for the MNAAS (Mobile Number Allocation & Activation Service) domain. It reads a property file for configuration, ensures only one instance runs, updates a shared status file, and sequentially executes three logical stages:

1. **File ingestion & aggregation** – invokes a separate shell script to collect raw active‑feed files, merge them and load them into a Hive/Impala table.  
2. **Retention of raw daily actives** – runs a Java utility (`DropNthOlderPartition`) to drop partitions older than the configured retention period, then refreshes the Hive/Impala metadata.  
3. **End‑of‑day retention for the “raw‑daily‑at‑eod” table** – same as step 2 but on a different table that holds a snapshot taken at the end of the day.

On success the driver writes a “Success” flag to the status file; on any failure it logs the error, sends an email and creates an SDP ticket, then exits with a non‑zero status.

---

## 2. Key Functions / Sub‑routines

| Function | Responsibility |
|----------|----------------|
| `MNAAS_File_Processing_Active` | Loops over all feed sources defined in the associative array `FEED_FILE_PATTERN_ACTIVES`; for each source logs start/end, calls the aggregation script (`$MNAASDailyActivesAggregationScriptName`) with the file‑pattern and the previous‑processed‑filepath variable. Updates the status file to flag **1** (file‑processing). |
| `MNAAS_retention_for_actives_raw_daily` | Sets status flag **2**, runs the Java class `DropNthOlderPartition` against the `actives_daily_tblname` Hive table, then refreshes the Impala view (`$aggr_actives_daily_tblname_refresh`). |
| `MNAAS_retention_for_actives_raw_daily_at_eod` | Sets status flag **3**, runs the same Java class against the `actives_raw_daily_at_eod_tblname` table, refreshes Impala metadata (fallback `invalidate metadata` if refresh fails). |
| `terminateCron_successful_completion` | Writes a final “Success” state (flag 0) to the status file, records run‑time, logs completion, and exits 0. |
| `terminateCron_Unsuccessful_completion` | Logs failure, calls `email_and_SDP_ticket_triggering_step`, and exits 1. |
| `email_and_SDP_ticket_triggering_step` | Marks job status as **Failure** in the status file, checks if an SDP ticket/email has already been generated, and if not sends a notification email (via `mail`) and an SDP ticket (via `mailx`). Updates the status file to indicate ticket creation. |
| **Main block** (bottom of file) | Checks for an existing PID using the status file; if none, writes its own PID, reads the current process flag, and decides which stages to run based on that flag (0/1 → all stages, 2 → retention only, 3 → end‑of‑day retention only). Handles “already running” case. |

---

## 3. Inputs, Outputs & Side‑effects

| Category | Details |
|----------|---------|
| **Configuration / Env** | Sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Actives_tbl_load_driver.properties`. Expected variables (examples): <br>• `MNAAS_DailyActivesCombinedLoadLogPath` – log file path (stderr redirected). <br>• `MNAAS_Daily_Actives_Load_CombinedActivesStatuFile` – shared status file (key‑value pairs). <br>• `FEED_FILE_PATTERN_ACTIVES` – associative array of feed‑source → filename pattern. <br>• `MNAASShellScriptPath`, `MNAASDailyActivesAggregationScriptName` – path & name of the aggregation script. <br>• `CLASSPATHVAR`, `Generic_Jar_Names`, `DropNthOlderPartition` – Java class/JARs for retention. <br>• Hive/Impala connection vars (`HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST`). <br>• Email / SDP vars (`ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM`). |
| **External scripts** | `$MNAASShellScriptPath/$MNAASDailyActivesAggregationScriptName` – performs the actual file ingestion & load. |
| **Java utility** | `DropNthOlderPartition` – drops older partitions from a Hive table based on a retention period argument. |
| **Impala commands** | `impala-shell -i $IMPALAD_HOST -q "<SQL>"` – refreshes tables or invalidates metadata. |
| **Status file** | Updated throughout the run (flags, process name, PID, job status, timestamps, email‑sent flag). |
| **Log file** | All `logger -s` output appended to `$MNAAS_DailyActivesCombinedLoadLogPath`. |
| **Email / SDP ticket** | Sent via `mail` and `mailx` on failure (once per run). |
| **Side‑effects** | - Hive/Impala tables are modified (partition drops, metadata refresh). <br>- Files may be moved/renamed by the aggregation script (not visible here). <br>- External monitoring (e.g., cron) may rely on the status file to decide whether to trigger downstream jobs. |
| **Assumptions** | - The property file exists and defines all required variables. <br>- The status file is writable and follows a simple `key=value` format. <br>- Java class and required JARs are present on the classpath. <br>- Impala/Hive services are reachable from the host. <br>- Mail utilities (`mail`, `mailx`) are configured for the local MTA. |

---

## 4. Integration Points with Other Components

| Component | Connection Detail |
|-----------|-------------------|
| **`MNAAS_Actives_tbl_Load.sh`** (the “real” load script) | This driver is invoked by the higher‑level cron job that also runs the load script; the driver updates the same status file used by the load script. |
| **Aggregation script** (`$MNAASDailyActivesAggregationScriptName`) | Called for each feed source; expected to read raw files, perform transformations, and load into the Hive table `actives_daily_tblname`. |
| **Java retention utility** (`DropNthOlderPartition`) | Shared across multiple MNAAS pipelines (e.g., for “Actives” and “Activations”). |
| **Impala metadata refresh queries** (`$aggr_actives_daily_tblname_refresh`, `$actives_raw_daily_at_eod_tblname_refresh`) | Defined in the property file; other downstream jobs depend on the refreshed tables. |
| **Monitoring / Scheduler** | Cron or Oozie job that checks the status file’s `MNAAS_Combined_ProcessStatusFlag` to decide whether to launch subsequent jobs (e.g., reporting, downstream analytics). |
| **SDP ticketing system** | Email sent to `insdp@tatacommunications.com` with specific headers; the ticketing system parses these to create incidents. |
| **`utils.js`** (from `move-Inventory\move-MSPS\utils`) | Not directly referenced, but the same property‑file pattern and status‑file handling are used across the Move suite, indicating a shared utility library for logging and status updates. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** – PID check may fail if the status file is corrupted or the previous process hung without clearing its PID. | Duplicate loads, data corruption, resource contention. | Add a timeout check (e.g., if PID exists > X hours, treat as stale and clean). |
| **Status‑file race condition** – Multiple scripts write to the same file without locking. | Inconsistent flags, wrong downstream behavior. | Use `flock` or atomic `sed -i` with a lock file around all status‑file updates. |
| **Java class not found / classpath mis‑configured** – Retention step aborts. | Data older than retention period not removed, storage bloat. | Validate classpath at script start; emit clear error and exit code 2. |
| **Impala refresh failure** – Table metadata stale. | Downstream queries return empty or stale data. | Add retry logic with exponential back‑off; fallback to `invalidate metadata`. |
| **Mail/SDP failure** – No alert on failure. | Incident may go unnoticed. | Capture mail exit status; if non‑zero, write to a secondary log and raise a system‑monitoring alarm (e.g., Nagios). |
| **Hard‑coded paths** – Property file location may change across environments. | Script fails in dev/test. | Parameterise the property file path via an environment variable (e.g., `MNAAS_PROP_FILE`). |
| **Unvalidated input patterns** – Malformed feed patterns could cause the aggregation script to process wrong files. | Data quality issues. | Validate each pattern against a whitelist or perform a dry‑run check before invoking the aggregation script. |

---

## 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure the property file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Actives_tbl_load_driver.properties` is present and contains all required variables.  
   - Verify that the status file path (`$MNAAS_Daily_Actives_Load_CombinedActivesStatuFile`) is writable.  
   - Confirm Java, Impala, and mail utilities are in the `$PATH`.  

2. **Typical execution (cron)**  
   ```bash
   /app/hadoop_users/MNAAS/scripts/MNAAS_Actives_tbl_load_driver.sh
   ```
   The script logs to `$MNAAS_DailyActivesCombinedLoadLogPath` and updates the status file.

3. **Manual run (debug mode)**  
   ```bash
   set -x   # already enabled in script
   ./MNAAS_Actives_tbl_load_driver.sh 2>&1 | tee /tmp/debug_MNAAS_Actives.log
   ```
   - Watch the console for the `logger -s` messages.  
   - After completion, inspect the status file to confirm `MNAAS_Combined_ProcessStatusFlag=0` and `MNAAS_job_status=Success`.  

4. **Checking for a stuck previous run**  
   ```bash
   grep MNAAS_Script_Process_Id $MNAAS_Daily_Actives_Load_CombinedActivesStatuFile
   ps -p <PID>
   ```
   If the PID is stale, remove the line from the status file or kill the process (after verification).  

5. **Testing failure handling**  
   - Force the aggregation script to exit with non‑zero status (e.g., rename it temporarily).  
   - Run the driver and verify that an email and SDP ticket are generated, and that the status file shows `MNAAS_job_status=Failure` and `MNAAS_email_sdp_created=Yes`.  

6. **Log inspection**  
   ```bash
   tail -f $MNAAS_DailyActivesCombinedLoadLogPath
   ```

---

## 7. External Config / Environment Variables

| Variable (from property file) | Purpose |
|-------------------------------|---------|
| `MNAAS_DailyActivesCombinedLoadLogPath` | Path to the unified log file (stderr of the script). |
| `MNAAS_Daily_Actives_Load_CombinedActivesStatuFile` | Central status/flag file shared across MNAAS pipelines. |
| `FEED_FILE_PATTERN_ACTIVES` (associative array) | Mapping of feed source names to filename glob patterns. |
| `MNAASShellScriptPath` | Directory containing the aggregation script. |
| `MNAASDailyActivesAggregationScriptName` | Name of the aggregation script invoked per feed. |
| `CLASSPATHVAR`, `Generic_Jar_Names` | Java classpath components for the retention utility. |
| `DropNthOlderPartition` | Fully‑qualified Java class name executed via `java -cp`. |
| `dbname`, `actives_daily_tblname`, `actives_raw_daily_at_eod_tblname` | Hive database and table identifiers. |
| `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST` | Connection endpoints for Hive/Impala. |
| `aggr_actives_daily_tblname_refresh`, `actives_raw_daily_at_eod_tblname_refresh` | SQL statements (usually `REFRESH <table>`) used after partition drops. |
| `mnaas_retention_period_actives_raw_daily`, `mnaas_retention_period_actives_raw_daily_at_eod` | Retention period (e.g., number of days) passed to the Java utility. |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email routing information for failure notifications. |
| `MNAASDailyActivesProcessingScript` | Human‑readable script name used in log messages and email subjects. |

If any of these variables are missing, the script will abort with a non‑zero exit code; therefore they should be validated at start‑up.

---

## 8. Suggested Improvements (TODO)

1. **Add robust file‑locking around status‑file updates** – use `flock` to prevent race conditions when multiple MNAAS jobs run concurrently (e.g., activations, actives).  
2. **Externalize retry logic for Impala refresh** – wrap `impala-shell` calls in a function that retries up to 3 times with exponential back‑off, and logs a distinct error if all attempts fail.

---