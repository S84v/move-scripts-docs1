**Sim_Inventory_Summary.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)  
`Sim_Inventory_Summary.sh` is a daily orchestration script that maintains the **SIM‑Inventory‑Summary** Hive/Impala table. It removes (drops) the current month’s partition – and, on the 1st or 2nd of the month, also the previous month and the 6‑month‑old partition – then reloads the current month’s data (and, when applicable, the previous month’s data) using Hive INSERT statements defined in an external properties file. After each Hive load it issues an Impala `REFRESH` to make the new data visible. The script records its progress in a shared *process‑status* file, writes detailed logs, prevents concurrent executions, and raises an SDP ticket via email if any step fails.

---

### 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **truncate_partition()** | Updates status flag to *1*, logs start, drops the current month partition (and optionally previous & 6‑month‑old partitions), runs Impala `REFRESH`, and handles success/failure. |
| **load_sim_inventory_summary_table()** | Updates status flag to *2*, logs start, runs the Hive INSERT query for the current month (and previous month on day 1‑2), runs Impala `REFRESH`, and handles success/failure. |
| **terminateCron_successful_completion()** | Resets status flag to *0* (Success), writes final log entries, and exits with status 0. |
| **terminateCron_Unsuccessful_completion()** | Logs failure, triggers email ticket, and exits with status 1. |
| **email_on_reject_triggering_step()** | Builds a minimal SDP‑style email (using `mailx`) to the support mailbox and logs that the ticket was created. |
| **Main program** | Reads the previous PID from the status file, ensures only one instance runs, decides which step(s) to execute based on the stored flag (0/1 → truncate + load, 2 → load only), and drives termination handling. |

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Inputs** | • `Sim_Inventory_Summary.properties` (contains all configurable variables).<br>• Environment variable `setparameter` (evaluated before sourcing the properties).<br>• System date (`date` command) for month calculations.<br>• Process‑status file (`$MNAAS_Sim_Inventory_Summary_ProcessStatusFileName`). |
| **Outputs** | • Hive partitions added/dropped in `$dbname.$MNAAS_Sim_Inventory_Summary_tblname`.<br>• Impala metadata refresh (`$MNAAS_Sim_Inventory_Summary_Refresh`).<br>• Log file (`$MNAAS_Sim_Inventory_Summary_logpath`).<br>• Updated process‑status file (flags, PID, job status). |
| **Side Effects** | • Potential data loss if a wrong partition is dropped.<br>• Email sent to `insdp@tatacommunications.com` on failure.<br>• System‑wide `ps` lookup to detect concurrent runs. |
| **Assumptions** | • Hive, Impala, and `mailx` are installed and reachable.<br>• The properties file defines all required variables (DB name, table name, queries, hosts, email addresses, etc.).<br>• The script runs under a user with permissions to modify the status file, drop/create Hive partitions, and send mail. |
| **External Services** | • **Hive** (via `hive -e`).<br>• **Impala** (via `impala-shell`).<br>• **SMTP** (used by `mailx`).<br>• **Filesystem** (process‑status and log files). |

---

### 4. Integration Points (how it connects to other scripts/components)  

| Connection | Description |
|------------|-------------|
| **Process‑status file** (`$MNAAS_Sim_Inventory_Summary_ProcessStatusFileName`) | Shared with other MNAAS daily scripts to coordinate execution order and prevent overlap. The flag values (0‑Success, 1‑Truncate, 2‑Load) are read/written by this script and may be inspected by monitoring tools or downstream jobs. |
| **Properties file** (`Sim_Inventory_Summary.properties`) | Central configuration used by multiple “SIM‑Inventory” related scripts (e.g., `Sim_Inventory_Sqoop.sh`). Any change here propagates to all callers. |
| **Hive/Impala tables** (`$MNAAS_Sim_Inventory_Summary_tblname`) | Populated here; downstream reporting or analytics jobs read from this table. |
| **Email/SDP ticketing** | Failure notification integrates with the company’s SDP incident system via a formatted email. Other scripts follow the same pattern, creating a consistent incident workflow. |
| **Cron scheduler** | The script is intended to be invoked by a daily cron entry (similar to other `Move_*.sh` scripts). The PID guard prevents overlapping cron runs. |
| **Potential downstream scripts** | Scripts that generate weekly/monthly aggregates (e.g., `MNAAS_Usage_Trend_Aggr.sh`) likely depend on the refreshed SIM‑Inventory‑Summary data. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Accidental partition drop** (wrong month) | Add a sanity check that the month string matches the expected format and that the partition exists before dropping. Consider a “dry‑run” flag for testing. |
| **Hive/Impala command failure** (network, syntax, permission) | Capture exit codes immediately after each command; implement retry logic for transient failures. Log full Hive/Impala output for post‑mortem. |
| **Stale PID causing false‑positive “already running”** | Verify that the PID still belongs to this script (e.g., check command line) before aborting. Clean up the status file on abnormal termination. |
| **Email delivery failure** (mailx mis‑config) | Check the return status of `mailx`; fallback to writing the incident to a local file or alerting via another channel (e.g., Slack). |
| **Missing/incorrect properties** | Validate required variables at script start; abort early with a clear error if any are undefined. |
| **Concurrent runs from different nodes** (if cron runs on multiple hosts) | Store the PID file on a shared filesystem (e.g., HDFS or NFS) and use file locking (`flock`) to guarantee exclusivity across nodes. |

---

### 6. Typical Execution / Debugging Steps  

1. **Run manually** (as the same user that cron uses):  
   ```bash
   cd /app/hadoop_users/MNAAS/
   ./move-mediation-scripts/bin/Sim_Inventory_Summary.sh
   ```
2. **Check logs** – tail the log file defined in the properties:  
   ```bash
   tail -f $MNAAS_Sim_Inventory_Summary_logpath
   ```
3. **Verify status file** – ensure flags are updated as expected:  
   ```bash
   cat $MNAAS_Sim_Inventory_Summary_ProcessStatusFileName
   ```
4. **Inspect Hive partitions** (optional):  
   ```bash
   hive -e "SHOW PARTITIONS $dbname.$MNAAS_Sim_Inventory_Summary_tblname;"
   ```
5. **Debug failures** – if the script exits with status 1, look for the “Script … terminated Unsuccessfully” entry in the log, then:  
   * Re‑run the failing Hive query directly to see the error.  
   * Verify Impala connectivity (`impala-shell -i $IMPALAD_HOST -q "SELECT 1;"`).  
   * Check the email inbox for the SDP ticket; the email body contains the failure reason.  

---

### 7. External Configuration & Environment Variables  

| Variable | Source | Use |
|----------|--------|-----|
| `setparameter` | Exported by the calling environment (e.g., cron wrapper) | Evaluated before sourcing the properties file – typically sets Java/Hadoop classpaths or other global flags. |
| `Sim_Inventory_Summary.properties` | File located at `/app/hadoop_users/MNAAS/MNAAS_Property_Files/` | Defines all `$MNAAS_*` variables used throughout the script (DB name, table name, Hive queries, Impala host, log & status file paths, email addresses, etc.). |
| `$MNAAS_Sim_Inventory_Summary_ProcessStatusFileName` | From properties | Central status/flag file used for concurrency control and progress tracking. |
| `$MNAAS_Sim_Inventory_Summary_logpath` | From properties | Path to the script’s log file. |
| `$MNAAS_Sim_Inventory_Summary_Query` / `$MNAAS_Sim_Inventory_Summary_Prev_Query` | From properties | Hive INSERT statements that populate the current and previous month partitions. |
| `$MNAAS_Sim_Inventory_Summary_Refresh` | From properties | Impala `REFRESH` command (e.g., `REFRESH $dbname.$MNAAS_Sim_Inventory_Summary_tblname`). |
| `$IMPALAD_HOST` | From properties | Hostname of the Impala daemon. |
| `$SDP_ticket_from_email`, `$T0_email` | From properties | Email addresses used when generating an SDP incident ticket. |
| `$MNAAS_Sim_Inventory_Summary_Script` | From properties | Human‑readable script identifier used in logs and email subject. |

---

### 8. Suggested Improvements (TODO)  

1. **Add robust parameter validation** – early exit with a clear message if any required `$MNAAS_*` variable is empty or malformed.  
2. **Implement file locking (`flock`)** on the process‑status file to guarantee single‑instance execution across multiple hosts, eliminating reliance on stale PID checks.  

---