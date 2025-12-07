**File:** `move-mediation-scripts/bin/MNAAS_CDR_Active_feed_customer_secs_mapping_table.sh`

---

## 1. High‑Level Summary
This Bash script extracts the entire `mnaas.feed_customer_secs_mapping` table from an Impala cluster, writes the result as a semicolon‑delimited CSV file (`FeedCustomerSecsMapping.csv`) to the Hadoop‑side staging directory `/backup1/MNAAS/CDR/Active/`, makes the file world‑readable/writable, and logs start/end timestamps to a daily cron‑log file. The CSV is later consumed by downstream “active‑data” jobs that move the file from the Hadoop server to the RTDAS server (`/CDR/Mnass/Active`).

---

## 2. Key Script Elements & Responsibilities
| Element | Type | Responsibility |
|---------|------|-----------------|
| `MNAAS_WeeklyonceActiveDataLoad_LogPath` | variable | Builds the path of the daily log file (`MNAAS_WeeklyonceActiveDataLoad_YYYY‑MM‑DD.log`). |
| `logger -s "extract the Active data …"` | command | Writes a human‑readable start message to syslog **and** appends it to the daily log file. |
| `impala-shell … -q "select * from mnaas.feed_customer_secs_mapping;" … -o /backup1/MNAAS/CDR/Active/FeedCustomerSecsMapping.csv --print_header '--output_delimiter=;'` | command | Executes the SELECT, streams results to a CSV file with header row, using `;` as field delimiter. |
| `chmod -R 777 /backup1/MNAAS/CDR/Active/FeedCustomerSecsMapping.csv` | command | Ensures any downstream process (potentially running under a different user) can read/write the file. |
| `logger -s "END TIME :" $(date)` | command | Writes an end‑time stamp to syslog and the daily log. |

*No functions or classes are defined; the script is a linear procedural flow.*

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | • Impala cluster reachable at `192.168.124.90` (hard‑coded). <br>• Table `mnaas.feed_customer_secs_mapping` must exist and be accessible to the Impala user running the script. |
| **Outputs** | • `/backup1/MNAAS/CDR/Active/FeedCustomerSecsMapping.csv` – semicolon‑delimited CSV with header. <br>• Log entries appended to `$MNAAS_WeeklyonceActiveDataLoad_LogPath`. |
| **Side Effects** | • File permissions changed to `777`. <br>• Syslog entries generated (via `logger -s`). |
| **Assumptions** | • The executing user has permission to run `impala-shell` and write to `/backup1/MNAAS/CDR/Active/`. <br>• The directory `/backup1/MNAAS/CDR/Active/` already exists. <br>• No network partitions between the script host and the Impala node. |

---

## 4. Integration Points & Connectivity

| Connected Component | How the Connection Occurs |
|---------------------|---------------------------|
| **Downstream “Active‑Data” mover** (e.g., `MNAAS_Actives_tbl_load_driver.sh` or similar) | Consumes the CSV file produced here and copies it to the RTDAS server (`/CDR/Mnass/Active`). |
| **Cron scheduler** | Typically invoked by a daily/weekly cron job (see naming convention `WeeklyonceActiveDataLoad`). |
| **Impala service** | Accessed via `impala-shell -i 192.168.124.90`. Any change in Impala host/IP must be reflected here. |
| **System logger** | `logger -s` writes to `/var/log/messages` (or equivalent) and to the script‑specific log file. |
| **Potential upstream driver** (`MNAAS_WeeklyonceActiveDataLoad` driver script) | May set environment variables (e.g., `MNAAS_WeeklyonceActiveDataLoad_LogPath`) before invoking this script. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Hard‑coded Impala host** | Failure if the cluster IP changes. | Externalize the host/IP to a config file or environment variable (e.g., `IMPALA_HOST`). |
| **`chmod 777`** | Overly permissive file rights; possible security breach. | Use least‑privilege permissions (e.g., `chmod 664` and set proper group ownership). |
| **No error handling** | Script continues silently on `impala-shell` failure, producing an empty or missing CSV. | Capture exit status of `impala-shell`; abort and log error if non‑zero. |
| **Assumes directory exists** | Job fails if `/backup1/MNAAS/CDR/Active/` is missing. | Add a `mkdir -p` guard before the `impala-shell` command. |
| **Log file growth** | Daily logs accumulate; could fill disk. | Rotate logs via `logrotate` or limit size in the script. |
| **No concurrency guard** | Simultaneous runs could overwrite the same CSV. | Use a lock file (`flock`) or include timestamp in the filename. |

---

## 6. Running & Debugging Guide

1. **Prerequisites**  
   - Ensure `impala-shell` is in `$PATH` and can connect to `192.168.124.90`.  
   - Verify write permission on `/backup1/MNAAS/CDR/Active/`.  
   - Confirm the script is executable: `chmod +x MNAAS_CDR_Active_feed_customer_secs_mapping_table.sh`.

2. **Typical Invocation** (called by a cron job or driver script)  
   ```bash
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_CDR_Active_feed_customer_secs_mapping_table.sh
   ```

3. **Manual Debug Run**  
   ```bash
   set -x   # enable bash tracing
   ./MNAAS_CDR_Active_feed_customer_secs_mapping_table.sh
   echo "Exit status: $?"
   tail -n 20 "$MNAAS_WeeklyonceActiveDataLoad_LogPath"
   ```
   - Check the generated CSV size and header.  
   - Verify the log file contains both start and end timestamps.

4. **Common Failure Checks**  
   - **Impala connection error**: Look for “Connection refused” or “Authentication failed” in the log.  
   - **Empty CSV**: Confirm the table has rows (`SELECT COUNT(*) FROM mnaas.feed_customer_secs_mapping;`).  
   - **Permission denied**: Ensure the script runs as a user with write access to the target directory.

---

## 7. Configuration & Environment Variables

| Variable | Source | Usage |
|----------|--------|-------|
| `MNAAS_WeeklyonceActiveDataLoad_LogPath` | Defined inside the script (derived from current date). | Path to the daily log file. |
| `IMPALA_HOST` (suggested) | Not currently used; would replace hard‑coded IP. | Allows environment‑driven host configuration. |
| `IMPALA_USER` / `IMPALA_PASSWORD` (if needed) | Not present; may be required for secure clusters. | Authentication for `impala-shell`. |

*No external config files are referenced; all paths and hostnames are hard‑coded.*

---

## 8. Suggested Improvements (TODO)

1. **Add Robust Error Handling**  
   ```bash
   impala-shell … -o … || {
       logger -s "Impala query failed" 2>>"$MNAAS_WeeklyonceActiveDataLoad_LogPath"
       exit 1
   }
   ```

2. **Externalize Connection Parameters**  
   - Move `IMPALA_HOST`, output directory, and delimiter into a shared properties file (e.g., `/app/hadoop_users/MNAAS/conf/mnaas.env`) and source it at script start.

*Implementing these two items will improve reliability, maintainability, and security.*