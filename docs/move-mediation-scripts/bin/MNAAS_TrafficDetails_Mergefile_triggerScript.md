**File:** `move-mediation-scripts/bin/MNAAS_TrafficDetails_Mergefile_triggerScript.sh`

---

## 1. Summary
This Bash script is a time‑based trigger that runs every hour (via cron) and decides whether to start the “merged traffic‑details” load into HDFS. It calculates the current hour modulo 2; if the result matches a hard‑coded value (`var=0`), it logs a “trigger” message and creates a lock file (`${MergeLockName}`). Down‑stream processes watch for this lock file to actually perform the heavy‑weight merge and HDFS import. If the hour does not match, it only logs that the trigger time is not applicable.

---

## 2. Key Elements & Responsibilities

| Element | Type | Responsibility |
|---------|------|-----------------|
| **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_TrafficDetails_Mergefile_triggerScript.properties`** | Source file | Supplies configurable variables such as `MNAAS_TrafficDetails_Mergefile_triggerScript_log` (log file path) and `MergeLockName` (full path of the lock file). |
| `dt_var` | Variable | Holds the current hour (`date +"%H"`) modulo 2 – determines the 2‑hour window. |
| `var` | Variable | Hard‑coded comparison value (`0`). Changing this would shift the trigger window. |
| `logger -s` | Command | Writes a timestamped message to syslog **and** to the script‑specific log file (stderr redirected). |
| `touch ${MergeLockName}` | Command | Creates (or updates) the lock file that signals downstream jobs to start the merge load. |
| `set -x` | Bash option | Enables command‑trace debugging; useful for troubleshooting. |

No functions or classes are defined; the script is a linear procedural flow.

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | - System clock (current hour).<br>- Environment variables defined in the sourced `.properties` file (log path, lock file name, possibly others). |
| **Outputs** | - Log entries appended to `$MNAAS_TrafficDetails_Mergefile_triggerScript_log` (and syslog).<br>- Creation/refresh of `${MergeLockName}` (empty file). |
| **Side Effects** | - Down‑stream processes (e.g., a Hadoop/Sqoop load script) poll `${MergeLockName}` to start the actual merge and HDFS import.<br>- Potentially triggers a Hadoop job indirectly. |
| **Assumptions** | - The properties file exists and is readable by the executing user.<br>- The executing user has write permission on the log file and the directory containing `${MergeLockName}`.<br>- `logger` is available on the host.<br>- A separate consumer script reliably removes the lock file after processing. |

---

## 4. Interaction with Other Components

| Connected Component | Interaction Point |
|---------------------|-------------------|
| **`MNAAS_TrafficDetails_Mergefile_triggerScript.properties`** | Provides `MNAAS_TrafficDetails_Mergefile_triggerScript_log` and `MergeLockName`. |
| **Down‑stream merge loader (e.g., `MNAAS_TrafficDetails_Mergefile_Load.sh` or a Hadoop job)** | Monitors `${MergeLockName}`; when the file appears, the loader starts the merge, writes results to HDFS, and removes the lock. |
| **Cron scheduler** | Typically invoked every hour (e.g., `0 * * * * /path/MNAAS_TrafficDetails_Mergefile_triggerScript.sh`). |
| **System syslog** | Receives the same messages logged via `logger -s`. |
| **Potential alerting/monitoring tools** | May watch the log file for “every 2 hours” messages to confirm successful triggers. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale lock file** – if downstream job fails to delete `${MergeLockName}` the script will keep recreating it, causing repeated unnecessary triggers. | Duplicate processing, resource waste. | Add a cleanup step (e.g., `rm -f ${MergeLockName}` before `touch`) or enforce a TTL check on the lock file’s age. |
| **Incorrect hour calculation** – daylight‑saving changes or clock drift could cause missed triggers. | Missed data loads. | Use UTC (`date -u +"%H"`) consistently across all scripts; verify NTP sync on the host. |
| **Missing/invalid properties file** – script aborts silently, no log entry. | No trigger, silent failure. | Add a check after sourcing: `if [ -z "$MergeLockName" ]; then logger "Properties not loaded"; exit 1; fi`. |
| **Permission issues** – inability to write log or lock file. | No trigger, error logged to syslog only. | Ensure the executing user belongs to the appropriate group and that directory permissions are set to `775` (or as required). |
| **Hard‑coded `var=0`** – future schedule changes require script edit. | Maintenance overhead. | Move `var` into the properties file or make it a command‑line argument. |

---

## 6. Running / Debugging the Script

1. **Typical execution (via cron):**  
   ```bash
   0 * * * * /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_TrafficDetails_Mergefile_triggerScript.sh
   ```

2. **Manual run (for testing):**  
   ```bash
   cd /app/hadoop_users/MNAAS/move-mediation-scripts/bin
   ./MNAAS_TrafficDetails_Mergefile_triggerScript.sh
   ```

3. **Debug mode:**  
   The script already enables `set -x`, which prints each command with expanded variables to stdout. Capture this output:  
   ```bash
   ./MNAAS_TrafficDetails_Mergefile_triggerScript.sh > /tmp/debug.log 2>&1
   ```

4. **Verification steps:**  
   - Check the log file defined by `$MNAAS_TrafficDetails_Mergefile_triggerScript_log` for the “every 2 hour” entry.  
   - Verify that `${MergeLockName}` exists (or was refreshed) when the hour matches.  
   - Confirm downstream loader picks up the lock (e.g., by tailing its own log).  

5. **Common troubleshooting:**  
   - If the log file is empty, ensure the properties file defines `MNAAS_TrafficDetails_Mergefile_triggerScript_log`.  
   - If the lock file is never created, verify the hour modulo calculation (`date +"%H"` returns a two‑digit hour).  
   - Use `env | grep MNAAS` to confirm that all required variables are exported.

---

## 7. External Configuration & Environment Variables

| Variable (from properties) | Purpose | Typical Location |
|----------------------------|---------|------------------|
| `MNAAS_TrafficDetails_Mergefile_triggerScript_log` | Path to the script‑specific log file (stderr of `logger`). | `/app/hadoop_users/MNAAS/logs/...` |
| `MergeLockName` | Full pathname of the lock file used as a trigger for downstream processing. | `/app/hadoop_users/MNAAS/locks/...` |
| *(Potential others)* | Any additional settings (e.g., HDFS target path) that downstream jobs may read. | Same properties file |

The script expects the properties file at:  
`/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_TrafficDetails_Mergefile_triggerScript.properties`

---

## 8. Suggested Improvements (TODO)

1. **Robust lock handling** – Before `touch`, check for an existing lock older than a configurable threshold (e.g., 4 h) and remove it, preventing stale locks from blocking future runs.  
2. **Parameterize the trigger interval** – Move `var=0` and the modulo divisor (`2`) into the properties file (or accept them as CLI arguments) so the schedule can be changed without script modification.

---