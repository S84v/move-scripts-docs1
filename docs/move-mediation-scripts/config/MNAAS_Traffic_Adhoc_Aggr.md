**High‑Level Documentation – `move-mediation-scripts/config/MNAAS_Traffic_Adhoc_Aggr.sh`**

---

### 1. Purpose (one‑paragraph summary)

`MNAAS_Traffic_Adhoc_Aggr.sh` is the master orchestration script for the “traffic‑adhoc aggregation” pipeline in the MNAAS data‑warehouse. It drives a four‑step process that (1) removes stale Hive/Impala partitions, (2) runs an intermediate Java‑based aggregation, (3) loads the aggregated rows into the final traffic‑adhoc table, and (4) builds a summary‑aggregation table. The script maintains a shared *process‑status* file, logs every step, refreshes Impala metadata, and on any failure sends an SDP ticket/e‑mail to the support team. It is typically invoked nightly by cron and is designed to be re‑entrant – it will resume from the last successful step if re‑run after a failure.

---

### 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|-----------------|
| **`MNAAS_remove_partitions_from_the_table`** | Updates status flag → 1, sets proper HDFS permissions, invokes Java class `drop_partitions_in_table` for both `traffic_aggr_adhoc` and `traffic_summary_aggr` tables, then runs `impala-shell` refreshes. |
| **`MNAAS_traffic_aggr_adhoc_inter_table_loading`** | Sets flag → 2, runs the Java aggregation class (`traffic_aggr_adhoc_aggregation_classname`) that reads raw daily partitions and writes to an intermediate Hive table, followed by an Impala refresh. |
| **`MNAAS_traffic_aggr_adhoc_table_loading`** | Sets flag → 3, launches a helper shell `traffic_aggr_adhoc_refresh.sh` (nohup), then runs the Java load class (`MNAAS_traffic_aggr_adhoc_load_classname`) to populate the final `traffic_aggr_adhoc` table and refreshes Impala. |
| **`MNAAS_traffic_summary_aggr_table_loading`** | Sets flag → 4, runs the Java summary‑aggregation class (`traffic_summary_aggr_aggregation_classname`) that builds the `traffic_summary_aggr` table, then refreshes Impala. |
| **`terminateCron_successful_completion`** | Resets status flag → 0, marks job as *Success*, clears email‑ticket flag, writes run‑time, logs completion and exits 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, calls `email_and_SDP_ticket_triggering_step`, exits 1. |
| **`email_and_SDP_ticket_triggering_step`** | Marks job as *Failure*, checks if an SDP ticket has already been raised, and if not sends a formatted e‑mail (via `mailx`) to the Cloudera support address and the MOVE dev team, then updates the status file. |
| **Main block** | Prevents concurrent runs using the PID stored in the status file, reads the current flag, and executes the required subset of the above functions to bring the pipeline to completion. |

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Configuration / Env variables** (sourced from `MNAAS_Traffic_Adhoc_Aggr.properties`) | `MNAAS_traffic_aggr_adhocLogPath`, `MNAAS_traffic_aggr_adhoc_ProcessStatusFileName`, `MNAAS_traffic_aggr_adhoc_ScriptName`, `CLASSPATHVAR`, `Generic_Jar_Names`, `MNAAS_Main_JarPath`, `traffic_aggr_adhoc_tblname`, `traffic_summary_aggr_tblname`, `traffic_aggr_adhoc_inter_tblname_refresh`, `traffic_aggr_adhoc_tblname_refresh`, `traffic_summary_aggr_tblname_refresh`, `IMPALAD_HOST`, `HIVE_HOST`, `HIVE_JDBC_PORT`, `Dname_*` (Java process names), `SDP_ticket_*` (mail parameters), `MOVE_DEV_TEAM`, etc. |
| **Static files** | `MNAAS_Traffic_AdhocPartitions_Daily_Uniq_FileName` (partition list) and its “temp” counterpart for the summary table. |
| **External services** | HDFS (chmod, directory tree), Hive Metastore (via JDBC), Impala daemon (metadata refresh), Java runtime (custom JARs), mail server (`mailx`), SDP ticketing system (email‑based). |
| **Outputs** | Updated Hive/Impala tables (`traffic_aggr_adhoc`, `traffic_summary_aggr`), refreshed Impala metadata, log file at `$MNAAS_traffic_aggr_adhocLogPath`, updated process‑status file, optional SDP ticket e‑mail. |
| **Side‑effects** | Changes file permissions on HDFS, may delete/replace partitions, writes to shared status file (used by other MOVE scripts), may spawn background `nohup` process (`traffic_aggr_adhoc_refresh.sh`). |
| **Assumptions** | • All referenced Java classes exist in `$CLASSPATHVAR:$Generic_Jar_Names` or `$MNAAS_Main_JarPath`. <br>• The status file is writable by the script user and follows the key‑value format used throughout the MOVE suite. <br>• Impala and Hive services are reachable from the host running the script. <br>• `mailx` is correctly configured to send external e‑mail. <br>• No other process modifies the same status file concurrently (single‑instance enforcement). |

---

### 4. Integration with Other Scripts / Components

| Connected Component | Interaction |
|---------------------|-------------|
| **`traffic_aggr_adhoc_refresh.sh`** (called via `nohup` in `MNAAS_traffic_aggr_adhoc_table_loading`) | Likely performs a final “refresh” of the traffic‑adhoc table (e.g., compaction or additional loading). |
| **Java JARs** (`drop_partitions_in_table`, aggregation and load classes) | Shared across many MOVE scripts (e.g., `service_based_charges.sh`, `traffic_nodups_summation.py`). They read the same partition files and write to the same Hive DB (`mnaas`). |
| **Process‑status file** (`$MNAAS_traffic_aggr_adhoc_ProcessStatusFileName`) | Same file is used by other orchestration scripts to coordinate retries and to drive downstream notifications (e.g., email/SDP ticket). |
| **Impala‑shell commands** | Refreshes tables that are later queried by reporting scripts such as `test.sh` or `traffic_aggr_adhoc_...` jobs. |
| **Cron scheduler** | The script is intended to be launched by a nightly cron entry (similar to other MOVE scripts). |
| **Support / ticketing** | Uses the same e‑mail template and SDP ticket addresses as other MOVE scripts (e.g., `service_based_charges.sh`). |
| **HDFS directories** (`/user/hive/warehouse/mnaas.db/...`) | Shared storage for all Hive tables used by the MOVE suite. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** (stale PID or race condition) | Duplicate partition drops or double loads → data corruption. | Verify PID check with `ps -p $PID` and lock file; add a timeout to clear stale PID entries. |
| **Java class failure (non‑zero exit)** | Pipeline stops, downstream jobs see incomplete data. | Capture Java stdout/stderr to separate log files; implement retry logic for transient failures. |
| **Impala refresh failure** | New partitions not visible to downstream queries. | Check `impala-shell` return code; if non‑zero, retry a limited number of times before aborting. |
| **HDFS permission errors** (chmod 777 may be insufficient on new directories) | Partition drop or load fails. | Ensure the script runs as the correct Hadoop user; consider using ACLs instead of world‑writable permissions. |
| **Process‑status file corruption** | Wrong flag values cause incorrect step execution. | Validate file format at start; backup the file before each write; use atomic `sed` replacements. |
| **Mail/SDP ticket not sent** | Failure goes unnoticed. | Add a fallback to write an entry into a monitoring system (e.g., Nagios) if `mailx` returns error. |
| **Missing partition list file** | Drop‑partition step aborts. | Pre‑flight check for existence and readability of `$MNAAS_Traffic_AdhocPartitions_Daily_Uniq_FileName*`. |
| **Resource exhaustion (Java OOM, HDFS space)** | Job crashes or hangs. | Monitor JVM heap usage; add alerts on HDFS usage thresholds. |

---

### 6. Running / Debugging the Script

1. **Standard execution (cron)**  
   ```bash
   /app/hadoop_users/MNAAS/MNAAS_CronFiles/MNAAS_Traffic_Adhoc_Aggr.sh
   ```
   The script writes its log to the path defined by `MNAAS_traffic_aggr_adhocLogPath`.

2. **Manual run (debug mode)**  
   ```bash
   bash -x /app/hadoop_users/MNAAS/MNAAS_CronFiles/MNAAS_Traffic_Adhoc_Aggr.sh > /tmp/debug.out 2>&1
   ```
   `set -x` is already enabled; the `-x` flag adds extra trace.

3. **Check current status**  
   ```bash
   grep -E 'MNAAS_Daily_ProcessStatusFlag|MNAAS_job_status' $MNAAS_traffic_aggr_adhoc_ProcessStatusFileName
   ```

4. **Force a specific step** (e.g., re‑run only the summary aggregation)  
   ```bash
   sed -i 's/MNAAS_Daily_ProcessStatusFlag=.*/MNAAS_Daily_ProcessStatusFlag=3/' $MNAAS_traffic_aggr_adhoc_ProcessStatusFileName
   /app/hadoop_users/MNAAS/MNAAS_CronFiles/MNAAS_Traffic_Adhoc_Aggr.sh
   ```

5. **Verify Impala tables** after run:  
   ```bash
   impala-shell -i $IMPALAD_HOST -q "SHOW TABLES IN mnaas LIKE 'traffic_%';"
   ```

6. **Inspect logs** for failures: look for lines containing “FAILED” or “terminated Unsuccessfully”.

---

### 7. External Configuration / Environment Dependencies

| File / Variable | Role |
|-----------------|------|
| **`MNAAS_Traffic_Adhoc_Aggr.properties`** | Supplies all `$MNAAS_*` variables (paths, DB names, Java class names, email addresses, etc.). |
| **`MNAAS_Traffic_AdhocPartitions_Daily_Uniq_Apr_mismatch`** | Text file listing HDFS partition directories to be dropped. |
| **`$MNAAS_traffic_aggr_adhocLogPath`** | Destination for `stderr` and explicit `logger` entries. |
| **`$MNAAS_traffic_aggr_adhoc_ProcessStatusFileName`** | Central status/lock file shared across the MOVE suite. |
| **Java classpath variables** (`CLASSPATHVAR`, `Generic_Jar_Names`, `MNAAS_Main_JarPath`) | Must point to the compiled JARs containing `drop_partitions_in_table`, aggregation, and load classes. |
| **Impala / Hive connection details** (`IMPALAD_HOST`, `HIVE_HOST`, `HIVE_JDBC_PORT`) | Required for metadata refresh and partition‑drop JDBC calls. |
| **Mail variables** (`SDP_ticket_from_email`, `SDP_ticket_to_email`, `MOVE_DEV_TEAM`) | Used by `email_and_SDP_ticket_triggering_step`. |
| **Process‑name variables** (`Dname_MNAAS_*`) | Provide unique identifiers for the `ps` checks that prevent duplicate Java processes. |

If any of these are missing or malformed, the script will abort with an error logged to the same log file.

---

### 8. Suggested Improvements (TODO)

1. **Replace ad‑hoc PID / `ps | grep` checks with a lockfile** (e.g., `flock`) to avoid false positives caused by the `grep` process itself and to automatically clean up stale locks.

2. **Externalize the Impala refresh commands** into a reusable function that validates the return code and retries, reducing duplication across the four step functions.

---