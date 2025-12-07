**File:** `move-mediation-scripts/bin/MNAAS_table_statistics_calldate_aggr.sh`

---

## 1. High‑Level Summary
This Bash orchestration script drives the daily “call‑date” statistics aggregation pipeline for the MNAAS (Move‑N‑Analytics‑As‑A‑Service) platform. It creates a raw aggregation file via a Java job, removes stale Hive/Impala partitions, loads the file into a staging (non‑partitioned) table, and finally populates the production usage‑aggregation table. Throughout the run it updates a shared *process‑status* file, writes detailed logs, and on failure raises an SDP ticket and sends an email alert.

---

## 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|-----------------|
| **`MNAAS_creation_of_file`** | Executes the Java class that extracts daily traffic data and writes it to `$MNAAS_table_statistics_calldate_aggregation_data_filename`. Updates the status flag to **1**. |
| **`MNAAS_remove_partitions_from_the_table`** | Calls a generic Java “drop‑partitions” utility to delete yesterday’s partitions from the target Hive table. Refreshes the Impala metadata. Updates status flag to **2**. |
| **`MNAAS_inter_table_loading`** | Copies the aggregation file to HDFS, sets permissions, then runs a Java loader to insert the data into an intermediate (non‑partitioned) Hive table. Updates status flag to **3**. |
| **`MNAAS_usage_aggr_table_loading`** | Runs the final Java job that aggregates the intermediate data into the production usage‑aggregation table. Refreshes Impala metadata. Updates status flag to **4**. |
| **`terminateCron_successful_completion`** | Resets the status file to “Success”, records run time, writes final log entries and exits with status 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, invokes `email_and_SDP_ticket_triggering_step`, and exits with status 1. |
| **`email_and_SDP_ticket_triggering_step`** | Updates the status file to “Failure”, checks if an alert has already been sent, then sends a mail to the MOVE dev team and creates an SDP ticket via `mailx`. |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_table_statistics_calldate_aggr.properties` – defines all environment‑specific variables (paths, class names, DB/table names, hosts, log & status file locations, email recipients, etc.). |
| **External Services** | • Java runtime (multiple JARs)  <br>• Hadoop HDFS (`hadoop fs`, `hdfs dfs`)  <br>• Hive/Impala (via JDBC host/port, `impala-shell`)  <br>• Mail system (`mail`, `mailx`)  <br>• SDP ticketing system (email‑based) |
| **Files Created / Modified** | • `$MNAAS_table_statistics_calldate_aggregation_data_filename` (raw CSV/TSV)  <br>• HDFS directory `$MNAAS_table_statistics_calldate_aggr_PathName/*`  <br>• Log file `$MNAAS_table_statistics_calldate_aggr_LogPath`  <br>• Process‑status file `$MNAAS_table_statistics_calldate_aggr_ProcessStatusFileName` (flags, PID, timestamps, email‑sent flag) |
| **Assumptions** | • The properties file exists and contains valid values. <br>• Required JARs are present on the classpath. <br>• Hadoop, Hive, and Impala services are reachable. <br>• The script runs under a user with write permission to the log/status files and HDFS path. |
| **Side Effects** | • Alters Hive/Impala table partitions (data removal). <br>• Inserts rows into staging and production tables. <br>• Sends email alerts and creates SDP tickets on failure. |

---

## 4. Integration Points & Call Flow  

1. **Up‑stream** – Typically triggered by a daily cron after the raw traffic ingestion jobs have completed (e.g., `MNAAS_traffic_load.sh`). Those jobs populate the source tables that the Java extraction class reads.  
2. **Down‑stream** – The final aggregated table (`$table_statistics_calldate_aggr_tblname`) is consumed by reporting scripts such as `MNAAS_report_data_loading.sh` and BI dashboards.  
3. **Shared Artifacts** –  
   * **Process‑status file** (`*_ProcessStatusFileName`) is also read by other orchestration scripts to avoid overlapping runs.  
   * **Log directory** is monitored by log‑aggregation tools (e.g., Splunk/ELK).  
4. **External Scripts** – The Java classes invoked (`$MNAAS_table_statistics_calldate_aggr_classname`, `$drop_partitions_in_table`, `$Load_nonpart_table`, `$MNAAS_table_statistics_calldate_loading_classname`) are compiled from the same codebase as other “move” scripts (e.g., `MNAAS_ipvprobe_*`).  

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Concurrent execution** – PID check may miss a stale PID, leading to duplicate loads. | Data duplication / partition conflicts. | Use a lock file (`flock`) in addition to PID check; ensure stale PID cleanup on host reboot. |
| **Java job failure** – Non‑zero exit code not captured if the Java process forks. | Incomplete data, downstream reports stale. | Verify that Java wrappers exit with the correct status; add explicit `wait` if backgrounding is used. |
| **HDFS permission errors** – `chmod 777` may be insufficient on secure clusters. | Load step aborts. | Switch to proper HDFS ACLs; log the exact HDFS error. |
| **Impala metadata refresh failure** – `impala-shell` may time out. | Queries see stale partitions. | Add retry logic with exponential back‑off; capture exit code of `impala-shell`. |
| **Email/SDP flood** – Repeated failures could generate many tickets. | Alert fatigue. | Implement a throttling counter in the status file (e.g., only send once per 24 h). |
| **Missing/invalid properties** – Script will `set -x` but continue with empty vars. | Silent failures, hard‑to‑debug. | Add validation block after sourcing the properties file; abort early if required vars are empty. |

---

## 6. Running / Debugging the Script  

| Step | Command / Action |
|------|------------------|
| **Standard execution** | The script is scheduled via cron (e.g., `0 2 * * * /app/hadoop_users/MNAAS/.../MNAAS_table_statistics_calldate_aggr.sh`). |
| **Manual run** | ```bash -x /app/hadoop_users/MNAAS/.../MNAAS_table_statistics_calldate_aggr.sh``` – `-x` already enabled, prints each command. |
| **Check status** | `cat $MNAAS_table_statistics_calldate_aggr_ProcessStatusFileName` – view flags, PID, last run time, email‑sent flag. |
| **Inspect logs** | `tail -f $MNAAS_table_statistics_calldate_aggr_LogPath` – real‑time view of progress and errors. |
| **Force a specific step** | Edit the status flag in the process‑status file to the desired value (0‑4) and re‑run; the script will resume from that step. |
| **Debug Java jobs** | The Java classpaths and arguments are logged; run the same `java -cp …` command manually to capture stack traces. |
| **Validate HDFS path** | `hdfs dfs -ls $MNAAS_table_statistics_calldate_aggr_PathName` – ensure the directory exists and is writable. |
| **Verify Impala refresh** | `impala-shell -i $IMPALAD_HOST -q "SHOW TABLES LIKE '$table_statistics_calldate_aggr_tblname'"` – confirm the table is visible after refresh. |

---

## 7. External Configuration & Environment Variables  

| Variable (from properties) | Purpose |
|----------------------------|---------|
| `MNAAS_table_statistics_calldate_aggr_LogPath` | Absolute path to the script’s log file. |
| `MNAAS_table_statistics_calldate_aggr_ProcessStatusFileName` | Shared status/flag file used for concurrency control and alerting. |
| `MNAAS_table_statistics_calldate_aggregation_data_filename` | Local file that receives the raw aggregation output from the first Java job. |
| `MNAAS_table_statistics_calldate_aggr_PathName` | HDFS directory where the aggregation file is staged before loading. |
| `MNAAS_table_statistics_calldate_aggr_classname` | Fully‑qualified Java class that creates the aggregation file. |
| `MNAAS_table_statistics_calldate_loading_classname` | Java class that performs the final aggregation into the production table. |
| `drop_partitions_in_table`, `Load_nonpart_table` | Generic Java utilities for partition removal and non‑partitioned table load. |
| `dbname`, `table_statistics_calldate_aggr_tblname`, `table_statistics_calldate_aggr_inter_tblname` | Hive database and table names. |
| `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST` | Connection endpoints for Hive/Impala. |
| `MNAAS_Main_JarPath`, `Generic_Jar_Names`, `CLASSPATHVAR` | JAR locations and classpath composition. |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email recipients / headers for failure notifications. |
| `nameNode` | HDFS NameNode address used by the loader. |

The script **sources** this properties file at the very beginning (`. /app/hadoop_users/MNAAS/.../MNAAS_table_statistics_calldate_aggr.properties`). Any change to the pipeline (e.g., new table name) must be reflected there.

---

## 8. Suggested Improvements (TODO)

1. **Robust Locking Mechanism** – Replace the PID‑based check with `flock` on a dedicated lock file to guarantee exclusive execution even across node restarts.
2. **Configuration Validation Block** – After sourcing the properties file, iterate over a required‑variables list and abort with a clear error if any are empty or point to non‑existent paths/files.

---