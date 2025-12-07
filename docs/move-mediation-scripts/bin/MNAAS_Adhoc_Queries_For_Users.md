**File:** `move-mediation-scripts/bin/MNAAS_Adhoc_Queries_For_Users.sh`

---

## 1. High‑Level Summary
This Bash script is the scheduled driver for the “MNAAS Ad‑hoc Queries for Users” reporting pipeline. It (a) ensures only one instance runs at a time, (b) cleans the output directory of files older than two days, (c) invokes a Java class that generates the ad‑hoc user reports, (d) updates a shared process‑status file with execution flags, (e) logs all activity to a dedicated log file, and (f) on failure creates an SDP ticket and notifies the MOVE development team via email. The script is intended to be executed by a cron job (or manually) on a Hadoop‑gateway node.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|-----------------|
| `MNAAS_delete_files_in_the_output_dir_older_than_2_days` | Sets process‑status flag = 1, updates process name, deletes files in `$MNAAS_Adhoc_Queries_for_users_output_dir` older than 2 days, logs success/failure. |
| `MNAAS_adhoc_process` | Sets process‑status flag = 2, updates process name, runs the Java reporting class (`$MNAAS_adhoc_reports_for_users_classname`) with required property files, makes output files world‑readable, logs success/failure. |
| `terminateCron_successful_completion` | Resets status flag = 0, marks job status = Success, clears “email SDP created” flag, writes run timestamp, logs final messages, exits 0. |
| `terminateCron_Unsuccessful_completion` | Logs failure, calls `email_and_SDP_ticket_triggering_step`, exits 1. |
| `email_and_SDP_ticket_triggering_step` | Marks job status = Failure, checks if an SDP ticket has already been raised; if not, sends an email to the support mailbox and creates an SDP ticket via `mailx`, updates status file flags, logs ticket creation. |
| **Main program** (bottom of file) | Checks the PID stored in the status file; if no live process, writes its own PID, decides which steps to run based on the current flag (0/1 → clean + process, 2 → only process), otherwise logs that a previous run is still active and exits successfully. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration / Env** | Sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Adhoc_Queries_For_Users.properties`. Expected variables (non‑exhaustive):<br>• `MNAAS_Adhoc_Queries_for_users_LogPath` – absolute path to the log file (stderr is redirected here).<br>• `MNAAS_Adhoc_Queries_for_users_ProcessStatus_FileName` – shared status file (key/value).<br>• `MNAAS_Adhoc_Queries_for_users_output_dir` – directory where Java writes report files.<br>• `MNAAS_Adhoc_Queries_for_users_SriptName` – script name used in logs.<br>• `CLASSPATHVAR`, `MNAAS_Main_JarPath`, `MNAAS_adhoc_reports_for_users_classname` – Java runtime parameters.<br>• `MNAAS_Property_filename`, `MNAAS_Adhoc_Queries_Property_filename`, `MNASS_Adhoc_Temp_Scripts_filename` – additional property files passed to Java.<br>• Email‑related vars: `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM`. |
| **External Services** | • **Java runtime** – executes the reporting class.<br>• **Syslog (`logger`)** – writes informational messages.<br>• **Mail (`mail` / `mailx`)** – sends failure notifications and SDP tickets.<br>• **File system** – reads/writes status file, deletes old files, writes report files, updates permissions. |
| **Outputs** | • Report files (CSV/TSV/etc.) placed in `$MNAAS_Adhoc_Queries_for_users_output_dir`.<br>• Log file at `$MNAAS_Adhoc_Queries_for_users_LogPath` (stderr + explicit logger calls).<br>• Updated status file reflecting flags, PID, timestamps, job status, and ticket‑creation flag.<br>• Optional email to `insdp@tatacommunications.com` and/or `$GTPMailId` on failure. |
| **Side Effects** | • Deletes any file older than 2 days in the output directory (potential data loss if retention policy changes).<br>• Changes permissions of all output files to `777` (wide open).<br>• May generate duplicate SDP tickets if the status file is not correctly reset. |
| **Assumptions** | • The properties file exists and defines all referenced variables.<br>• The Java class runs without interactive prompts and returns 0 on success.<br>• The host has `mail`/`mailx` configured for outbound SMTP.<br>• The status file is writable by the script user and is the single source of truth for process coordination. |

---

## 4. Interaction with Other Components

| Component | Relationship |
|-----------|--------------|
| **Other MNAAS driver scripts** (e.g., `MNAAS_Actives_tbl_load_driver.sh`, `MNAAS_Activations_seq_check.sh`) | Share the same *process‑status* file pattern (`MNAAS_*_ProcessStatus_FileName`). Coordination is achieved by checking the `MNAAS_Script_Process_Id` entry; only one script of a given family runs at a time. |
| **Java reporting JAR** (`$MNAAS_Main_JarPath`) | Provides the core business logic for generating ad‑hoc queries. The script is a thin wrapper that supplies configuration files and handles orchestration. |
| **Cron scheduler** (likely defined in Azure Pipelines or a system crontab) | Triggers the script on a regular cadence (e.g., daily). The script’s self‑PID guard prevents overlapping runs. |
| **SDP ticketing system** (via email to `insdp@tatacommunications.com`) | Failure handling integrates with the internal incident management platform; the email body follows a templated format used by downstream ticket processors. |
| **Logging infrastructure** (syslog) | All `logger -s` calls are captured by the host’s syslog daemon, enabling centralized monitoring. |
| **Move‑Mediation pipelines** (Azure Pipelines YAML) | The script may be invoked as part of a larger CI/CD pipeline that provisions the environment, ensures property files are up‑to‑date, and validates output. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Stale PID / orphaned lock** – if the script crashes without clearing `MNAAS_Script_Process_Id`, subsequent runs will be blocked. | Production data not refreshed. | Implement a watchdog that checks the age of the PID entry; if older than a threshold, clear it and log a warning. |
| **Incorrect/missing property values** – undefined variables cause command failures or unintended deletions. | Job failure, possible data loss. | Validate required variables at script start; abort with clear error if any are empty. |
| **Broad file permissions (`chmod 777`)** – security exposure of report data. | Unauthorized access. | Use a more restrictive mode (e.g., `640`) and ensure the consuming group has read rights. |
| **Deletion of files older than 2 days** – may conflict with downstream processes that need longer retention. | Loss of data needed for audits. | Make the retention period configurable via a property; add a safeguard to skip deletion if a “retain” flag file exists. |
| **Email/SDP ticket flood** – repeated failures could generate many tickets. | Alert fatigue. | Add a back‑off counter in the status file (e.g., `MNAAS_failure_count`) and suppress ticket creation after N attempts until manual reset. |
| **Java class non‑idempotent** – if the Java process is re‑run without cleaning, duplicate rows may appear. | Corrupted reports. | Ensure the Java class overwrites existing files or include a pre‑run cleanup step. |

---

## 6. Running / Debugging the Script

### Normal Execution (Cron)
```bash
# Example crontab entry (run daily at 02:30)
30 2 * * * /app/hadoop_users/MNAAS/MNAAS_Scripts/bin/MNAAS_Adhoc_Queries_For_Users.sh
```

### Manual Run (for testing)
```bash
# Export a minimal set of variables if the properties file is unavailable
export MNAAS_Adhoc_Queries_for_users_LogPath=/tmp/mnaas_adhoc.log
export MNAAS_Adhoc_Queries_for_users_ProcessStatus_FileName=/tmp/mnaas_status.properties
export MNAAS_Adhoc_Queries_for_users_output_dir=/tmp/mnaas_output
# ... set other required vars ...

# Run with trace enabled (already set via `set -x` in script)
bash -x /app/hadoop_users/MNAAS/MNAAS_Scripts/bin/MNAAS_Adhoc_Queries_For_Users.sh
```

### Debugging Tips
1. **Check the status file** – Verify flags, PID, and timestamps:
   ```bash
   cat $MNAAS_Adhoc_Queries_for_users_ProcessStatus_FileName
   ```
2. **Inspect the log** – All `logger` output and `stderr` are appended to `$MNAAS_Adhoc_Queries_for_users_LogPath`.
3. **Validate Java execution** – Run the Java command manually with `-verbose:class` to ensure classpath is correct.
4. **Confirm email delivery** – Check `/var/log/maillog` (or equivalent) for `mail`/`mailx` activity.
5. **Process lock** – If the script reports “Previous … Process is running already”, verify with:
   ```bash
   ps -fp $(grep MNAAS_Script_Process_Id $MNAAS_Adhoc_Queries_for_users_ProcessStatus_FileName | cut -d= -f2)
   ```

---

## 7. External Configuration & Environment Variables

| Variable (from properties) | Purpose | Typical Source |
|----------------------------|---------|----------------|
| `MNAAS_Adhoc_Queries_for_users_LogPath` | Path for the script’s log file (stderr redirection). | `MNAAS_Adhoc_Queries_For_Users.properties` |
| `MNAAS_Adhoc_Queries_for_users_ProcessStatus_FileName` | Shared status/lock file used across MNAAS scripts. | Same properties file |
| `MNAAS_Adhoc_Queries_for_users_output_dir` | Directory where Java writes report files. | Same |
| `MNAAS_Adhoc_Queries_for_users_SriptName` | Human‑readable script identifier for logs/emails. | Same |
| `CLASSPATHVAR`, `MNAAS_Main_JarPath`, `MNAAS_adhoc_reports_for_users_classname` | Java runtime configuration. | Same |
| `MNAAS_Property_filename`, `MNAAS_Adhoc_Queries_Property_filename`, `MNASS_Adhoc_Temp_Scripts_filename` | Additional property files passed to Java. | Same |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | Email routing for failure notifications. | Same |
| `MNAAS_email_sdp_created` (in status file) | Flag indicating whether an SDP ticket has already been raised. | Managed by script |

If any of these variables are missing, the script will fail early (e.g., `sed` will not find the target file). Adding a validation block at the top of the script is recommended.

---

## 8. Suggested Improvements (TODO)

1. **Add pre‑flight validation** – Before any processing, loop through a required‑variables list and abort with a clear error if any are empty or files missing. This prevents silent failures caused by mis‑configured properties.
2. **Replace `chmod 777` with a configurable, least‑privilege mode** – Introduce a property `MNAAS_output_file_permission` (default `640`) and apply it via `chmod`. This reduces security exposure while preserving downstream access.

---