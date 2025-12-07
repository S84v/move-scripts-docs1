**MNAAS_Load_SOTA_Table.sh – High‑Level Documentation**

---

### 1. Purpose & Production Summary
`MNAAS_Load_SOTA_Table.sh` is a daily batch job that enriches AI/ML‑derived “best‑window” records with SIM, VIN and SECS information (joined via IMSI), writes the enriched rows to a temporary Hive table, generates a flat‑file using a Java JAR, loads that file into the final API SOTA Hive table, and updates a shared process‑status property file. It also handles success/failure logging, process‑locking, and automatic email/SDP ticket creation on errors.

---

### 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **enrich_raw_data** | Truncates the temp API table, runs an Impala query that joins `traffic_details_raw_daily`, `kyc_snapshot`, and the AI‑predicted best‑window table to produce enriched rows, and inserts them into `${api_best_window_sota_temp_tbl}`. |
| **generate_file_for_load** | Invokes a Java JAR (`$sota_load_jar`) with a property file to convert the Hive temp table into a delimited load file (`$sota_load_file`). Sets file permissions. |
| **load_main_table** | If the load file exists, truncates the final API table, copies the file to HDFS, loads it into Hive (`${api_best_window_sota_tbl}`), and refreshes the Impala metadata. |
| **terminateCron_successful_completion** | Resets process‑status flags to *Success*, records run time, logs completion, and exits 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, updates run‑time flag, triggers email/SDP ticket, and exits 1. |
| **email_and_SDP_ticket_triggering_step** | Sends a failure notification email (using `mail`) and marks that an SDP ticket has been created to avoid duplicate alerts. |
| **Main Program** | Implements a lock‑file mechanism via a PID stored in the status file, reads the current process flag, and executes the appropriate subset of the three stages (enrich → generate → load) based on that flag. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_SOTA_Load.properties` – defines all `$` variables used (DB names, table names, file paths, hostnames, email lists, etc.). |
| **External Services** | - Hive (via `hive -e`) <br> - Impala (via `impala-shell`) <br> - HDFS (`hdfs dfs`) <br> - Java runtime (JAR execution) <br> - System logger (`logger`) <br> - Mail utility (`mail`) |
| **Primary Input Data** | - AI/ML best‑window prediction table `${pgw_best_window_pred_tbl}` <br> - Raw traffic details `mnaas.traffic_details_raw_daily` <br> - KYC snapshot `mnaas.kyc_snapshot` |
| **Generated Artifacts** | - Enriched Hive temp table `${api_best_window_sota_temp_tbl}` <br> - Load file `$sota_load_file` (CSV/TSV) <br> - HDFS copy `$hdfs_enrich_file` <br> - Final Hive table `${api_best_window_sota_tbl}` |
| **Process‑Status File** | `$sota_table_load_ProcessStatusFileName` – a simple key‑value file tracking PID, flags, job status, timestamps, and email‑sent flag. |
| **Logs** | `${sota_load_LogPath}` with daily log file name (`*_YYYY-MM-DD`). |
| **Side Effects** | - Updates process‑status file (flags, timestamps). <br> - Sends email on failure. <br> - May create an SDP ticket (external ticketing system, not shown). |
| **Assumptions** | - All Hive/Impala tables exist and are accessible to the script’s Hadoop user. <br> - HDFS paths have sufficient space and correct permissions. <br> - Java JAR is compatible with the runtime environment. <br> - Mail server is reachable and `mail` command works. <br> - The status property file is writable by the script. |

---

### 4. Interaction with Other Scripts / Components

| Connected Component | Interaction Detail |
|---------------------|--------------------|
| **MNAAS_SOTA_Load.properties** | Centralised configuration used by multiple SOTA‑related jobs (e.g., other load or validation scripts). |
| **MNAAS_GBS_Load.sh**, **MNAAS_IMEIChange_tbl_Load.sh**, etc. | Share the same process‑status file pattern and logging conventions; may run in the same nightly window, competing for Hive/Impala resources. |
| **AI/ML Prediction Pipeline** | Populates `${pgw_best_window_pred_tbl}` earlier in the day; this script consumes that table. |
| **Downstream API Services** | Consume `${api_best_window_sota_tbl}` via Impala/ Hive for real‑time API responses. |
| **SDP Ticketing System** | Triggered indirectly via `email_and_SDP_ticket_triggering_step` (the script only sends an email; the ticket is created by an external monitoring process that watches that email). |
| **Cron Scheduler** | The script is scheduled once per day (crontab entry not shown) and relies on the lock‑mechanism to avoid overlapping runs. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Hive/Impala query failure** (e.g., schema change) | Incomplete enrichment, downstream API errors | Add explicit schema validation before query; capture error output to a separate alert file. |
| **Java JAR execution error** (missing JAR, OOM) | No load file → job aborts | Verify JAR checksum at start; monitor JVM memory; add retry logic with exponential back‑off. |
| **HDFS permission or space shortage** | Load step fails, job stops | Pre‑run HDFS health check (disk usage, write permission); alert if free space < threshold. |
| **Stale PID lock** (script crashed leaving PID in status file) | Subsequent runs blocked | Implement a timeout check (e.g., if PID older than X hours, clear lock). |
| **Email/SDP ticket flood** (multiple failures generate many tickets) | Alert fatigue | Ensure `MNAAS_email_sdp_created` flag is reliably reset only after ticket closure; consider rate‑limiting. |
| **Property file corruption** | All variables become undefined → script crash | Store a backup copy; validate required keys at start; exit with clear error if missing. |

---

### 6. Running & Debugging the Script

1. **Manual Invocation**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Property_Files
   . MNAAS_SOTA_Load.properties   # load env vars (optional, script does it)
   /path/to/move-mediation-scripts/bin/MNAAS_Load_SOTA_Table.sh
   ```
2. **Check Logs**  
   - Daily log: `${sota_load_LogPath}$(date +_%F)` (e.g., `/var/log/mnaas/sota_load_2024-12-04`).  
   - System logger entries (`/var/log/messages` or `journalctl`) for `logger -s` output.

3. **Debug Mode**  
   - The script already runs with `set -vx` (verbose + trace).  
   - To step through a specific function, comment out the main flow and call the function directly, e.g., `enrich_raw_data`.

4. **Validate Process Lock**  
   ```bash
   grep MNAAS_Script_Process_Id $sota_table_load_ProcessStatusFileName
   ps -p <PID> -o cmd=
   ```
   If the PID is stale, edit the status file to set `MNAAS_Script_Process_Id=0`.

5. **Simulate Failure**  
   - Force a non‑zero exit from a stage (e.g., `false` after the Hive command) to verify email/SDP ticket generation.

6. **Dependency Checks**  
   ```bash
   which hive impala-shell hdfs java mail logger
   ```

---

### 7. External Configurations & Environment Variables

| Variable (from properties) | Meaning |
|----------------------------|---------|
| `sota_table_load_ProcessStatusFileName` | Path to the shared key‑value status file. |
| `sota_load_LogPath` | Directory for daily log files. |
| `dbname` | Hive database name (e.g., `mnaas`). |
| `api_best_window_sota_temp_tbl` | Temp Hive table for enriched data. |
| `api_best_window_sota_tbl` | Final Hive table consumed by the API. |
| `pgw_best_window_pred_tbl` | Source AI/ML prediction table. |
| `IMPALAD_HOST` | Impala daemon host for metadata refresh. |
| `sota_load_file` | Local file generated by the Java JAR. |
| `sota_load_jar` / `sota_load_prop_file` | Java JAR and its property file used for file generation. |
| `hdfs_enrich_path` / `hdfs_enrich_file` | HDFS destination directory and file name for the load file. |
| `ccList`, `GTPMailId` | Email recipients for failure notifications. |
| `ppu_daily_report_Scriptname`, `sota_load_Scriptname` | Script name strings used in log messages. |

*All of the above are defined in `MNAAS_SOTA_Load.properties`. The script assumes they are present and correctly formatted.*

---

### 8. Suggested Improvements (TODO)

1. **Add Idempotent Cleanup** – Before truncating tables or removing HDFS files, verify existence and optionally archive previous data (e.g., copy to a backup HDFS path). This protects against accidental data loss during a partial run.

2. **Implement Centralised Error Handling** – Wrap each external command (`hive`, `impala-shell`, `hdfs`, `java`) in a helper function that logs the command, captures stdout/stderr, and returns a standardized error code. This will simplify debugging and make the script more maintainable.