**MNAAS_Usage_Trend_Aggr.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)

`MNAAS_Usage_Trend_Aggr.sh` is the orchestrator for the daily “usage‑trend” aggregation pipeline in the MNAAS mediation suite. It creates a temporary usage‑trend file, removes stale Hive partitions, loads the file into a staging (non‑partitioned) Hive table, then aggregates the data into the final daily usage‑trend table. The script is guarded by a process‑status file to allow safe restart after a failure and to prevent concurrent executions. All steps are performed via Java JARs, Hadoop/HDFS commands, and Impala/Hive refreshes; failures trigger email alerts and an SDP ticket.

---

### 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **MNAAS_creation_of_file** | Generates the daily usage‑trend flat file (`$MNAAS_User_Trend_Data_filename`) by invoking the Java class `$MNAAS_usage_trend_aggregation_classname`. Updates process‑status flag to **1**. |
| **MNAAS_remove_partitions_from_the_table** | Drops daily partitions listed in `$MNAAS_Traffic_Partitions_Daily_Uniq_FileName` from the Hive table `$usage_trend_daily_aggr_tblname`. Refreshes the Impala metadata. Updates flag to **2**. |
| **MNAAS_usage_inter_table_loading** | Copies the generated flat file to HDFS (`$MNAAS_Daily_Aggregation_Usertrend_PathName`), sets permissions, and loads it into a temporary non‑partitioned Hive table `$usage_trend_inter_daily_aggr_tblname`. Updates flag to **3**. |
| **MNAAS_usage_aggr_table_loading** | Executes the final aggregation Java class `$MNAAS_usage_trend_load_classname` to populate the daily aggregated usage‑trend table. Refreshes Impala metadata. Updates flag to **4**. |
| **terminateCron_successful_completion** | Resets the process‑status file to “Success”, clears the running‑PID, logs completion, and exits with status 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, invokes `email_and_SDP_ticket_triggering_step`, and exits with status 1. |
| **email_and_SDP_ticket_triggering_step** | Sends a failure notification email to the MOVE dev team, creates an SDP ticket via `mailx`, and marks the status file to avoid duplicate tickets. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_Usage_Trend_Aggr.properties` – defines all environment variables used (paths, class names, DB names, hosts, ports, email lists, etc.). |
| **External Files** | • Process‑status file: `$MNAAS_usage_trend_Aggr_ProcessStatusFileName` (holds flags, PID, timestamps).<br>• Log file: `$MNAAS_usage_trend_AggrLogPath`.<br>• Partition list file: `$MNAAS_Traffic_Partitions_Daily_Uniq_FileName` (produced by upstream traffic‑detail scripts). |
| **Generated Files** | `$MNAAS_User_Trend_Data_filename` – flat file with daily usage data (temporary, truncated each run). |
| **HDFS** | Writes to `$MNAAS_Daily_Aggregation_Usertrend_PathName/*`. |
| **Hive / Impala** | Alters partitions on `$usage_trend_daily_aggr_tblname`; loads data into `$usage_trend_inter_daily_aggr_tblname`; refreshes Impala metadata. |
| **Java Jobs** | Four distinct JAR executions (aggregation, partition drop, load, final aggregation). |
| **Notifications** | Email via `mail` and `mailx`; SDP ticket creation. |
| **Assumptions** | • All variables are correctly defined in the properties file.<br>• Java classpaths and JARs are present and executable.<br>• Hadoop, Hive, Impala services are reachable and the user has required permissions.<br>• Mail utilities are configured on the host.<br>• The process‑status file exists and is writable. |

---

### 4. Interaction with Other Scripts / Components

| Connected Component | Relationship |
|---------------------|--------------|
| **`MNAAS_TrafficDetails_tbl_Load_new.sh`** (previous script) | Produces the daily traffic partition list (`$MNAAS_Traffic_Partitions_Daily_Uniq_FileName`) consumed by `MNAAS_remove_partitions_from_the_table`. |
| **`MNAAS_Traffic_tbl_with_nodups_sms_only_loading.sh`** (previous script) | Also contributes to the same partition list; ensures duplicate‑free traffic data before usage aggregation. |
| **Java JARs** (`$MNAAS_Main_JarPath`, `$Generic_Jar_Names`) | Implement the core business logic for file creation, partition management, and aggregation. |
| **Hadoop / HDFS** | Source/target for temporary files; required for `hadoop fs -rm`, `copyFromLocal`, `chmod`. |
| **Hive / Impala** | Target tables for partition removal and final aggregation; refreshed via `impala-shell`. |
| **SDP Ticketing System** | Integrated via `mailx` to raise an incident when the script fails. |
| **Monitoring / Cron** | Typically invoked by a daily cron job; the script itself guards against concurrent runs using the PID stored in the status file. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Concurrent execution** – stale PID or missing lock may cause duplicate runs. | Verify PID existence before starting; consider using a filesystem lock (`flock`) in addition to the status file. |
| **Java job failure** – non‑zero exit code leaves tables in inconsistent state. | Add retry logic for transient failures; ensure idempotent partition removal. |
| **HDFS permission errors** – `chmod 777` may be insufficient or insecure. | Use proper HDFS ACLs; log permission errors explicitly. |
| **Missing/invalid properties** – script aborts early with obscure errors. | Validate required variables after sourcing the properties file; fail fast with clear messages. |
| **Email/SDP flood** – repeated failures could generate many tickets. | Guard ticket creation with a timestamp check (e.g., only one ticket per hour). |
| **Stale process‑status file** – after a crash the flag may stay non‑zero. | Include a cleanup step in the cron wrapper to reset flags after a configurable timeout. |

---

### 6. Running / Debugging the Script

1. **Prerequisites** – Ensure the properties file exists at `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Usage_Trend_Aggr.properties` and all referenced JARs, Hive tables, and HDFS paths are accessible.
2. **Manual Execution**  
   ```bash
   cd /path/to/move-mediation-scripts/bin
   ./MNAAS_Usage_Trend_Aggr.sh
   ```
   The script already runs with `set -x`, so each command is echoed to the console and logged.
3. **Force a specific step** – Edit the process‑status file (`$MNAAS_usage_trend_Aggr_ProcessStatusFileName`) and set `MNAAS_Daily_ProcessStatusFlag` to the desired step (0‑4) before invoking the script.
4. **Check logs** – Tail the log file:
   ```bash
   tail -f $MNAAS_usage_trend_AggrLogPath
   ```
5. **Debugging Java jobs** – The Java commands include `-Dname=...`; add `-verbose:class` or increase log level inside the JAR if needed.
6. **Verify PID handling** – After a run, confirm the PID entry in the status file matches the actual process (`ps -p <pid>`). If stale, manually clear the entry.

---

### 7. External Configuration & Environment Variables

| Variable (populated from properties) | Usage |
|--------------------------------------|-------|
| `MNAAS_usage_trend_AggrLogPath` | Log file location (stderr redirection). |
| `MNAAS_usage_trend_Aggr_ProcessStatusFileName` | Central status/lock file. |
| `MNAAS_User_Trend_Data_filename` | Temporary flat file generated by the aggregation Java class. |
| `MNAAS_Traffic_Partitions_Daily_Uniq_FileName` | List of Hive partitions to drop (produced upstream). |
| `CLASSPATHVAR`, `MNAAS_Main_JarPath`, `Generic_Jar_Names` | Java classpaths for all JAR executions. |
| `Dname_*` variables | Human‑readable names passed to Java via `-Dname=` (used for logging/monitoring). |
| `dbname`, `usage_trend_daily_aggr_tblname`, `usage_trend_inter_daily_aggr_tblname` | Hive database and table names. |
| `HIVE_HOST`, `HIVE_JDBC_PORT`, `IMPALAD_HOST` | Connection endpoints for Hive/Impala. |
| `MNAAS_Daily_Aggregation_Usertrend_PathName` | HDFS directory for the temporary file. |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email recipients / headers for failure notifications. |
| `usage_trend_daily_aggr_tblname_refresh` | Impala `REFRESH` statement executed after partition drop / final load. |

---

### 8. Suggested Improvements (TODO)

1. **Replace ad‑hoc PID lock with `flock`** – Guarantees atomic lock acquisition and automatic release on process exit, reducing stale‑PID scenarios.  
2. **Externalize email/SDP parameters** – Move all mail‑related variables to a dedicated `email.properties` file and add a sanity‑check routine to verify connectivity before the main workflow starts.

---