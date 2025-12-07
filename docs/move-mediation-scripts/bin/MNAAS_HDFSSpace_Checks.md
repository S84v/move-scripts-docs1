**File:** `move-mediation-scripts\bin\MNAAS_HDFSSpace_Checks.sh`  

---

## 1. High‑Level Summary
This script is a daily health‑check that monitors HDFS storage utilisation for the MNAAS mediation platform. It queries HDFS capacity, compares the used‑percentage against an 80 % threshold, logs the result, and – when the threshold is exceeded – sends an alert e‑mail (and, in production, creates an SDP ticket). The script also updates a shared *process‑status* file that other MNAAS daily jobs read to coordinate execution and to surface success/failure information.

---

## 2. Important Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| `HDFS_Space_validation` | Executes `hadoop fs -df -h`, parses the usage %; logs success/failure; sends alert e‑mail if usage > 80 %; updates status file flags. |
| `terminateCron_successful_completion` | Writes a “success” state to the process‑status file, records run‑time, logs termination, and exits with code 0. |
| `terminateCron_Unsuccessful_completion` | Logs failure, optionally triggers e‑mail/SDP ticket (only in PROD), records run‑time, and exits with code 1. |
| `email_and_SDP_ticket_triggering_step` | Marks the job as failed in the status file, checks whether an SDP ticket/e‑mail has already been generated, and if not, records ticket creation time and logs the action. |
| `email_and_SDP_ticket_triggering_step_validation` | Simple wrapper that only logs ticket creation (used only for validation paths). |
| Main block (bottom) | Guard against concurrent runs, reads the global flag (`MNAAS_Daily_ProcessStatusFlag`) from the status file, decides whether to invoke `HDFS_Space_validation`, and finally calls the appropriate termination routine. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Configuration files** | `MNAAS_ShellScript.properties` – generic MNAAS env vars (e.g., `MNAASConfPath`, `MNAASLocalLogPath`, `ENV_MODE`). <br> `MNAAS_HDFSSpace_Checks.properties` – script‑specific overrides (e.g., thresholds, mail recipients). |
| **Process‑status file** | `$MNAASConfPath/MNAAS_HDFSSpace_Checks_ProcessStatusFile` – plain‑text key/value file edited with `sed`. Flags used: `MNAAS_Daily_ProcessStatusFlag`, `MNAAS_Daily_ProcessName`, `MNAAS_job_status`, `MNAAS_email_sdp_created`, `MNAAS_job_ran_time`. |
| **External services** | - Hadoop HDFS (`hadoop fs -df -h`). <br> - Local syslog (`logger`). <br> - Mail subsystem (`mailx`). <br> - SDP ticketing system (invoked indirectly via `email_and_SDP_ticket_triggering_step`). |
| **Outputs** | - Log files under `$MNAASLocalLogPath/MNAAS_HDFSSpace_Checks_Log_YYYY-MM-DD`. <br> - Updated process‑status file. <br> - Alert e‑mail (and possibly SDP ticket). |
| **Assumptions** | - `hadoop` command is in `$PATH` and returns output in the format `Filesystem Size Used Available Use% Mounted on`. <br> - The status file exists and is writable by the script user. <br> - `mailx` is configured to send mail from `hdfs@mtthdoped01.intl.vsnl.co.in`. <br> - `ENV_MODE` is set to `PROD` for production runs. |
| **Side‑effects** | - Modifies shared status file (potential race condition if multiple jobs edit simultaneously). <br> - Generates e‑mail traffic and possibly creates external tickets. |

---

## 4. Integration with Other Scripts / Components  

| Connected Component | Interaction |
|---------------------|-------------|
| **Other MNAAS daily jobs** (e.g., `MNAAS_Daily_Validations_Checks.sh`, `MNAAS_Daily_Recon.sh`) | All read/write the same *process‑status* file to coordinate start/stop flags and to surface overall job health. |
| **Cron scheduler** | Typically invoked via a daily cron entry; the script contains its own guard (`ps aux | grep -w $MNAAS_HDFSSpace_Checks_ScriptName`) to avoid overlapping runs. |
| **SDP ticketing system** | Not called directly; the script only records that a ticket *should* be created (via the flag `MNAAS_email_sdp_created`). Down‑stream automation (e.g., a ticket‑creation daemon) watches this flag. |
| **Monitoring/Alerting platform** | The log file and e‑mail alerts are consumed by Ops monitoring dashboards (e.g., Splunk, ELK). |
| **Hadoop cluster** | The script runs on a node with Hadoop client installed; it queries the NameNode for capacity data. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Incorrect parsing of `hadoop fs -df -h` output** – format changes break the `awk`/`sed` extraction. | Add defensive parsing (e.g., verify that `$usage_percent` ends with `%` and fallback to `hdfs dfsadmin -report`). |
| **Concurrent edits to the status file** – race condition if two scripts modify the same file simultaneously. | Serialize access using `flock` or move to a lightweight DB (e.g., SQLite) for flag storage. |
| **Hard‑coded 80 % threshold** – may become unsuitable as data grows. | Externalize the threshold in the properties file and expose it as a configurable parameter. |
| **Mail delivery failure** – alerts silently lost. | Capture `mailx` exit code; on failure, write to a separate “mail‑failure” log and optionally retry. |
| **Script runs on a node without proper Hadoop client** – `hadoop` command not found. | Verify `$PATH` and `hadoop version` at start; abort with clear error if missing. |
| **SDP ticket duplication** – multiple failures could generate duplicate tickets. | Ensure the flag `MNAAS_email_sdp_created` is atomically checked/updated (e.g., via `sed -i` with a lock). |

---

## 6. Running & Debugging the Script  

1. **Typical execution (via cron)**  
   ```bash
   0 2 * * * /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_HDFSSpace_Checks.sh >> /dev/null 2>&1
   ```
2. **Manual run (debug mode)**  
   ```bash
   cd /app/hadoop_users/MNAAS/move-mediation-scripts/bin
   set -x   # enable shell tracing
   ./MNAAS_HDFSSpace_Checks.sh
   ```
   - Check the generated log file:  
     `tail -f $MNAASLocalLogPath/MNAAS_HDFSSpace_Checks_Log_$(date +%F)`.
   - Verify the status file contents after run:  
     `cat $MNAASConfPath/MNAAS_HDFSSpace_Checks_ProcessStatusFile`.

3. **Common troubleshooting steps**  
   - Ensure the property files exist and are readable.  
   - Run `hadoop fs -df -h` manually and confirm the output format.  
   - Test e‑mail sending: `echo test | mailx -s "test" -c "$cc_List" -r $from_mail_Id "$MOVE_DEV_TEAM"`.  
   - If the script exits with code 1, inspect the log for “terminated Unsuccessfully” messages and verify whether `ENV_MODE` is set to `PROD`.  

---

## 7. External Config / Environment Variables  

| Variable | Source | Purpose |
|----------|--------|---------|
| `MNAASConfPath` | `MNAAS_ShellScript.properties` | Directory containing the shared process‑status file. |
| `MNAASLocalLogPath` | `MNAAS_ShellScript.properties` | Base path for script‑specific log files. |
| `ENV_MODE` | `MNAAS_ShellScript.properties` | Determines whether SDP ticket creation is performed (`PROD` only). |
| `MOVE_DEV_TEAM` | `MNAAS_HDFSSpace_Checks.properties` (presumed) | E‑mail distribution list for alerts (`move-mediation-it-ops@tatacommunications.com,hadoop-dev@tatacommunications.com`). |
| `cc_List`, `from_mail_Id`, `recepient_mail_Id` | Hard‑coded in script (could be moved to properties) | E‑mail header fields. |
| `MNAAS_HDFSSpace_Checks_ProcessStatusFileName` | Constructed from `MNAASConfPath` | Shared flag file. |
| `MNAAS_HDFSSpace_Checks_LogPath` | Constructed from `MNAASLocalLogPath` | Log file location (date‑appended). |

---

## 8. Suggested TODO / Improvements  

1. **Add file‑locking** – wrap all reads/writes to the process‑status file with `flock` to eliminate race conditions.  
2. **Externalize thresholds & e‑mail parameters** – move `80` (usage limit), `cc_List`, `from_mail_Id`, and recipient lists into `MNAAS_HDFSSpace_Checks.properties` for easier tuning without script edits.  

---