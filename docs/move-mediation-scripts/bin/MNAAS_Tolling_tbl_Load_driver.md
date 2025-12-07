**File:** `move-mediation-scripts/bin/MNAAS_Tolling_tbl_Load_driver.sh`  

---

## 1. Purpose (one‑paragraph summary)

This driver script orchestrates the daily “Tolling” data‑movement pipeline for the MNAAS mediation platform. It reads a property file to obtain configuration, checks that no previous instance is still running, updates a shared status file, iterates over each configured Tolling feed, and invokes the aggregation script (`MNAASDailyTollingAggregationScriptName`) with the appropriate file‑pattern and last‑processed‑file pointer. On successful completion it marks the job as *Success*; on any error it logs the failure, sends an alert e‑mail, creates an SDP ticket, and updates the status file to *Failure*. All activity is written to a central log file.

---

## 2. Key Functions / Logical Blocks

| Function / Block | Responsibility |
|------------------|----------------|
| **`. /app/hadoop_users/MNAAS/.../MNAAS_Tolling_tbl_Load_driver.properties`** | Sources all environment‑specific variables (paths, arrays, e‑mail lists, script names). |
| **`MNAAS_File_Processing_Tolling`** | Loops over `FEED_FILE_PATTERN_TOLLING` associative array, logs start/end, calls the aggregation script for each feed, and aborts on any non‑zero exit status. |
| **`terminateCron_successful_completion`** | Writes *Success* flags, timestamps, and a “completed” message to the combined status file; exits 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs failure, invokes `email_and_SDP_ticket_triggering_step`, exits 1. |
| **`email_and_SDP_ticket_triggering_step`** | Updates status file to *Failure*, checks if an SDP ticket/e‑mail has already been generated, and if not sends a formatted e‑mail (via `mail`/`mailx`) and creates an SDP ticket. |
| **Main script block** | Checks for an existing PID in the status file, validates the process flag, decides whether to start processing or skip, and finally calls the appropriate termination routine. |

---

## 3. Inputs, Outputs & Side‑Effects

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `MNAAS_Tolling_tbl_Load_driver.properties` – defines:<br>• `$MNAAS_DailyTollingCombinedLoadLogPath` (log file)<br>• `$MNAAS_Daily_Tolling_Load_CombinedTollingStatuFile` (status file)<br>• `FEED_FILE_PATTERN_TOLLING` (associative array of feed → filename pattern)<br>• `$MNAASShellScriptPath` and `$MNAASDailyTollingAggregationScriptName` (aggregation script location)<br>• `$MNAASDailyTollingProcessingScript`, `$MNAASDailyTollingProcessingScript` (self‑reference for logging)<br>• `$ccList`, `$GTPMailId`, `$SDP_ticket_from_email`, `$MOVE_DEV_TEAM` (e‑mail routing) |
| **Runtime arguments** | None – script is self‑contained, driven entirely by the property file. |
| **External scripts** | `$MNAASShellScriptPath/$MNAASDailyTollingAggregationScriptName` – performs the actual file aggregation and load for a single feed. |
| **External services** | • Local file system (status file, log file, feed files)<br>• `mail` / `mailx` (SMTP) for alert e‑mail<br>• SDP ticketing system (via e‑mail to `insdp@tatacommunications.com`) |
| **Outputs** | • Updated status file (`MNAAS_Daily_Tolling_Load_CombinedTollingStatuFile`) with flags, timestamps, PID, job status.<br>• Log entries appended to `$MNAAS_DailyTollingCombinedLoadLogPath`.<br>• Optional alert e‑mail and SDP ticket on failure. |
| **Assumptions** | • Property file exists and is readable.<br>• All referenced variables are defined in the property file.<br>• The status file is writable by the script user.<br>• The aggregation script returns `0` on success and non‑zero on error.<br>• `mail`/`mailx` are correctly configured on the host.<br>• Only one instance runs at a time (PID tracking). |

---

## 4. Interaction with Other Components

| Component | Connection Detail |
|-----------|-------------------|
| **MNAAS_Tolling_tbl_Load.sh** (or similar load script) | This driver is the *orchestrator*; the actual Hive/Impala load is performed inside the aggregation script invoked per feed. |
| **Other driver scripts** (e.g., `MNAAS_TapErrors_tbl_Load.sh`, `MNAAS_Telena_*`) | All share the same pattern of a status file, PID tracking, and email/SDP alert logic. The status file name is unique per domain (`*_CombinedTollingStatuFile`). |
| **Property files** (`MNAAS_Tolling_tbl_Load_driver.properties`) | Centralised configuration; any change to feed patterns, paths, or e‑mail lists propagates to this driver. |
| **Cron scheduler** | Typically executed nightly via a cron entry; the script itself checks for concurrent runs. |
| **SDP ticketing system** | Failure alerts are sent as e‑mail to `insdp@tatacommunications.com` with specific header tags for automatic ticket creation. |
| **Logging infrastructure** | Log file path is defined in the property file; downstream monitoring (e.g., Splunk, ELK) can ingest this file. |

---

## 5. Operational Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Stale PID / false‑positive “process already running”** | Add a sanity check on the PID’s start time (e.g., compare with current time) and optionally kill orphaned processes after a timeout. |
| **Status file contention (multiple scripts editing same file)** | Ensure each domain uses its own status file; if shared, protect updates with `flock`. |
| **Missing or malformed property file** | Validate required variables after sourcing; exit early with a clear error message. |
| **Aggregation script failure not captured** | Verify that the called script always returns a proper exit code; wrap the call in `set -e` or capture `$?` as done. |
| **Email/SDP ticket flood on repeated failures** | The script checks `MNAAS_email_sdp_created` flag; ensure the flag is reset on the next successful run. |
| **Log file growth** | Rotate the log file via logrotate or a size‑based truncation step. |
| **Hard‑coded paths** | Move absolute paths into the property file and document them; avoid inline literals. |

---

## 6. Running / Debugging the Script

1. **Standard execution (cron)**  
   ```bash
   /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Tolling_tbl_Load_driver.properties
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_Tolling_tbl_Load_driver.sh
   ```
   The script will log to the path defined in the property file and exit with status 0 (success) or 1 (failure).

2. **Manual run (for testing)**  
   ```bash
   cd /app/hadoop_users/MNAAS/move-mediation-scripts/bin
   ./MNAAS_Tolling_tbl_Load_driver.sh
   ```
   *Tip:* prepend `bash -x` (already enabled via `set -x`) to see each command as it executes.

3. **Debugging steps**  
   - Verify the property file loads correctly: `source /path/to/properties && env | grep MNAAS`.  
   - Check the status file content before and after run (`cat $MNAAS_Daily_Tolling_Load_CombinedTollingStatuFile`).  
   - Force a failure in the aggregation script (e.g., `exit 1`) and confirm that an e‑mail and SDP ticket are generated.  
   - Tail the log file while the script runs: `tail -f $MNAAS_DailyTollingCombinedLoadLogPath`.  

4. **Checking for orphaned processes**  
   ```bash
   ps -ef | grep MNAAS_Tolling_tbl_Load_driver.sh
   ```

---

## 7. External Configuration & Environment Variables

| Variable (from properties) | Meaning / Usage |
|----------------------------|-----------------|
| `MNAAS_DailyTollingCombinedLoadLogPath` | Full path to the combined log file for this driver. |
| `MNAAS_Daily_Tolling_Load_CombinedTollingStatuFile` | Shared status file that stores flags, PID, timestamps, etc. |
| `FEED_FILE_PATTERN_TOLLING` | Associative array mapping feed source names to filename glob patterns. |
| `MNAASShellScriptPath` | Directory containing the aggregation script. |
| `MNAASDailyTollingAggregationScriptName` | Name of the aggregation script invoked per feed. |
| `MNAASDailyTollingProcessingScript` | Self‑reference used for logging (usually the driver filename). |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM` | E‑mail routing parameters for failure alerts. |
| `MNAASDailyTollingProcessingScript` (duplicate) | Used in log messages and status updates. |

*If any of these are missing, the script will fail early; therefore the property file must be audited whenever changes are made.*

---

## 8. Suggested Improvements (TODO)

1. **Add robust property validation** – after sourcing, loop over a whitelist of required variables and abort with a clear message if any are undefined.
2. **Implement file‑lock (`flock`) around status‑file updates** – prevents race conditions when multiple drivers (or manual runs) attempt to modify the same status file concurrently.

---