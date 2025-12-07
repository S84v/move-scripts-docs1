**File:** `move-mediation-scripts/bin/MNAAS_Traffic_tbl_with_nodups_loading_apr21_mismatch.sh`

---

## 1. Purpose (one‑paragraph summary)

This script orchestrates the daily “traffic‑details‑with‑no‑duplicates” load for the MNAAS data‑mediation pipeline (April‑2021 partition‑mismatch variant). It sequentially (a) removes stale Hive/Impala partitions, (b) loads KYC feed data as a prerequisite, (c) runs two Java‑based loading stages (inter‑table and final table), (d) executes an SMS‑only post‑processing step, and (e) updates a shared process‑status file while handling success/failure notifications (email and SDP ticket). The script is designed to be idempotent, prevent concurrent executions, and integrate with the broader MNAAS traffic‑aggregation suite.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|-----------------|
| **`MNAAS_remove_partitions_from_the_table`** | Updates process‑status flag to *1*, checks that the partition‑drop Java job is not already running, invokes `drop_partitions_in_table` with the April‑2021 partition config, logs success/failure, and refreshes the Impala table. |
| **`MNAAS_Traffic_tbl_with_nodups_inter_table_loading`** | Sets flag to *2*, runs the KYC feed loading script (`MNAAS_Daily_KYC_Feed_tbl_Loading_Script`), then launches the “inter‑table” Java loader (`MNAAS_Traffic_tbl_with_nodups_inter_loading_load_classname`). |
| **`MNAAS_Traffic_tbl_with_nodups_table_loading`** | Sets flag to *3*, runs the final “no‑dups” Java loader (`MNAAS_Traffic_tbl_with_nodups_loading_load_classname`), refreshes the Impala table, and logs outcome. |
| **`MNAAS_Traffic_tbl_with_nodups_sms_only`** | Sets flag to *4* and executes a shell script that extracts/loads SMS‑only traffic (`MNAAS_traffic_no_dups_sms_only_ScriptName`). |
| **`terminateCron_successful_completion`** | Resets status flags to *0* (Success), writes a final log entry, and exits with code 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, triggers `email_and_SDP_ticket_triggering_step`, and exits with code 1. |
| **`email_and_SDP_ticket_triggering_step`** | Updates the status file to *Failure*, sends a notification email to the MOVE dev team, creates an SDP ticket via `mailx`, and marks the ticket‑sent flag to avoid duplicate alerts. |
| **Main block** | Reads the PID from the status file, prevents concurrent runs, determines the current flag value, and invokes the appropriate subset of the above functions to resume a partially‑completed run. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration file** | `MNAAS_Traffic_tbl_with_nodups_loading.properties` – defines all environment variables used (log paths, status file, class names, jar locations, Hive/Impala hosts, etc.). |
| **Process‑status file** | `$MNAAS_Traffic_tbl_with_nodups_loading_ProcessStatusFileName` – a simple key‑value file tracking flags, PID, job status, email‑sent flag, etc. |
| **External scripts** | `MNAAS_Daily_KYC_Feed_tbl_Loading_Script` (KYC feed load) and `MNAAS_traffic_no_dups_sms_only_ScriptName` (SMS‑only processing). |
| **Java classes / JARs** | `drop_partitions_in_table`, `MNAAS_Traffic_tbl_with_nodups_inter_loading_load_classname`, `MNAAS_Traffic_tbl_with_nodups_loading_load_classname`. |
| **Databases / Data stores** | Hive tables (`traffic_details_raw_daily_with_no_dups_tblname`), Impala tables (same name refreshed via `impala-shell`). |
| **Messaging / Notification** | System logger (`logger`), `mail` (plain email), `mailx` (SDP ticket). |
| **Outputs** | – Log file (`$MNAAS_Traffic_tbl_with_nodups_loadingLogPath`).<br>– Updated Hive/Impala tables.<br>– Process‑status file reflecting final state.<br>– Email / SDP ticket on failure. |
| **Assumptions** | • All required environment variables are correctly set in the properties file.<br>• Java runtime, Hive, Impala, and mail services are reachable from the host.<br>• No other instance of the same Java class is running (checked via `ps`).<br>• The partition‑config file `/app/hadoop_users/MNAAS/MNAAS_Configuration_Files/MNAAS_Traffic_Partitions_Daily_Uniq_Apr_mismatch` exists and is valid for the current date range. |

---

## 4. Integration Points with Other Scripts / Components

| Component | Connection Detail |
|-----------|-------------------|
| **`MNAAS_Traffic_tbl_with_nodups_loading.properties`** | Centralised property store used by many traffic‑loading scripts (e.g., `MNAAS_Traffic_tbl_Load.sh`, `MNAAS_Traffic_seq_check.sh`). |
| **KYC Feed Loader** (`MNAAS_Daily_KYC_Feed_tbl_Loading_Script`) | Must complete successfully before any traffic load proceeds; shared across daily aggregation pipelines. |
| **SMS‑Only Loader** (`MNAAS_traffic_no_dups_sms_only_ScriptName`) | Re‑used by other “no‑dups” scripts (e.g., `MNAAS_Traffic_tbl_with_nodups_loading.sh`). |
| **Partition Drop Java Class** (`drop_partitions_in_table`) | Same class invoked by other daily table‑maintenance scripts (e.g., `MNAAS_Traffic_tbl_with_nodups_loading.sh`). |
| **Impala Refresh Queries** (`traffic_details_raw_daily_with_no_dups_tblname_refresh`) | Defined in the properties file; other aggregation scripts rely on the refreshed table. |
| **Process‑status file** | Shared with monitoring/alerting tools and possibly with a UI that displays job health (e.g., `MNAAS_Traffic_seq_check.sh`). |
| **SDP Ticketing** | Uses the same email‑to‑SDP gateway as other MOVE scripts (`MNAAS_Traffic_Loading_Progress_Trend_Mail.sh`). |
| **Cron Scheduler** | Typically invoked from a daily cron entry; other scripts in the same family have similar cron wrappers. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** (multiple instances of the same Java class or script) | Data corruption, duplicate loads, partition loss | The script already checks PID and `ps`. Enforce a system‑wide lock file and monitor the status file for stale PIDs. |
| **Missing or malformed properties** | Job aborts early, silent failures | Validate the properties file at script start (e.g., `source` with `set -u`), and add a sanity‑check block that exits with a clear error if required vars are empty. |
| **Java class failure** (non‑zero exit) | Incomplete load, downstream jobs fail | Capture Java stdout/stderr to a dedicated log, and add retry logic for transient failures (e.g., network hiccup to Hive). |
| **Impala refresh failure** | Tables not visible to downstream analytics | Check the exit code of `impala-shell`; on failure, trigger a separate “refresh retry” job and alert. |
| **Email/SDP ticket not sent** (mail service down) | No visibility of failure to ops team | Add fallback to write a file‑based alert in a monitored directory; also log the mail command’s exit status. |
| **Partition‑config mismatch** (April‑2021 specific file) | Wrong partitions dropped, data loss | Version‑control the config file, and add a checksum verification step before invoking the drop job. |
| **KYC feed dependency failure** | Whole pipeline stops | Isolate KYC loading into its own retryable job; if it fails, raise a high‑severity alert rather than silently aborting the traffic load. |

---

## 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure the properties file exists and is readable.  
   - Verify that required JARs and configuration files are present on the host.  
   - Confirm that the user has permission to write to the log and status files and to execute `impala-shell`, `java`, and `mailx`.

2. **Typical Invocation** (via cron or manually)  
   ```bash
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_Traffic_tbl_with_nodups_loading_apr21_mismatch.sh
   ```

3. **Manual Debug**  
   - The script starts with `set -x`; run it interactively to see each command.  
   - Tail the log file while the script runs:  
     ```bash
     tail -f $MNAAS_Traffic_tbl_with_nodups_loadingLogPath
     ```  
   - Check the process‑status file to see the current flag and PID:  
     ```bash
     cat $MNAAS_Traffic_tbl_with_nodups_loading_ProcessStatusFileName
     ```  
   - If a Java step fails, inspect the Java class logs (often written to `$MNAAS_Traffic_tbl_with_nodups_loadingLogPath`).

4. **Force Re‑run after a Failure**  
   - Edit the status file to set `MNAAS_Daily_ProcessStatusFlag=0` (or the appropriate step number) and clear the PID line.  
   - Re‑execute the script; it will resume from the flagged step.

5. **Testing**  
   - Run the script with a “dry‑run” flag (not currently present) – consider adding `--dry-run` that only logs the intended actions without invoking Java or Impala.

---

## 7. External Config / Environment Variables

| Variable (populated in the `.properties` file) | Meaning |
|-----------------------------------------------|---------|
| `MNAAS_Traffic_tbl_with_nodups_loadingLogPath` | Full path to the script’s log file. |
| `MNAAS_Traffic_tbl_with_nodups_loading_ProcessStatusFileName` | Path to the shared status/flag file. |
| `CLASSPATHVAR`, `Generic_Jar_Names`, `MNAAS_Main_JarPath` | Java classpath components for the loader JARs. |
| `Dname_MNAAS_drop_partitions_Traffic_tbl_with_nodups_loading_tbl` | Identifier used to detect a running partition‑drop job. |
| `drop_partitions_in_table` | Fully‑qualified Java class name for partition removal. |
| `traffic_details_raw_daily_with_no_dups_tblname_refresh` | Impala SQL statement to refresh the target table. |
| `IMPALAD_HOST` | Impala daemon host for `impala-shell`. |
| `MNAAS_Daily_KYC_Feed_tbl_Loading_Script` | Path to the KYC feed loading script. |
| `MNAAS_traffic_no_dups_sms_only_ScriptName` | Path to the SMS‑only processing script. |
| `MNAAS_traffic_aggr_adhoc_ScriptName` | Human‑readable name used in log/email subjects. |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email routing parameters for failure notifications. |
| `MNAAS_Traffic_tbl_with_nodups_inter_loading_load_classname`, `MNAAS_Traffic_tbl_with_nodups_loading_load_classname` | Java class names for the two loading stages. |
| `MNAAS_Property_filename` | Reference to the same properties file (passed to Java). |
| `MNAAS_Traffic_Partitions_Daily_Uniq_Apr_mismatch` | Partition‑definition file specific to the April‑2021 mismatch scenario. |

---

## 8. Suggested Improvements (TODO)

1. **Add a “dry‑run / validation” mode** – before invoking any Java or Impala command, verify that all required files, tables, and services are reachable and log a summary. This reduces accidental data loss during configuration changes.

2. **Centralise status‑file handling** – extract the flag‑update and PID‑management logic into a reusable library (e.g., `MNAAS_status_util.sh`) so that all MOVE scripts share identical semantics and can be unit‑tested independently.