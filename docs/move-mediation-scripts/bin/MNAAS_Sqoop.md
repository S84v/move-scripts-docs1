**MNAAS_Sqoop.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)
`MNAAS_Sqoop.sh` orchestrates the nightly extraction of two reference tables – **Rate** and **Location** – from the Oracle‑based MNAAS data warehouse into Hadoop. It uses Sqoop to pull the data into HDFS, loads the raw files into Hive staging tables, merges them into production Hive tables, refreshes the corresponding Impala tables, and updates a shared process‑status file. The script also handles success/failure signalling, generates log files, and raises an SDP ticket with email notification on error. It is designed to be invoked by a cron job and to cooperate with other MNAAS scripts that read/write the same status file.

---

### 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|-----------------|
| **Rate_table_Sqoop** | • Set process‑status flag = 1 (Rate step)  <br>• Remove previous HDFS files  <br>• Run Sqoop import for the Rate table  <br>• Load data into Hive staging (`rate_temp_table_name`) then into production (`rate_table_name`)  <br>• Refresh Impala metadata  <br>• Log each step and exit on failure |
| **Location_table_Sqoop** | • Set process‑status flag = 2 (Location step)  <br>• Remove previous HDFS files  <br>• Run Sqoop import for the Location table  <br>• Truncate and load Hive table directly  <br>• Refresh Impala metadata  <br>• Clean up the daily log file |
| **terminateCron_successful_completion** | • Reset status flag to 0 (idle)  <br>• Mark job status = Success  <br>• Record run time and write final log entries |
| **terminateCron_Unsuccessful_completion** | • Log failure, invoke **email_and_SDP_ticket_triggering_step**, exit with non‑zero code |
| **email_and_SDP_ticket_triggering_step** | • Mark job status = Failure  <br>• If no ticket/email has been sent, send an email to the GTP mailbox and raise an SDP ticket via `mailx`  <br>• Update status file to indicate ticket was created |
| **main block (bottom of script)** | • Guard against concurrent runs (process count check)  <br>• Read current flag from status file and decide which step(s) to execute (Rate → Location, or only Location)  <br>• Call the appropriate functions and finish with success or failure handling |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration / Env** | `MNAAS_ShellScript.properties` (sourced) – defines all variables used (e.g., `$Rate_Location_Sqoop_ProcessStatusFile`, `$Rate_table_Dir`, `$OrgDetails_*`, `$IMPALAD_HOST`, email lists, etc.). |
| **External Services** | • Oracle DB (via JDBC)  <br>• Hadoop HDFS (file removal, permission changes)  <br>• Hive (DDL/DML)  <br>• Impala (metadata refresh)  <br>• Local mail subsystem (`mail`, `mailx`) for notifications  <br>• SDP ticketing system (via email) |
| **Inputs** | • Process‑status file (`$Rate_Location_Sqoop_ProcessStatusFile`) – holds flags, timestamps, email‑sent flag.  <br>• Sqoop query strings (`$Rate_table_Query`, `$Location_table_Query`). |
| **Outputs** | • HDFS directories `$Rate_table_Dir` and `$Location_table_Dir` populated with raw CSV/TSV files.  <br>• Hive tables `$dbname.$rate_table_name`, `$dbname.$rate_temp_table_name`, `$dbname.$location_table_name`.  <br>• Daily log file `$Rate_Location_table_SqoopLogName$(date +_%F)`.  <br>• Updated status file (flags, timestamps, job status). |
| **Side Effects** | • Deletes any existing files in the target HDFS directories before each run.  <br>• Changes HDFS permissions to 777 on the import directories.  <br>• May trigger downstream processes that poll the status file or read the Hive tables. |
| **Assumptions** | • All required environment variables are defined in the properties file.  <br>• The script runs on a node with Hadoop, Hive, Impala, and Sqoop binaries in `$PATH`.  <br>• Oracle credentials are valid and network‑reachable.  <br>• The user executing the script has write access to HDFS, Hive, and the status file location. |

---

### 4. Integration Points (how it connects to other scripts/components)

| Connection | Description |
|------------|-------------|
| **Process‑status file** (`$Rate_Location_Sqoop_ProcessStatusFile`) | Shared with many other MNAAS scripts (e.g., `MNAAS_RawFileCount_Checks.sh`, `MNAAS_SimInventory_tbl_Load.sh`). Those scripts read the flag to decide whether to start, and this script updates it to indicate progress/completion. |
| **Hive tables** (`rate_*`, `location_*`) | Consumed by downstream analytics, reporting, or transformation scripts (e.g., `MNAAS_RAReports.sh`, `MNAAS_SimChange_tbl_Load.sh`). |
| **Impala refresh** | Guarantees that any Impala‑based queries launched by other batch jobs see the latest data. |
| **Email/SDP ticket** | Notifies the Move Development team (`$MOVE_DEV_TEAM`) and Cloudera support; other monitoring tools may watch the mailbox for failure alerts. |
| **Cron scheduler** | Typically invoked from a daily cron entry (e.g., `0 2 * * * /path/MNAAS_Sqoop.sh`). The script itself checks for concurrent instances to avoid overlapping runs. |
| **Properties file** | Centralised configuration used by the whole MNAAS suite; changes here affect all scripts that source it. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Concurrent execution** – two instances could corrupt HDFS directories or status file. | The script already checks process count; enforce a lock file (`flock`) in addition to the `ps` guard. |
| **Credential exposure** – Oracle password stored in plain text in the properties file. | Move passwords to a secure vault (e.g., HashiCorp Vault) and retrieve at runtime, or use Oracle wallet. |
| **Partial data load** – Sqoop succeeds but Hive load fails, leaving stale files. | Add a cleanup step that removes HDFS files on Hive load failure; consider using Hive `INSERT OVERWRITE` instead of truncate+load. |
| **Log file growth** – Daily log files are never rotated if the script fails before the `rm -rf` line. | Implement log rotation (e.g., `logrotate`) or always truncate the log at start. |
| **Network/DB outage** – Sqoop import may hang or time‑out. | Set Sqoop `--connect-timeout` and `--fetch-size`; add a timeout wrapper (`timeout` command) around the Sqoop call. |
| **Email/SDP flood** – Repeated failures could generate many tickets. | The status flag `MNAAS_email_sdp_created` prevents duplicate tickets; ensure it is reset on the next successful run. |
| **Permission changes (chmod 777)** – Overly permissive HDFS permissions. | Use a more restrictive ACL (e.g., 750) and grant required groups only. |

---

### 6. Running / Debugging the Script

| Step | Command / Action |
|------|-------------------|
| **Manual execution** | ```bash /app/hadoop_users/MNAAS/MNAAS_Sqoop.sh``` (run as the same user that cron uses). |
| **Enable verbose logging** | The script already starts with `set -x`; you can also export `HADOOP_ROOT_LOGGER=DEBUG,console` before running. |
| **Check prerequisite variables** | ```grep -E 'Rate_|Location_|OrgDetails_|IMPALAD_HOST' /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties``` |
| **Inspect status file** | ```cat $Rate_Location_Sqoop_ProcessStatusFile``` – verify flag values before/after run. |
| **Force a specific step** | Edit the status file to set `MNAAS_Daily_ProcessStatusFlag=1` (run Rate only) or `=2` (run Location only) and re‑execute. |
| **View logs** | ```less $Rate_Location_table_SqoopLogName$(date +_%F)``` – contains timestamps for each sub‑step. |
| **Simulate failure** | Introduce a syntax error in the query variable or temporarily rename the Hive table; verify that an email and SDP ticket are generated. |
| **Check for concurrent runs** | ```ps -ef | grep MNAAS_Sqoop.sh``` – ensure only one instance is active. |
| **Validate Impala refresh** | After successful run, run `impala-shell -i $IMPALAD_HOST -q "describe $dbname.$rate_table_name"` to confirm metadata is up‑to‑date. |

---

### 7. External Configuration & Environment Variables

| Variable (from properties) | Usage |
|----------------------------|-------|
| `Rate_Location_Sqoop_ProcessStatusFile` | Path to the shared status file that stores flags, timestamps, job status, and email‑sent flag. |
| `Rate_Location_table_SqoopLogName` | Base name for daily log files (date suffix added at runtime). |
| `Rate_table_Dir`, `Location_table_Dir` | HDFS directories where Sqoop writes raw files. |
| `Rate_table_Query`, `Location_table_Query` | Full Sqoop `--query` strings (including `$CONDITIONS`). |
| `OrgDetails_ServerName`, `OrgDetails_PortNumber`, `OrgDetails_Service`, `OrgDetails_Username`, `OrgDetails_Password` | Oracle JDBC connection details. |
| `dbname`, `rate_table_name`, `rate_temp_table_name`, `location_table_name` | Hive database and table identifiers. |
| `IMPALAD_HOST` | Impala daemon host for metadata refresh. |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM`, `SDP_Receipient_List` | Email routing for failure notifications and SDP ticket creation. |
| `MNAAS_Sqoop_Scriptname` | Expected to be set in the properties file; used in log and email subjects. |

If any of these are missing or empty, the script will fail early (e.g., Sqoop will reject an empty connection string).

---

### 8. Suggested Improvements (TODO)

1. **Secure Credential Management** – Replace plain‑text Oracle credentials with a vault‑based retrieval mechanism or Oracle wallet to reduce security exposure.
2. **Robust Concurrency Control** – Implement a lock file (`flock -n /tmp/MNAAS_Sqoop.lock`) at script start to guarantee single‑instance execution even if the `ps` guard is bypassed (e.g., during a rapid restart).

---