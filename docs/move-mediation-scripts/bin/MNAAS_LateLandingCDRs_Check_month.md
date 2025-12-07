**File:** `move-mediation-scripts\bin\MNAAS_LateLandingCDRs_Check_month.sh`  

---

## 1. Purpose (one‑paragraph summary)

This script is a monthly validation job that scans the `mnaas.traffic_details_raw_daily` Impala table for “late‑landing” Call Detail Records (CDRs) belonging to the previous month. It extracts any offending rows into a CSV file, emails the report to a configurable recipient list, and updates a shared process‑status file used by the broader MNAAS (Move‑Mediation‑As‑A‑Service) orchestration framework. The script also logs its activity, creates an SDP ticket on failure (in production), and enforces a single‑instance execution model via a PID‑style check.

---

## 2. Key Functions / Logical Blocks

| Function / Block | Responsibility |
|------------------|----------------|
| **Source properties** (`. /app/hadoop_users/.../MNAAS_LateLandingCDRs_Check.properties`) | Loads environment‑specific variables (paths, flags, `ENV_MODE`, `setparameter`, etc.). |
| **Late_Landing_CDRs_Check** | *Core logic*: refreshes Impala metadata, runs a count query, extracts matching CDR rows to `$output_file_final`, sets file permissions, sends an email with attachment if rows exist, updates the process‑status file with success flags. |
| **terminateCron_successful_completion** | Resets the daily process flag, writes final status (`Success`) and runtime timestamp to the status file, logs termination, and exits 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, optionally triggers `email_and_SDP_ticket_triggering_step` when `ENV_MODE=PROD`, logs end‑time, and exits 1. |
| **email_and_SDP_ticket_triggering_step** | Marks the job as `Failure` in the status file, checks whether an SDP ticket/email has already been generated (`MNAAS_email_sdp_created` flag), and if not, updates the flag, timestamps, and logs ticket creation. |
| **email_and_SDP_ticket_triggering_step_validation** | Simple wrapper that only logs ticket creation (used for validation paths). |
| **Main guard** (`if [ \`ps aux| grep -w $MNAAS_LateLandingCDRs_Check_ScriptName | wc -l\` -le 3 ]; then …`) | Ensures only one instance runs; reads the `MNAAS_Daily_ProcessStatusFlag` from the status file, decides whether to invoke the core check or abort, and finally logs a “process already running” message if another instance is detected. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration / Env** | `MNAAS_LateLandingCDRs_Check.properties` (defines `MNAASConfPath`, `MNAASLocalLogPath`, `ENV_MODE`, `setparameter`, etc.). |
| **External Services** | - **Impala** (`impala-shell -i 192.168.124.93`) – runs two queries (count & select). <br> - **SMTP** via `mailx` – sends report email. <br> - **System logger** (`logger`). |
| **Files read** | - Process‑status file: `$MNAASConfPath/MNAAS_LateLandingCDRs_Check_month_ProcessStatusFile` <br> - Property file (source). |
| **Files written** | - CSV report: `/app/hadoop_users/MNAAS/MNAAS_Configuration_Files/LateLanding_CDRs/LateLanding_CDRs_for_month${current_month}.csv` (chmod 777). <br> - Log entries appended to `$MNAASLocalLogPath/MNAAS_LateLandingCDRs_Check_month_Log_$(date +_%F)`. <br> - Updated process‑status file (multiple `sed` in‑place edits). |
| **Side effects** | - Sends email with attachment (if rows exist). <br> - May trigger an SDP ticket creation (via external ticketing system – not shown in script). |
| **Assumptions** | - Impala host `192.168.124.93` reachable and `traffic_details_raw_daily` table present. <br> - `mailx` configured with a working MTA. <br> - The status file exists and is writable by the script user. <br> - `setparameter` variable expands to required environment settings (e.g., `ENV_MODE`). <br> - The script runs under a user with permission to `chmod 777` the output CSV. |

---

## 4. Interaction with Other Components

| Component | Relationship |
|-----------|--------------|
| **MNAAS_LateLandingCDRs_Check.sh** (daily version) | Shares the same status file naming convention (`MNAAS_LateLandingCDRs_Check_*_ProcessStatusFile`). Both scripts update the same flags, allowing the orchestration layer to know whether a daily or monthly validation is in progress. |
| **Process‑status framework** (`MNAAS_*.properties` + status files) | Centralised status tracking used by many MNAAS validation scripts (e.g., `MNAAS_Daily_Validations_Checks_*`). The flags (`MNAAS_Daily_ProcessStatusFlag`, `MNAAS_job_status`, `MNAAS_email_sdp_created`) are read/written by this script and other jobs to coordinate execution. |
| **SDP ticketing system** | Invoked indirectly via `email_and_SDP_ticket_triggering_step`; the script only toggles a flag, the actual ticket creation is performed by an external process that monitors the flag or by a downstream script. |
| **Cron scheduler** | The script is intended to be scheduled (likely monthly) via cron; the internal PID guard prevents overlapping runs. |
| **Logging infrastructure** | Writes to `$MNAASLocalLogPath/...` which is consumed by monitoring/alerting tools (e.g., Splunk, ELK). |
| **Mail distribution list** | Hard‑coded `recepient_mail_Id` and `cc_List`; other scripts may use the same addresses for consistency. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Hard‑coded email addresses** | Change‑management overhead; accidental exposure of personal addresses. | Externalise recipients to the properties file or a central config map; read from environment variables. |
| **`chmod 777` on CSV** | Over‑permissive file rights; potential data leakage. | Use a restrictive mode (e.g., `640`) and ensure the owning group has read access for downstream consumers. |
| **In‑place `sed` edits on shared status file** | Race conditions if multiple jobs modify the file simultaneously. | Implement file locking (`flock`) around status‑file updates or move to a transactional store (e.g., Zookeeper, DB). |
| **Impala query failures not captured** | Silent failure leading to missing alerts. | Capture exit codes of `impala-shell` commands; on non‑zero exit, invoke `terminateCron_Unsuccessful_completion`. |
| **Large result set causing memory/IO pressure** | Potential out‑of‑disk or long email send times. | Add a row‑limit warning, compress CSV before mailing, or switch to a file‑based transfer (SFTP) for very large reports. |
| **Single‑instance guard based on `ps` count** | May mis‑detect other processes with similar name; false positives/negatives. | Use a PID file (`/var/run/...`) with `flock` for reliable singleton enforcement. |
| **Missing/incorrect property file** | Script aborts early or uses default empty values. | Validate required variables after sourcing; abort with clear error message if any are unset. |

---

## 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure the property file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_LateLandingCDRs_Check.properties` is present and contains at least: `MNAASConfPath`, `MNAASLocalLogPath`, `ENV_MODE`, `setparameter`.  
   - Verify `impala-shell` can connect to `192.168.124.93` and the table `mnaas.traffic_details_raw_daily` exists.  
   - Confirm `mailx` is configured (test with a simple `mailx -s test user@example.com`).  

2. **Manual execution** (for testing)  
   ```bash
   # Switch to the script user (e.g., mnaas)
   cd /app/hadoop_users/MNAAS/move-mediation-scripts/bin
   ./MNAAS_LateLandingCDRs_Check_month.sh   # add -x for trace
   ```
   - Add `set -x` at the top of the script (or run `bash -x ./script.sh`) to see each command.  
   - Check the generated log file: `$MNAASLocalLogPath/MNAAS_LateLandingCDRs_Check_month_Log_$(date +_%F)`.  
   - Verify the CSV file exists and contains data (`head -n 5 $output_file_final`).  

3. **Cron integration**  
   - Typical cron entry (run on the 2nd day of each month at 02:00):  
     ```cron
     0 2 2 * * /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_LateLandingCDRs_Check_month.sh >> /dev/null 2>&1
     ```  
   - Ensure the cron user has read/write access to the status file and log directory.  

4. **Debugging failures**  
   - Look for `FAILURE` entries in the log file.  
   - If the script exits with code 1, check whether `ENV_MODE=PROD` triggered the SDP ticket step; confirm the ticketing system’s monitoring of the flag.  
   - Verify the status file’s flags (`grep MNAAS_` ...) to see the last recorded state.  

---

## 7. External Configurations & Environment Variables

| Variable | Source | Usage |
|----------|--------|-------|
| `MNAASConfPath` | `MNAAS_LateLandingCDRs_Check.properties` | Directory for the process‑status file. |
| `MNAASLocalLogPath` | `MNAAS_LateLandingCDRs_Check.properties` | Base path for log files. |
| `ENV_MODE` | `MNAAS_LateLandingCDRs_Check.properties` (or exported before run) | Determines whether SDP ticket creation is performed on failure (`PROD` triggers). |
| `setparameter` | `MNAAS_LateLandingCDRs_Check.properties` | Expected to contain additional `export` statements; evaluated via `eval $setparameter`. |
| `MNAAS_Daily_ProcessStatusFlag`, `MNAAS_job_status`, `MNAAS_email_sdp_created`, `MNAAS_job_ran_time` | Stored inside the status file; read/written by the script. |
| `from_mail_Id`, `recepient_mail_Id`, `cc_List` | Hard‑coded in script (should be moved to properties). |
| `output_file_final` | Constructed in script using `$current_month`. |

---

## 8. Suggested Improvements (TODO)

1. **Externalise all email parameters** – move `recepient_mail_Id`, `cc_List`, and `from_mail_Id` into the properties file or a dedicated email‑config JSON/YAML. This will simplify environment changes and avoid hard‑coded personal addresses.

2. **Add robust error handling for Impala commands** – capture the exit status of each `impala-shell` invocation; on failure, log the error, update the status file to `Failure`, and invoke the SDP ticket step immediately rather than proceeding to the mailing logic.  

---