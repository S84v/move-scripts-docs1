**End‑to‑End MOVE NL vs Mediation Reconciliation Script**  
`move-mediation-scripts/bin/End_to_End_MOVENL_Mediation_Dashborad.sh`

---

### 1. High‑Level Summary
This Bash script orchestrates the end‑to‑end reconciliation between the MOVE‑NL data set and the Mediation data warehouse for a given month. It drives three Hive reconciliation jobs (traffic‑file month‑end, traffic‑file daily, and customer‑monthly report), updates a shared process‑status file, logs progress, and on failure automatically raises an SDP ticket and sends an email to the support team. The script is intended to be executed nightly (or on demand) via cron, using a properties file that supplies all environment‑specific paths, email addresses, and job‑control flags.

---

### 2. Core Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **Med_Traffic_File_Month_End_Recon_Process** | Runs `med_traffic_file_recon.hql` for the *previous* month (only if current day ≤ 5). Updates status flag = 1 and logs success/failure. |
| **Med_Traffic_File_Recon_Process** | Runs `med_traffic_file_recon.hql` for the month supplied via `$file_date` (or previous month if not supplied). Updates status flag = 2. |
| **Med_Cust_Monthly_Rep_Recon** | Runs `med_cust_monthly_rep_recon.hql` for the month supplied via `$file_month` (or previous month). Updates status flag = 3. |
| **terminateCron_successful_completion** | Resets status flag = 0, marks job as *Success*, writes run‑time, logs completion, and exits 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, invokes `email_and_SDP_ticket_triggering_step`, and exits 1. |
| **email_and_SDP_ticket_triggering_step** | If an SDP ticket has not yet been created, composes a `mailx` message with predefined headers and sends it to the configured recipients; updates status file to indicate ticket/email sent. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Script Arguments** | `$1` → `file_date` (format `YYYYMM`), `$2` → `file_month` (format `YYYY‑MM`). If omitted, defaults to the previous calendar month. |
| **External Config / Env Vars** (populated by the sourced properties file) | `E2E_MoveNL_Mediation_Dashboard_LogName` – path to the log file.<br>`E2E_MoveNL_Mediation_Dashboard_ProcessStatusFileName` – key‑value status file.<br>`MNAASShellScriptPath` – directory containing the `.hql` scripts.<br>`SDP_ticket_from_email`, `SDP_ticket_to_email`, `MOVE_DEV_TEAM` – email routing.<br>`E2E_MoveNL_Mediation_Dashboard_ScriptName` – script identifier used in logs/emails. |
| **Hive Scripts Executed** | `med_traffic_file_recon.hql` (traffic file reconciliation).<br>`med_cust_monthly_rep_recon.hql` (customer monthly report reconciliation). |
| **Outputs** | Updated status file (flags, job status, timestamps, ticket flag).<br>Log file entries (start/end timestamps, success/failure messages). |
| **Side Effects** | Potential creation of an SDP ticket via `mailx`.<br>Modification of the shared status file (used by other orchestration components). |
| **Assumptions** | • The properties file exists and defines all referenced variables.<br>• Hive CLI is available and the `.hql` files are syntactically correct.<br>• The status file is a plain text `KEY=VALUE` file and is writable by the script user.<br>• `mailx` is configured for outbound SMTP.<br>• No concurrent executions of this script (or of any other process that writes the same status file). |

---

### 4. Interaction with Other Components

| Component | Connection Point |
|-----------|------------------|
| **Properties File** (`E2E_MoveNL_Mediation_Dashboard.properties`) | Provides all runtime configuration (paths, email addresses, status‑file location). |
| **Hive Reconciliation Jobs** (`*.hql`) | Executed by this script; they read/write tables in the Mediation Hive warehouse. |
| **Process‑Status File** (`$E2E_MoveNL_Mediation_Dashboard_ProcessStatusFileName`) | Shared with other MOVE‑NL pipelines; its flags determine which steps are run on subsequent invocations. |
| **SDP Ticketing System** | Triggered via `mailx`; downstream ticketing automation consumes the email. |
| **Cron Scheduler** | Typically invoked nightly; the script’s flag logic allows it to resume from the last successful step if a previous run failed. |
| **Logging Infrastructure** | Uses `logger` (syslog) and redirects to a dedicated log file; downstream monitoring dashboards may ingest this log. |

---

### 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Status file corruption** (sed in‑place edits) | Subsequent runs may mis‑interpret flags, causing skipped steps or infinite loops. | • Use atomic write (`mv temp → status`) instead of direct `sed`.<br>• Add checksum verification after each edit.<br>• Restrict script execution to a single instance (lock file). |
| **Hive job failure not captured** (non‑zero exit but script continues) | Incomplete reconciliation, stale data. | • Ensure `set -e` or explicit exit on any non‑zero Hive return.<br>• Capture Hive logs for deeper analysis. |
| **Email/SDP ticket not sent** (mailx mis‑configuration) | Failure not escalated, prolonged outage. | • Verify mailx connectivity before production run.<br>• Add fallback to write a local alert file. |
| **Date calculation edge cases** (e.g., month‑end on Jan 31) | Wrong month processed, data mismatch. | • Use GNU `date` with explicit timezone and locale.<br>• Add unit tests for date logic. |
| **Concurrent runs** (multiple cron entries) | Race conditions on status file and Hive tables. | • Implement a PID lock file at script start.<br>• Schedule with sufficient gap or use a workflow engine (Airflow/Oozie). |

---

### 6. Running & Debugging the Script

**Typical Invocation (via cron or manually):**
```bash
# Run for previous month (default)
bash End_to_End_MOVENL_Mediation_Dashborad.sh

# Run for a specific month (e.g., March 2024)
bash End_to_End_MOVENL_Mediation_Dashborad.sh 202403 2024-03
```

**Prerequisites Before Execution**
1. Verify that `/app/hadoop_users/MNAAS/MNAAS_Property_Files/E2E_MoveNL_Mediation_Dashboard.properties` exists and contains all required variables.
2. Ensure the user has write permission on the log file and status file.
3. Confirm Hive CLI (`hive`) is in `$PATH` and the `.hql` scripts are present under `$MNAASShellScriptPath`.

**Debugging Steps**
1. The script starts with `set -x`; the full command trace is written to the log file (`$E2E_MoveNL_Mediation_Dashboard_LogName`). Review this file for the exact commands executed.
2. Check the status file after each run to see the current flag values.
3. Manually run a Hive query to verify it succeeds:
   ```bash
   hive --hiveconf filedate=202403 -f $MNAASShellScriptPath/med_traffic_file_recon.hql
   ```
4. If a failure occurs, the script will invoke `email_and_SDP_ticket_triggering_step`. Verify that an email was sent and that the ticket flag (`MNAAS_email_sdp_created`) is set to `Yes`.
5. Use `ps -ef | grep End_to_End_MOVENL` to ensure no stray instances are running.

---

### 7. External Configuration & Environment Variables

| Variable | Source | Purpose |
|----------|--------|---------|
| `E2E_MoveNL_Mediation_Dashboard_LogName` | properties file | Path to the script‑specific log file. |
| `E2E_MoveNL_Mediation_Dashboard_ProcessStatusFileName` | properties file | Shared status file tracking flags and job state. |
| `MNAASShellScriptPath` | properties file | Directory containing the Hive `.hql` scripts. |
| `SDP_ticket_from_email` / `SDP_ticket_to_email` | properties file | Email addresses used for SDP ticket creation. |
| `MOVE_DEV_TEAM` | properties file | CC list for failure notifications. |
| `E2E_MoveNL_Mediation_Dashboard_ScriptName` | properties file | Human‑readable identifier used in logs/emails. |
| `MNAAS_job_status`, `MNAAS_email_sdp_created`, `MNAAS_job_ran_time` | status file | Fields updated by the script to reflect run outcome. |

---

### 8. Suggested Improvements (TODO)

1. **Atomic Status File Updates** – Replace each `sed -i` sequence with a “write‑to‑temp‑file then `mv`” pattern to avoid partial writes and enable rollback on error.
2. **Integrate with a Workflow Engine** – Move the flag‑based orchestration into a proper DAG (e.g., Apache Airflow) to gain better visibility, retries, and concurrency control, eliminating the need for manual flag manipulation. 

---