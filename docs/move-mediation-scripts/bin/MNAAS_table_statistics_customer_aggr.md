**File:** `move-mediation-scripts\bin\MNAAS_table_statistics_customer_aggr.sh`  

---

## 1. High‑Level Summary
This Bash driver orchestrates the daily/monthly creation, partition cleanup, staging‑area load, and final aggregation of the **customer‑statistics** table used by the MNAAS (Move‑N‑Analytics‑As‑a‑Service) reporting layer. It runs a sequence of Java jobs (file generation, Hive/Impala table manipulation, and final aggregation) while maintaining a shared *process‑status* file that records the current step, PID, and overall job health. On failure it logs the error, updates the status file, and raises an SDP ticket + email notification.

---

## 2. Core Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **`MNAAS_creation_of_file`** | Generates the raw aggregation data file (`$MNAAS_table_statistics_customer_aggregation_data_filename`) by invoking the main Java class defined in the properties file. Resets the status flag to **1** (file‑creation step). |
| **`MNAAS_remove_partitions_from_the_table`** | Drops daily partitions from the target Hive/Impala table (`$table_statistics_customer_aggr_tblname`) using a generic Java utility (`$drop_partitions_in_table`). Sets status flag to **2**. Refreshes Impala metadata. |
| **`MNAAS_inter_table_loading`** | Copies the generated file to HDFS, adjusts permissions, and loads it into a *temporary* (non‑partitioned) table (`$table_statistics_customer_aggr_inter_tblname`) via another generic Java loader (`$Load_nonpart_table`). Sets status flag to **3**. |
| **`MNAAS_usage_aggr_table_loading`** | Executes the final aggregation Java class (`$MNAAS_table_statistics_customer_loading_classname`) that merges the temp table into the production aggregated table. Refreshes Impala metadata. Sets status flag to **4**. |
| **`terminateCron_successful_completion`** | Writes a **Success** state (flag = 0) to the status file, logs completion timestamps, and exits with code 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, invokes `email_and_SDP_ticket_triggering_step`, and exits with code 1. |
| **`email_and_SDP_ticket_triggering_step`** | Updates the status file to **Failure**, checks if an SDP ticket/email has already been raised, and if not sends a formatted email (via `mailx`) to the Move‑Dev team and the SDP ticketing address. Marks the ticket‑sent flag. |
| **Main block** | Prevents concurrent runs by checking the PID stored in the status file, determines the current flag value, and executes the required subset of the above functions to resume from the last successful step. |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration** | Sourced from `$MNAAS_table_statistics_customer_aggr.properties`. Expected keys (examples): <br>• `MNAAS_table_statistics_customer_aggr_LogPath` (log file) <br>• `MNAAS_table_statistics_customer_aggr_ProcessStatusFileName` (shared status file) <br>• `MNAAS_table_statistics_customer_aggregation_data_filename` (local temp file) <br>• `MNAAS_table_statistics_customer_aggr_PathName` (HDFS staging dir) <br>• `MNAAS_Main_JarPath`, `CLASSPATHVAR`, `Generic_Jar_Names` (classpath) <br>• Hive/Impala connection vars (`HIVE_HOST`, `IMPALAD_HOST`, `HIVE_JDBC_PORT`) <br>• Email & SDP vars (`GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM`, `ccList`). |
| **External Services** | • **Hadoop HDFS** – file copy / delete / chmod <br>• **Hive** – partition drop (via Java) <br>• **Impala** – metadata refresh (`impala-shell`) <br>• **Java runtime** – multiple job jars <br>• **Mail system** – `mail` / `mailx` for notifications <br>• **SDP ticketing** – email to `insdp@tatacommunications.com`. |
| **Inputs** | • Daily unique partition list file (`$MNAAS_Traffic_Partitions_Daily_Uniq_FileName`). <br>• Database name (`$dbname`). <br>• Table names (`$table_statistics_customer_aggr_tblname`, `$table_statistics_customer_aggr_inter_tblname`). |
| **Outputs** | • Aggregated data file on local FS (`$MNAAS_table_statistics_customer_aggregation_data_filename`). <br>• HDFS staging files under `$MNAAS_table_statistics_customer_aggr_PathName`. <br>• Updated Hive/Impala tables (final aggregated table). <br>• Log file (`$MNAAS_table_statistics_customer_aggr_LogPath`). <br>• Updated process‑status file (flags, PID, timestamps). |
| **Side Effects** | • Potentially drops partitions from a production Hive table (irreversible if not restored). <br>• Sends email & SDP ticket on failure. <br>• Updates shared status file used by other MNAAS scripts (e.g., weekly KYC feed, traffic detail loads). |
| **Assumptions** | • Only one instance runs at a time (enforced via PID check). <br>• Java classes are compatible with the supplied classpath and Hadoop version. <br>• HDFS, Hive, Impala services are reachable from the host. <br>• Mailx is correctly configured for outbound mail. <br>• The status file exists and is writable by the script user. |

---

## 4. Integration Points & Call Graph  

| Connected Component | Interaction |
|---------------------|-------------|
| **Other MNAAS load scripts** (e.g., `MNAAS_Traffic_tbl_*`, `MNAAS_Usage_Trend_Aggr.sh`) | Share the same *process‑status* directory and may be scheduled sequentially; they rely on the same `$MNAAS_*_ProcessStatusFileName` naming convention. |
| **Java JARs** (`$MNAAS_Main_JarPath`, `$Generic_Jar_Names`) | Provide the actual data‑generation, partition‑drop, and load logic. The script only passes parameters. |
| **Hive/Impala** | The script invokes Hive‑related Java utilities and runs `impala-shell` to refresh metadata after partition changes. |
| **HDFS** | Staging directory (`$MNAAS_table_statistics_customer_aggr_PathName`) is used as a temporary landing zone for the generated file before loading. |
| **SDP Ticketing / Email** | On failure, the script sends a formatted email to the SDP ticketing address; this is part of the broader incident‑management workflow. |
| **Cron / Scheduler** | The script is intended to be launched by a cron job (or equivalent scheduler) that sets environment variables and ensures daily execution. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** – two instances could corrupt the status file or drop the same partitions twice. | Data loss / duplicate processing. | PID check already present; enforce exclusive lock (e.g., `flock`) and monitor the status file for stale PIDs. |
| **Partition drop failure** – if the Java drop utility fails, old partitions remain, causing duplicate rows on subsequent loads. | Inconsistent reporting. | Add a verification step after drop (e.g., `hive -e "SHOW PARTITIONS …"`). |
| **HDFS permission errors** – `chmod 777` may be insufficient on secured clusters. | Load job aborts. | Use proper HDFS ACLs; log the exact `hdfs dfs -chmod` exit code. |
| **Email/SDP flood** – repeated failures could generate many tickets. | Alert fatigue. | Debounce ticket creation (already checks flag) and add a back‑off counter in the status file. |
| **Hard‑coded class names / paths** – changes to JAR locations require property updates. | Script breakage. | Externalize all paths into the properties file and validate them at start‑up. |
| **Missing/Corrupt status file** – script aborts early. | No processing. | Add a self‑heal routine: if file missing, create with default values. |

---

## 6. Running / Debugging the Script  

1. **Prerequisites**  
   - Ensure the properties file `MNAAS_table_statistics_customer_aggr.properties` is present and readable.  
   - Verify that the Java classpath variables point to the correct JARs.  
   - Confirm HDFS, Hive, Impala, and mail services are reachable from the host.  

2. **Typical Execution** (via cron or manual):  
   ```bash
   export MNAAS_ENV=prod   # if required by the properties file
   /app/hadoop_users/MNAAS/MNAAS_bin/MNAAS_table_statistics_customer_aggr.sh
   ```

3. **Debugging Steps**  
   - The script starts with `set -x`; logs each command to the log file defined by `$MNAAS_table_statistics_customer_aggr_LogPath`. Tail this file in real time: `tail -f $MNAAS_table_statistics_customer_aggr_LogPath`.  
   - Check the **process‑status** file for the current flag and PID: `cat $MNAAS_table_statistics_customer_aggr_ProcessStatusFileName`.  
   - If a step fails, the exit code of the Java call is logged; re‑run the failing function manually (e.g., `MNAAS_remove_partitions_from_the_table`) after fixing the underlying issue.  
   - Verify HDFS contents: `hdfs dfs -ls $MNAAS_table_statistics_customer_aggr_PathName`.  
   - Confirm Impala metadata refresh succeeded: `impala-shell -i $IMPALAD_HOST -q "SHOW TABLES LIKE '$table_statistics_customer_aggr_tblname'"`.  

4. **Force Restart from a Specific Step**  
   - Edit the status file and set `MNAAS_Daily_ProcessStatusFlag` to the desired step number (0‑4). The main block will resume from that step on next run.  

---

## 7. External Config / Environment Variables  

| Variable (from properties) | Purpose |
|----------------------------|---------|
| `MNAAS_table_statistics_customer_aggr_LogPath` | Path to the script’s log file. |
| `MNAAS_table_statistics_customer_aggr_ProcessStatusFileName` | Shared status/heartbeat file. |
| `MNAAS_table_statistics_customer_aggregation_data_filename` | Local temporary file that holds generated aggregation data. |
| `MNAAS_table_statistics_customer_aggr_PathName` | HDFS staging directory for the temporary file. |
| `MNAAS_Traffic_Partitions_Daily_Uniq_FileName` | Text file listing daily partitions to be removed. |
| `MNAAS_Main_JarPath`, `Generic_Jar_Names`, `CLASSPATHVAR` | Classpath components for Java jobs. |
| `drop_partitions_in_table`, `Load_nonpart_table` | Fully‑qualified Java class names for generic utilities. |
| `MNAAS_table_statistics_customer_loading_classname` | Java class that performs the final aggregation. |
| `dbname`, `table_statistics_customer_aggr_tblname`, `table_statistics_customer_aggr_inter_tblname` | Hive database and table identifiers. |
| `HIVE_HOST`, `IMPALAD_HOST`, `HIVE_JDBC_PORT` | Connection endpoints for Hive/Impala. |
| `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM`, `ccList` | Email addresses used for failure notifications. |
| `table_statistics_customer_aggr_ScriptName` | Human‑readable script identifier used in logs/emails. |

If any of these are missing or empty, the script will likely abort; a sanity‑check at the top of the script (e.g., `: ${var?Missing}`) would improve robustness.

---

## 8. Suggested Improvements (TODO)

1. **Add a pre‑flight validation block** – before any processing, verify that all required variables, files, and external services are reachable; exit early with a clear error message if not.
2. **Replace ad‑hoc PID lock with `flock`** – using `flock -n /var/lock/MNAAS_table_statistics_customer_aggr.lock` would guarantee atomic exclusive execution and automatically clean up stale locks.

---