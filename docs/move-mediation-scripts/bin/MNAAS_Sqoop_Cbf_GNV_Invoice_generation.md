**MNAAS_Sqoop_Cbf_GNV_Invoice_generation.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)  
This script is the daily “Sqoop load” step for the **CBF → GNV invoice generation** data set. It extracts the `cbf_auto_move_details_mview` view from a PostgreSQL CBF database, writes the raw delimited files to a dedicated HDFS staging directory, truncates the target Hive table, loads the new data into Hive (and refreshes the corresponding Impala table), and updates a shared process‑status file. It also handles concurrency control, logging, success/failure flagging, and automatic email/SDP ticket generation on error.

---

### 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **CBF_GNV_Invoice_generation_table_Sqoop** | Core ETL: clean HDFS staging area, truncate Hive table, run Sqoop import, load data into Hive, refresh Impala, and log each step. |
| **terminateCron_successful_completion** | Reset process‑status flags to *Success*, record run‑time, write final log entries, and exit with status 0. |
| **terminateCron_Unsuccessful_completion** | Log failure, update status file with *Failure*, invoke email/SDP alert, and exit with status 1. |
| **email_and_SDP_ticket_triggering_step** | Send a single notification email (and implicitly create an SDP ticket) when a failure occurs, guarding against duplicate alerts. |
| **Main program block** | Concurrency guard (PID check), reads the daily flag, decides whether to run the Sqoop step, and routes to the appropriate termination routine. |

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | • Property file `MNAAS_Sqoop_Cbf_GNV_Invoice_generation.properties` (defines all variables).<br>• Process‑status file (path stored in `$CBF_GNV_Invoice_generation_Sqoop_ProcessStatusFile`).<br>• PostgreSQL connection details (`$OrgDetails_*`). |
| **Outputs** | • HDFS directory `${CBF_GNV_Invoice_generation_table_Dir}` containing raw Sqoop files.<br>• Hive table `$dbname.$CBF_GNV_Invoice_generation_table_name` populated with the new rows.<br>• Impala table refreshed.<br>• Log file `$CBF_GNV_Invoice_generation_table_SqoopLogName` (appended). |
| **Side‑effects** | • Deletes any existing files under the HDFS staging directory (`hadoop fs -rm -r …/*`).<br>• Truncates the Hive table before load.<br>• Updates the shared status file with flags, timestamps, PID, and email‑sent flag.<br>• Sends an email (and implicitly creates an SDP ticket) on failure. |
| **Assumptions** | • Hive, Impala, Hadoop, and Sqoop binaries are in the PATH.<br>• The target Hive table already exists and matches the source schema (fields terminated by `^`).<br>• The process‑status file is writable by the script user.<br>• Mail utility (`mail`) is configured and reachable.<br>• No other concurrent instance of the same script runs (enforced by PID check). |

---

### 4. Integration with Other Scripts / Components  

| Connected Component | Interaction |
|---------------------|-------------|
| **Other MNAAS Sqoop scripts** (e.g., `MNAAS_Sqoop.sh`, `MNAAS_SimInventory_tbl_Load.sh`) | Follow the same status‑file convention; the daily flag (`MNAAS_Daily_ProcessStatusFlag`) is shared across the pipeline to coordinate sequencing. |
| **Process‑status file** (`*_Sqoop_ProcessStatusFile`) | Updated by this script and read by downstream jobs (e.g., downstream aggregation or reporting scripts) to decide whether to proceed. |
| **Hive/Impala** | The target table is consumed by downstream analytics jobs (monthly reports, invoicing, etc.). |
| **PostgreSQL CBF source** | Source of the `cbf_auto_move_details_mview` view; any schema change here must be reflected in the Hive table definition. |
| **Email/SDP system** | Failure alerts are sent to the distribution list `${GTPMailId}` and CC list `${ccList}`; the flag `MNAAS_email_sdp_created` prevents duplicate tickets. |
| **Cron scheduler** | The script is typically invoked by a daily cron entry (e.g., `0 2 * * * /path/MNAAS_Sqoop_Cbf_GNV_Invoice_generation.sh`). |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale PID / false‑positive concurrency guard** | New run aborted while previous run actually finished. | Add a timeout check on the PID file (e.g., verify process start time) or clean the PID entry on abnormal exit. |
| **Property file missing or malformed** | Script fails early, no data load. | Validate required variables at start; exit with clear error code if any are empty. |
| **PostgreSQL connectivity loss** | Sqoop import fails → downstream jobs missing data. | Implement retry logic (e.g., 3 attempts with exponential back‑off) before marking failure. |
| **Hive table schema drift** | Load succeeds but data mis‑aligned, causing downstream errors. | Add a schema‑validation step (e.g., compare column count/types) before `hive -e "load data …"` . |
| **HDFS permission errors** | Staging directory cannot be cleared or written. | Ensure the script runs under a dedicated service account with proper ACLs; log `hadoop fs -ls` output on error. |
| **Email flooding on repeated failures** | SDP ticket spam, alert fatigue. | Enforce a cooldown (e.g., only send email if last alert > 4 h ago) or integrate with a ticketing API that deduplicates. |
| **Log file growth** | Unlimited log size over time. | Rotate logs daily via `logrotate` or truncate after a defined size. |

---

### 6. Running / Debugging the Script  

| Step | Command / Action |
|------|-------------------|
| **Manual execution** | `bash /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Sqoop_Cbf_GNV_Invoice_generation.properties && ./MNAAS_Sqoop_Cbf_GNV_Invoice_generation.sh` (ensure the property file is sourced). |
| **Enable verbose tracing** | The script already sets `set -vx`; keep it for debugging. |
| **Check PID lock** | `cat $CBF_GNV_Invoice_generation_Sqoop_ProcessStatusFile | grep MNAAS_Script_Process_Id` to see the stored PID; verify with `ps -p <pid>`. |
| **Inspect logs** | Tail the log file: `tail -f $CBF_GNV_Invoice_generation_table_SqoopLogName`. |
| **Force a run ignoring the flag** | Temporarily set `MNAAS_Daily_ProcessStatusFlag=0` in the status file, then re‑run. |
| **Simulate failure** | Introduce a syntax error or change the PostgreSQL password; observe email/SDP alert generation. |
| **Validate Hive load** | After run, query Hive: `hive -e "select count(*) from $dbname.$CBF_GNV_Invoice_generation_table_name;"`. |
| **Impala refresh check** | `impala-shell -i $IMPALAD_HOST -q "describe $dbname.$CBF_GNV_Invoice_generation_table_name;"`. |

---

### 7. External Configuration, Environment Variables & Files  

| Variable / File | Origin | Usage |
|-----------------|--------|-------|
| **`MNAAS_Sqoop_Cbf_GNV_Invoice_generation.properties`** | External property file (same directory as other MNAAS scripts) | Defines all `$CBF_GNV_Invoice_generation_*`, `$OrgDetails_*`, `$IMPALAD_HOST`, `$GTPMailId`, `$ccList`, `$dbname`, `$location_table_name`, etc. |
| **`$CBF_GNV_Invoice_generation_Sqoop_ProcessStatusFile`** | Path from properties | Central status file shared across daily jobs; holds flags, PID, timestamps, email‑sent flag. |
| **`$CBF_GNV_Invoice_generation_table_SqoopLogName`** | Path from properties | Log file where all stdout/stderr are appended. |
| **`$OrgDetails_*`** (ServerName, PortNumber, Service, Username, Password) | PostgreSQL connection details | Used by Sqoop `--connect` and authentication. |
| **`$IMPALAD_HOST`** | Impala daemon host | Used for `impala-shell -i`. |
| **`$GTPMailId`, `$ccList`** | Email recipients | Used by `mail` command for failure alerts. |
| **`$MNAAS_Sqoop_CBF_GNV_Invoice_generation_Scriptname`** | Usually set in properties (script filename) | Used in log messages and email subject. |
| **`$MNAAS_FlagValue`** (read from status file) | Runtime flag indicating whether the daily process may run (0 or 1). | Controls entry into the Sqoop step. |

---

### 8. Suggested TODO / Improvements  

1. **Add robust parameter validation** – early exit with a clear error if any required property (e.g., DB credentials, HDFS dir, Hive table) is missing or empty.  
2. **Implement retry/back‑off for external calls** – wrap the Sqoop import and Hive load in a loop that retries up to 3 times on transient failures, logging each attempt.  

--- 

*End of documentation.*