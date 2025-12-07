**MNAAS_Billing_tables_loading.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)
`MNAAS_Billing_tables_loading.sh` orchestrates the nightly loading of five billing‑related tables for the MNAAS (Mobile Network As A Service) platform. It sequentially invokes dedicated loader scripts for each table, maintains a persistent process‑status file to allow safe restart/recovery, writes detailed logs, and on failure sends an email and creates an SDP ticket. The script is designed to run under a cron schedule, guaranteeing that only one instance executes at a time.

---

### 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **MNAAS_billable_sub_with_charge_and_limit_table** | Updates status flag = 1, logs start, runs the *billable_sub_with_charge_and_limit* loader, checks exit code, logs success/failure, aborts on error. |
| **MNAAS_customers_with_overage_flag_table** | Updates status flag = 2, runs the *customers_with_overage_flag* loader, same error handling pattern. |
| **MNAAS_usage_overage_charge_table** | Updates status flag = 3, runs the *usage_overage_charge* loader, same error handling pattern. |
| **MNAAS_service_based_charges_table** | Updates status flag = 4, runs the *service_based_charges* loader, same error handling pattern. |
| **MNAAS_customer_invoice_summary_estimated_table** | Updates status flag = 5, runs the *customer_invoice_summary_estimated* loader, same error handling pattern. |
| **terminateCron_successful_completion** | Resets status flag = 0, marks job as *Success*, records run time, clears email‑ticket flag, writes final log entries, exits 0. |
| **terminateCron_Unsuccessful_completion** | Records run time, marks job as *Failure*, triggers email/SDP ticket, exits 1. |
| **email_and_SDP_ticket_triggering_step** | Sends a pre‑formatted email to `hadoop-dev@tatacommunications.com` (CC list from `$ccList`) if a ticket has not yet been created, updates the ticket‑created flag, logs the action. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_Billing_tables_loading.properties` – defines all variables used (paths, script names, log file, status file, email list, etc.). |
| **Environment variables** | `MailId` (hard‑coded), optional `$ccList` for CC recipients. |
| **External scripts invoked** | Five loader scripts located in `$MNAASShellScriptPath` – names are read from the properties file. |
| **Process‑status file** | `$MNAAS_Billing_tables_loading_ProcessStatusFileName` – a key‑value file tracking PID, current flag, job status, timestamps, email‑ticket flag. Updated throughout execution. |
| **Log file** | `$MNAAS_Billing_tables_loadingLogPath` – appended with timestamps, status messages, and `logger -s` output. |
| **Email / SDP ticket** | On failure, an email is sent via the `mail` command; the script assumes an SDP ticketing system is triggered by that email (no API call). |
| **Side effects** | Populates/updates billing tables in the target database (performed by the called loader scripts), creates log entries, may generate an email/SDP ticket, writes to the status file. |
| **Assumptions** | - All loader scripts are executable and return 0 on success.<br>- The status file exists and is writable.<br>- `mail` command is configured and can reach the recipient list.<br>- The script runs under a user with permissions to read/write the log, status file, and execute the loader scripts.<br>- No other process modifies the status file concurrently. |

---

### 4. Integration Points (how it connects to other components)

| Component | Connection Detail |
|-----------|-------------------|
| **MNAAS_Billing_tables_loading.properties** | Provides all runtime parameters (paths, script names, log/status file locations). |
| **Loader scripts** (`*_loading_ScriptName`) | Each function calls one of these scripts; they perform the actual data extraction, transformation, and loading into the billing tables. |
| **Process‑status file** | Shared with other MNAAS batch jobs (e.g., `MNAAS_Billing_Export.sh`) to coordinate run state and avoid overlapping executions. |
| **Email system** | Uses the Unix `mail` utility; the email body references the log file location for troubleshooting. |
| **SDP ticketing** | Implicit – the email is expected to be routed to an SDP (Service Desk Platform) system that creates a ticket. |
| **Cron scheduler** | The script is intended to be invoked by a cron entry (not shown) that runs nightly. |
| **Potential downstream consumers** | Downstream reporting or billing systems that read the populated tables; not directly referenced but rely on successful completion. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Stale PID in status file** – if the script crashes, the PID may remain, preventing future runs. | Add a sanity check on the PID’s start time (`ps -p $PID -o lstart=`) and clear it if the process is not alive for > 2 hours. |
| **Status‑file corruption** – concurrent writes could corrupt the key‑value file. | Serialize access using `flock` on the status file before any `sed` operation. |
| **Loader script failure not captured** – non‑zero exit code but script still writes success logs. | Ensure each loader script exits with proper status; optionally capture its stdout/stderr to the main log. |
| **Email flood** – repeated failures could generate many emails. | Implement a back‑off counter in the status file (e.g., `email_retry_count`) and suppress emails after N attempts until manual reset. |
| **Missing/incorrect properties** – undefined variables cause silent failures. | Validate required variables after sourcing the properties file; abort with a clear error if any are empty. |
| **Insufficient permissions** – inability to write log or status file. | Run a pre‑flight permission check (`test -w $file`) and alert via syslog if not writable. |

---

### 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure `$MNAAS_Billing_tables_loading_ProcessStatusFileName` and `$MNAAS_Billing_tables_loadingLogPath` exist and are writable by the executing user.  
   - Verify that all loader scripts referenced in the properties file are present in `$MNAASShellScriptPath` and have execute permission (`chmod +x`).  
   - Confirm the `mail` command works (test by sending a simple email).  

2. **Manual execution**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Property_Files   # location of the properties file
   . MNAAS_Billing_tables_loading.properties          # load vars into current shell
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_Billing_tables_loading.sh
   ```
   - Observe console output (due to `set -x`).  
   - Tail the log file to watch progress: `tail -f $MNAAS_Billing_tables_loadingLogPath`.  

3. **Debugging tips**  
   - **Check PID handling**: `cat $MNAAS_Billing_tables_loading_ProcessStatusFileName | grep MNAAS_Script_Process_Id`.  
   - **Force a specific step**: Manually set `MNAAS_Daily_ProcessStatusFlag` in the status file to the desired flag (1‑5) and re‑run; the script will resume from that step.  
   - **Capture loader output**: Edit a function to redirect the loader’s stdout/stderr to the main log, e.g., `sh $script >> $MNAAS_Billing_tables_loadingLogPath 2>&1`.  
   - **Simulate failure**: Temporarily change a loader script to `exit 1` and verify that the email/SDP ticket logic fires.  

4. **Cron integration**  
   - Typical cron entry (run nightly at 02:00):  
     ```cron
     0 2 * * * /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_Billing_tables_loading.sh >> /dev/null 2>&1
     ```

---

### 7. External Configuration & Environment Dependencies

| Item | Usage |
|------|-------|
| **`MNAAS_Billing_tables_loading.properties`** | Supplies all variable definitions: <br>• `$MNAASShellScriptPath` – directory containing loader scripts.<br>• `$MNAAS_Billing_tables_loadingLogPath` – absolute path to the log file.<br>• `$MNAAS_Billing_tables_loading_ProcessStatusFileName` – path to the status file.<br>• `$MNAAS_billable_sub_with_charge_and_limit_loading_ScriptName`, etc. – names of the loader scripts.<br>• `$MNAAS_Billing_tables_loading_ScriptName` – the name of this orchestrator script (used in status updates). |
| **`MailId` (hard‑coded)** | Destination address for failure notifications. |
| **`ccList` (environment variable)** | Optional comma‑separated list of CC recipients for the failure email. |
| **`$PATH` / `sh`** | The script invokes other shell scripts via `sh`; they must be reachable via the PATH or absolute path. |
| **`mail` command** | Required for email notifications; assumes a working MTA on the host. |

---

### 8. Suggested Improvements (TODO)

1. **Add robust status‑file locking** – wrap every `sed` operation with `flock $MNAAS_Billing_tables_loading_ProcessStatusFileName` to prevent race conditions if multiple jobs ever touch the same file.  
2. **Parameter validation block** – after sourcing the properties file, iterate over required variables and abort with a clear error if any are empty or point to non‑existent files/directories.  

---