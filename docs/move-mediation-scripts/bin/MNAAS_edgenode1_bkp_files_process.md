**File:** `move-mediation-scripts/bin/MNAAS_edgenode1_bkp_files_process.sh`  

---

## 1. High‑Level Summary
`MNAAS_edgenode1_bkp_files_process.sh` is a Bash driver that orchestrates the retrieval, validation, and loading of a bounded set of backup files (e.g., *actives_raw_daily*, *traffic_raw_daily*, etc.) from a remote “edge‑node‑1” backup server into the Hadoop/Impala data‑lake.  
The script:

1. **Guards against concurrent runs** using a PID stored in a shared process‑status file.  
2. **Copies** the most recent N zipped backup files that fall within a user‑supplied month range (`startdate_filter` – `enddate_filter`).  
3. **Unzips**, adjusts permissions, and **loads** the files into a temporary HDFS staging directory.  
4. **Triggers a Spark job** (via `spark-submit`) that transforms the raw files into a backup table.  
5. **Invalidates Impala metadata** and cleans up local copies.  
6. **Updates** the process‑status file and, on failure, sends an email and creates an SDP ticket.

The script is invoked by a cron job (or manually) with three arguments: the target table name, a start month, and an end month.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|-----------------|
| `SCP_backup_files_to_edgenode1` | *Phase 1*: Selects up‑to‑`$No_of_files_to_process` zipped files from `$MNAAS_File_processed_location_ZIP/<folder_name>` on the remote `$backup_server` whose embedded month lies between the supplied start/end dates, copies them via `scp`, and gunzips them locally under `$MNAAS_File_processed_location_enode1/<folder_name>`. Updates the process‑status flag to **1**. |
| `load_files_into_backup_table` | *Phase 2*: Creates/clears the HDFS staging path `$MNAAS_Rawtablesload_PathName/`, copies the unzipped CSVs into HDFS, runs the Spark job `$MNAAS_Script_name` with arguments `<table_name> $MNAAS_Rawtablesload_PathName/`, and on success removes local files, runs `impala-shell` to invalidate metadata, and logs success. Updates the process‑status flag to **2**. |
| `terminateCron_successful_completion` | Writes a *Success* state to the process‑status file (flags = 0, job_status = Success, email_sdp_created = No, timestamps) and exits with code 0. |
| `terminateCron_Unsuccessful_completion` | Writes a *Failure* state, triggers `email_and_SDP_ticket_triggering_step` when `ENV_MODE=PROD`, logs, and exits with code 1. |
| `email_and_SDP_ticket_triggering_step` | Sends a pre‑formatted email (via `mailx`) to the support mailbox and marks `MNAAS_email_sdp_created=Yes` to avoid duplicate tickets. Also updates `MNAAS_job_status=Failure`. |
| **Main program** (bottom of file) | PID guard: reads previous PID from `$MNAAS_files_backup_ProcessStatusFile`, ensures no overlapping run, then decides which phase(s) to execute based on the current `MNAAS_Monthly_ProcessStatusFlag` (0/1 → copy + load, 2 → load only). |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Command‑line arguments** | 1️⃣ `table_name` (e.g., `actives_raw_daily`)  <br>2️⃣ `startdate_filter` (month in `MMMYYYY` format, e.g., `FEB2020`) <br>3️⃣ `enddate_filter` (same format) |
| **External configuration** | Sourced via `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_bkp_files_process.properties`. This file defines all `$MNAAS_*` variables used throughout the script (paths, server names, flags, log locations, Spark script name, etc.). |
| **Environment variables** | `ENV_MODE` (expected values: `PROD` or others) – controls whether remote copy and ticketing are performed. |
| **External services / resources** | - Remote backup server (`$backup_server`) reachable via SSH/SCP.<br>- HDFS (`hadoop fs` commands).<br>- Spark/YARN cluster (`spark-submit`).<br>- Impala (`impala-shell`).<br>- Local filesystem (log dir, staging dir).<br>- Mail system (`mailx`). |
| **Outputs** | - Copied & unzipped CSV files under `$MNAAS_File_processed_location_enode1/<folder_name>`.<br>- HDFS files under `$MNAAS_Rawtablesload_PathName/`.<br>- Process‑status file (`$MNAAS_files_backup_ProcessStatusFile`) updated with flags, timestamps, job status.<br>- Log file (`$MNAAS_backup_files_logpath`).<br>- Optional email / SDP ticket on failure. |
| **Side effects** | - Changes file permissions (`chmod 777`).<br>- Removes any pre‑existing files in the HDFS staging path.<br>- Invalidates Impala metadata for the target backup table.<br>- May generate a ticket in the SDP system (via email). |
| **Assumptions** | - The property file exists and defines all required variables.<br>- The remote backup server’s directory structure matches `$MNAAS_File_processed_location_ZIP/<folder_name>`.<br>- Files are named `<prefix>_<MMYYYY>.gz` where the month part can be extracted as shown.<br>- The Spark script (`$MNAAS_Script_name`) is idempotent and can be run on the staging path.<br>- The user executing the script has SSH keys set up for password‑less access to `$backup_server` and sufficient HDFS permissions. |

---

## 4. Interaction with Other Scripts / Components

| Component | Relationship |
|-----------|--------------|
| **Other `MNAAS_create_customer_cdr_*` scripts** | Those scripts generate the original CDR files that eventually become the backup files copied here. This script consumes the *backup* versions of those CDRs. |
| **`MNAAS_bkp_files_process.properties`** | Central configuration shared across all backup‑processing drivers (e.g., traffic, actives, tolling). |
| **`MNAAS_Script_name` (Spark job)** | The actual transformation logic for the backup table; likely a Scala/Java Spark application that mirrors the logic of the corresponding CDR creation scripts. |
| **Process‑status file (`$MNAAS_files_backup_ProcessStatusFile`)** | Shared state used by multiple backup‑processing scripts to coordinate phases and avoid concurrency. |
| **Cron scheduler** | Typically invoked nightly/weekly; the PID guard ensures only one instance runs at a time across the whole backup‑processing suite. |
| **SDP ticketing / mail system** | Integrated via `email_and_SDP_ticket_triggering_step`; other scripts use the same pattern for failure notification. |
| **HDFS / Impala** | The staging directory and backup tables are common data‑lake objects used by downstream reporting/analytics pipelines. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Remote server unreachable or SSH key expired** | No files copied → job stalls or fails. | Monitor SSH connectivity; rotate keys regularly; add retry logic around `scp`. |
| **Incorrect month parsing (date format mismatch)** | Files outside the intended window may be processed or required files missed. | Validate `startdate_filter`/`enddate_filter` format early; log parsed epoch values; add unit tests for date extraction. |
| **Large file volume exceeding disk space on edge node** | `scp` succeeds but local unzip fails, causing partial loads. | Pre‑check free space (`df`) before copy; enforce a maximum total size; clean up stale files. |
| **Concurrent runs due to stale PID in status file** | Duplicate processing, race conditions, corrupted tables. | Ensure PID is cleared on both success and failure paths; add a timeout check to purge stale entries. |
| **Spark job failure (code != 0)** | Backup table not refreshed → downstream reports stale. | Capture Spark logs; alert on failure; consider automatic retry with back‑off. |
| **Impala metadata invalidation failure** | Queries may read stale data. | Verify `impala-shell` exit code; retry or fallback to `refresh` command. |
| **Email/SDP ticket flood** | Repeated failures generate many tickets. | Guard with `MNAAS_email_sdp_created` flag (already present); add rate‑limiting. |
| **Hard‑coded `chmod 777`** | Overly permissive file permissions. | Review security policy; replace with least‑privilege mode (e.g., `750`). |

---

## 6. Running / Debugging the Script

### Typical Invocation
```bash
# Example: process actives backup files from Feb 2020 through Feb 2021
./MNAAS_edgenode1_bkp_files_process.sh actives_raw_daily FEB2020 FEB2021
```

### Prerequisites
1. Ensure the property file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_bkp_files_process.properties` is present and readable.
2. Verify SSH key‑based access to `$backup_server`.
3. Confirm HDFS and Impala connectivity for the user running the script.
4. Check that the log directory (`$MNAAS_backup_files_logpath`) is writable.

### Debug Steps
| Step | Action |
|------|--------|
| **Enable Verbose** | The script already runs `set -x`; you can also export `BASH_DEBUG=1` to increase output. |
| **Tail the log** | `tail -f $MNAAS_backup_files_logpath` while the job runs. |
| **Inspect Process‑Status File** | `cat $MNAAS_files_backup_ProcessStatusFile` to see current flags, PID, timestamps. |
| **Force a specific phase** | Manually set `MNAAS_Monthly_ProcessStatusFlag=1` (copy + load) or `=2` (load only) before execution to test each branch. |
| **Simulate failure** | Introduce a non‑existent file in the remote directory and observe the email/SDP ticket generation. |
| **Check Spark logs** | Spark driver logs are under YARN’s application UI; locate the application ID printed by `spark-submit`. |
| **Validate HDFS content** | `hadoop fs -ls $MNAAS_Rawtablesload_PathName/` before and after the run. |
| **Verify Impala metadata** | `impala-shell -i $IMPALAD_HOST -q "show tables like 'mnaas.${table_name}_bkp'"`. |

---

## 7. External Config / Environment Variables

| Variable (defined in property file) | Purpose |
|-------------------------------------|---------|
| `MNAAS_backup_files_logpath` | Path to the script’s log file. |
| `MNAAS_files_backup_ProcessStatusFile` | Shared status file used for PID guard and flag management. |
| `ENV_MODE` | Determines whether production‑only actions (remote copy, ticketing) are executed. |
| `backup_server` | Hostname/FQDN of the remote edge‑node‑1 backup server. |
| `MNAAS_File_processed_location_ZIP` | Remote directory containing zipped backup files. |
| `No_of_files_to_process` | Maximum number of files to copy per run. |
| `MNAAS_File_processed_location_enode1` | Local staging directory for unzipped CSVs. |
| `MNAAS_Rawtablesload_PathName` | HDFS staging path for Spark ingestion. |
| `MNAAS_Script_name` | Full path to the Spark application JAR / Python script. |
| `IMPALAD_HOST` | Impala daemon host for metadata invalidation. |
| `MNAAS_backup_enode1_files_Scriptname` | Human‑readable name used in logs / tickets. |
| `MNAAS_job_status`, `MNAAS_email_sdp_created`, etc. | Flags stored in the status file. |

If any of these variables are missing or empty, the script will fail early (e.g., `ssh`/`scp` with empty host, `hadoop fs` with empty path). Validation of the property file is therefore a prerequisite.

---

## 8. Suggested Improvements (TODO)

1. **Add Robust Parameter Validation** – Verify that `startdate_filter` and `enddate_filter` match the expected `MMMYYYY` pattern and that `startdate <= enddate`. Exit with a clear error message if validation fails.

2. **Introduce Retry & Back‑off Logic for Remote Operations** – Wrap `ssh`/`scp` calls in a loop with exponential back‑off (max 3 attempts). Log each retry and surface a distinct error code for persistent connectivity issues.

*(Both changes improve reliability and make troubleshooting easier without altering existing business logic.)*