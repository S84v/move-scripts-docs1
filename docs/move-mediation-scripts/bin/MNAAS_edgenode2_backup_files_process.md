**File:** `move-mediation-scripts/bin/MNAAS_edgenode2_backup_files_process.sh`

---

## 1. High‑Level Summary
This Bash driver extracts a five‑month slice of data from an Impala table (the table name is supplied as the sole argument), writes the result as a semi‑colon‑delimited CSV file, compresses it, and securely copies the archive to the Edge‑Node‑2 backup server. Execution is guarded by a PID‑check to avoid concurrent runs, and the script updates a shared *process‑status* file at each stage. On success the status file is marked “Success”; on any failure an SDP ticket and notification email are raised (in production only). The script is invoked by a cron job and logs all activity to a central log file.

---

## 2. Key Functions / Logical Blocks

| Function / Block | Responsibility |
|------------------|----------------|
| **Argument handling** | Validates a single argument `<Table_name>`; aborts if missing. |
| **`. MNAAS_bkp_files_process.properties`** | Sources configuration variables (paths, hosts, env mode, etc.). |
| **`load_backup_files()`** | *a)* Computes start/end dates (5‑month window). <br>*b)* Creates target folder. <br>*c)* Updates status flag to **1** (load phase). <br>*d)* Runs an `impala-shell` query to export rows to `$MNAAS_File_processed_location_ZIP/${folder_name}/$file_name`. <br>*e)* Logs success/failure and aborts on error. |
| **`SCP_backup_files_to_edgenode2()`** | *a)* Updates status flag to **2** (transfer phase). <br>*b)* Gzips the CSV file. <br>*c)* If `ENV_MODE=PROD`, copies the `.gz` file via `scp` to `$backup_server`. <br>*d)* On success removes the local archive; on failure aborts. |
| **`terminateCron_successful_completion()`** | Resets status flags to **0**, marks job status *Success*, records run time, writes final log entries, and exits 0. |
| **`terminateCron_Unsuccessful_completion()`** | Logs failure, triggers email/SDP ticket (production only), records run time, and exits 1. |
| **`email_and_SDP_ticket_triggering_step()`** | Checks if an email/SDP ticket has already been created (via flag in status file). If not, composes a simple mail using `mailx` and updates the status file flags. |
| **PID‑guard / Main program** | Reads previous PID from status file, verifies the process is not running, writes its own PID, decides which phases to run based on the current `MNAAS_Monthly_ProcessStatusFlag` (0/1 → load + transfer, 2 → transfer only). Handles mismatched flags or concurrent runs as failures. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Input arguments** | `<Table_name>` – exact Impala table name (e.g. `mnaas_customer_traffic`). |
| **Configuration files** | `MNAAS_bkp_files_process.properties` (defines all environment‑specific variables). |
| **External services** | - **Impala** (`impala-shell` on `$IMPALAD_HOST`). <br>- **SCP target** (`$backup_server`). <br>- **Mail system** (`mailx`). |
| **Generated files** | - CSV: `mnaas_<table>_<hhYYYY>.csv` in `$MNAAS_File_processed_location_ZIP/<folder_name>/`. <br>- GZIP archive: same name with `.gz`. |
| **Log output** | Appended to `$MNAAS_backup_files_logpath`. |
| **Process‑status file** | `$MNAAS_files_backup_ProcessStatusFile` – multiple key/value pairs (flags, timestamps, PID, job status, email‑sent flag). |
| **Side effects** | - Updates status file flags at each stage. <br>- Sends email & creates SDP ticket on failure (prod only). <br>- Removes local `.gz` after successful transfer. |
| **Assumptions** | - The property file exists and contains valid paths/hosts. <br>- Impala table is partitioned on `partition_date`. <br>- The script runs on a host with `impala-shell`, `gzip`, `scp`, `mailx` installed. <br>- `$ENV_MODE` is set to `PROD` or another value to control ticketing. <br>- Sufficient disk space for CSV and gzip files. |

---

## 4. Interaction with Other Components

| Component | Connection Point |
|-----------|------------------|
| **Edge‑Node‑1 backup script** (`MNAAS_edgenode1_bkp_files_process.sh`) | Mirrors the same workflow for a different edge node; both scripts share the same status file and log path. |
| **Up‑stream data generation** | Impala tables are populated by the Mediation layer (e.g., CDR ingestion pipelines). This script merely extracts a slice; any change in table schema or partitioning must be reflected here. |
| **Down‑stream consumption** | The `.gz` files landed on `$backup_server` are later consumed by archival or analytics jobs (not shown). |
| **SDP ticketing / monitoring** | Email and ticket creation integrate with the internal SDP incident system; the script updates the status file which may be polled by monitoring dashboards. |
| **Cron scheduler** | Typically invoked via a daily/weekly cron entry; the PID‑guard prevents overlapping runs. |
| **Shared status file** | `$MNAAS_files_backup_ProcessStatusFile` is a central coordination point for multiple backup scripts (edge‑node‑1, edge‑node‑2, possibly others). |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Stale PID / orphaned process** | New run blocked indefinitely. | Implement a lockfile with timeout; verify the PID is still alive before aborting. |
| **Impala query failure (e.g., schema change)** | No CSV generated → downstream jobs fail. | Add schema validation step; capture and log the exact query error; alert on non‑zero row count. |
| **Insufficient disk space** | Script aborts during CSV creation or gzip. | Pre‑run disk‑space check; rotate/clean old files; monitor `$MNAAS_File_processed_location_ZIP`. |
| **SCP transfer failure** | Backup not delivered; data loss risk. | Retry logic with exponential back‑off; checksum verification after copy. |
| **Status‑file corruption** | Flags become inconsistent, causing incorrect flow. | Use atomic `sed` replacements or write to a temp file then `mv`; backup the status file before each run. |
| **Email/SDP ticket spam** | Repeated failures generate many tickets. | Rate‑limit ticket creation; ensure the “email_sent” flag is reliably persisted. |
| **Hard‑coded date window** (always 5‑month offset) | May miss required data if business rule changes. | Externalize the window size into the properties file. |

---

## 6. Running / Debugging the Script

### Normal Execution
```bash
# Example: process the table mnaas_customer_traffic
./MNAAS_edgenode2_backup_files_process.sh mnaas_customer_traffic
```
- The script must be run by the user that owns the Hadoop/HDFS environment (typically `hadoop` or a service account).
- It is normally scheduled via cron; ensure the environment loads the property file (absolute path is hard‑coded).

### Debug Steps
1. **Enable Verbose Logging** – The script already runs `set -x`; you can increase log detail by adding `export BASH_XTRACEFD=1` before invocation.
2. **Tail the Log**  
   ```bash
   tail -f $MNAAS_backup_files_logpath
   ```
3. **Inspect the Status File**  
   ```bash
   cat $MNAAS_files_backup_ProcessStatusFile
   ```
   Verify flags, PID, and timestamps.
4. **Manual Impala Test** – Run the generated query directly:
   ```bash
   impala-shell -i $IMPALAD_HOST -q "SELECT * FROM mnaas.<Table_name> WHERE partition_date >= '2023-01-01' AND partition_date <= '2023-01-31';"
   ```
5. **Check SCP Connectivity**  
   ```bash
   scp -v $MNAAS_File_processed_location_ZIP/<folder>/<file>.gz $backup_server:/tmp/
   ```
6. **Force a Failure** – Temporarily set `ENV_MODE=DEV` to skip the SCP step and verify the email/ticket path is not triggered.

---

## 7. External Configuration & Environment Variables

| Variable (from properties) | Purpose |
|----------------------------|---------|
| `MNAAS_backup_files_logpath` | Path to the script’s log file (stderr redirected). |
| `MNAAS_File_processed_location_ZIP` | Base directory where CSV and gz files are written. |
| `MNAAS_files_backup_ProcessStatusFile` | Central status/flag file shared across backup scripts. |
| `backup_server` | Hostname/IP of the Edge‑Node‑2 server receiving the gzip archive. |
| `IMPALAD_HOST` | Impala daemon host for `impala-shell`. |
| `ENV_MODE` | Controls production vs. non‑production behavior (ticket/email). |
| `MNAAS_backup_enode2_files_Scriptname` | Human‑readable script identifier used in logs and tickets. |
| `SDP_ticket_cc_email` | CC list for the incident email. |
| `MNAAS_job_status`, `MNAAS_email_sdp_created`, etc. | Keys inside the status file that the script updates. |

*Note:* The property file is sourced with `.` (dot) which imports all variables into the script’s environment. If any of these variables are missing or malformed, the script will fail early (e.g., `sed` on a non‑existent file).

---

## 8. Suggested Improvements (TODO)

1. **Replace PID‑guard with a robust lockfile** – Use `flock` or a lock directory with a timeout to avoid stale PID issues and simplify concurrency handling.
2. **Add retry & checksum verification for SCP** – After copy, compute an MD5/SHA256 on both source and destination (via a remote command) and retry up to 3 times before aborting.

---