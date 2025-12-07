**High‑Level Summary**  
`test.sh` is a lightweight ad‑hoc reporting script that executes a single Impala query against the MNAAS data‑warehouse, extracts a consolidated view of daily traffic file statistics (record counts, usage, reject counts, etc.) for the previous calendar day, and emails the raw query output to a hard‑coded recipient. It is primarily used for manual verification or quick diagnostics rather than as a scheduled production job.

---

## 1. Core Logic Overview
| Step | Action | Details |
|------|--------|---------|
| 1 | Build & run Impala query | Uses `impala-shell -i 192.168.124.90 -q "<SQL>"` to pull data from `mnaas.traffic_file_record_count`, `traffic_details_raw_daily`, and `traffic_details_raw_reject_daily`. The query filters on `substring(filename,35,8)` matching yesterday’s `yyyyMMdd` value (computed with `adddate(now(), -1)`). |
| 2 | Capture output | Standard‑output and standard‑error of the Impala command are stored in the variable `output_log`. |
| 3 | Email result | The content of `output_log` is piped to `mailx` with subject **“RA Report Output”** and sent to `anusha.gurumurthy@contractor.tatacommunications.com`. |

No functions, loops, or external scripts are invoked.

---

## 2. Important Elements (Shell “components”)

| Element | Responsibility |
|---------|-----------------|
| `impala-shell` | CLI client that connects to the Impala daemon at `192.168.124.90` and runs the supplied SQL. |
| `output_log` variable | Holds the complete textual result (including any error messages) from the Impala query. |
| `mailx` | Sends the captured output as the body of an email. The subject and recipient are hard‑coded. |
| Inline SQL query | Performs a left‑outer join across three tables to produce a per‑file, per‑call‑type summary for the previous day. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Description |
|----------|-------------|
| **Inputs** | • Impala server IP (`192.168.124.90`) – static in script.<br>• Impala tables: `mnaas.traffic_file_record_count`, `traffic_details_raw_daily`, `traffic_details_raw_reject_daily`.<br>• Implicit date: “yesterday” derived from Impala’s `now()` function.<br>• Local environment must have `impala-shell` and `mailx` in `$PATH`. |
| **Outputs** | • Email sent to `anusha.gurumurthy@contractor.tatacommunications.com` containing the raw query result (or error text). |
| **Side Effects** | • Network traffic to Impala daemon.<br>• SMTP traffic via `mailx` (relies on local MTA configuration). |
| **Assumptions** | • Impala daemon reachable from the host running the script.<br>• User executing the script has permission to query the referenced tables.<br>• Local MTA is correctly configured to deliver external mail.<br>• No special environment variables are required (none are referenced). |

---

## 4. Interaction with the Wider Move‑Mediation Ecosystem

| Connection Point | How `test.sh` fits |
|------------------|--------------------|
| **Reporting suite** | Mirrors the data‑extraction pattern used in other reporting scripts (e.g., `product_status_report.sh`). It can be used as a sanity‑check before or after scheduled jobs like `runGBS.sh` or `service_based_charges.sh`. |
| **Data warehouse** | Reads the same MNAAS tables that downstream ETL jobs populate, providing a quick view of the most recent load. |
| **Operational monitoring** | The email destination is a developer/analyst address; the script can be invoked manually when a ticket is opened to verify data integrity. |
| **Potential callers** | Could be called from a wrapper script or cron entry for ad‑hoc diagnostics; no explicit integration points are coded. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Impala host unreachable** | No data returned; email contains connection error. | Add a connectivity check (`nc -zv 192.168.124.90 21000`) before running the query; alert on failure. |
| **Large result set** | Email may be truncated or bounce; increased load on Impala. | Limit rows (`LIMIT 1000`) for ad‑hoc runs or pipe output to a file and attach only when size < threshold. |
| **Hard‑coded credentials/recipients** | Maintenance overhead; changes require script edit. | Externalize IP, recipient list, and subject into a config file or environment variables. |
| **No error handling** | Script always sends email, even if the query fails. | Check `$?` after `impala-shell`; if non‑zero, prepend “ERROR:” to email subject or route to a different address. |
| **Mailx not configured** | Email silently dropped. | Verify local MTA status; optionally fallback to `sendmail` or an API‑based mailer. |

---

## 6. Running & Debugging the Script

1. **Basic execution**  
   ```bash
   ./test.sh
   ```
   - Observe the email inbox of the hard‑coded recipient.

2. **Dry‑run / view query output locally**  
   ```bash
   ./test.sh > /tmp/ra_report.txt 2>&1
   cat /tmp/ra_report.txt
   ```
   - This bypasses the mail step (remove the `mailx` line or comment it out).

3. **Check exit status**  
   ```bash
   ./test.sh
   echo "Impala exit code: $?"
   ```
   - Non‑zero indicates a failure; add `set -e` at the top of the script for early abort.

4. **Verbose debugging**  
   ```bash
   set -x   # enable shell tracing
   ./test.sh
   set +x
   ```
   - Shows the exact command line passed to `impala-shell`.

5. **Validate Impala connectivity**  
   ```bash
   impala-shell -i 192.168.124.90 -q "SELECT 1"
   ```

---

## 7. External Configuration / Environment Dependencies

| Item | Current Usage | Suggested Externalization |
|------|---------------|---------------------------|
| Impala host (`192.168.124.90`) | Hard‑coded in the `impala-shell -i` argument. | Read from an environment variable `IMPALA_HOST` or a config file (`/etc/move/conf.d/impala.cfg`). |
| Email recipient | Hard‑coded in `mailx -s "...".` | Use a variable `RA_REPORT_RECIPIENT` to allow multiple stakeholders. |
| Email subject | Fixed string “RA Report Output”. | Parameterize via `RA_REPORT_SUBJECT`. |
| Date filter logic | Embedded in SQL (`adddate(now(), -1)`). | Accept an optional command‑line argument `--date YYYYMMDD` for flexibility. |

---

## 8. Suggested TODO / Improvements (short list)

1. **Add parameterization & error handling** – Refactor the script to accept host, date, and recipient as arguments or environment variables, and abort with a clear error message if the Impala query fails.
2. **Introduce logging** – Write the raw query output and any error messages to a rotating log file (`/var/log/move/ra_report.log`) before emailing, enabling post‑mortem analysis without relying on email delivery. 

--- 

*End of documentation for `move-mediation-scripts/bin/test.sh`.*