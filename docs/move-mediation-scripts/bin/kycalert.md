**File:** `move-mediation-scripts/bin/kycalert.sh`  

---

### 1. Summary
`kycalert.sh` is a daily monitoring script that validates the ingestion of KYC (Know‑Your‑Customer) files for the previous calendar day. It queries an Impala table (`mnaas.kyc_inter_raw_daily`) to count distinct file names that match the target date, writes the result to a CSV file, and raises email alerts if the count is zero or falls below the expected minimum (24 files). The script is used in production to detect upstream feed failures early and notify the data‑engineering team.

---

### 2. Core Logic (functions / logical blocks)

| Block | Responsibility |
|-------|-----------------|
| **Date preparation** | Computes `querydate` as *yesterday* in `YYYYMMDD` format and exports it as a Hive variable (`kycdate`). |
| **Impala query** | Executes an Impala‑shell command that selects `COUNT(DISTINCT filename)` and a substring (hour) from `mnaas.kyc_inter_raw_daily` where `filename` contains `querydate`. Results are written to `/Input_Data_Staging1/MNAAS_DailyFiles_v1/staging/kycmediation/config/kycfeedcount.csv`. |
| **Result extraction** | Reads the first column of the CSV (`processedfile`) to obtain the total distinct file count. |
| **Threshold checks & alerting** | • If the query succeeded and `processedfile` is non‑empty: <br> ‑ If `< 24` → send “count below threshold” alert. <br> ‑ If `= 0` (empty) → send “no files processed” alert. <br>• If the Impala query fails → send a generic “query error” alert. |
| **Mail dispatch** | Uses `mailx` (or `mail`) to send alerts to a static list of recipients with a concise message. |

*Note:* The script does not define reusable functions; each logical block is a sequential Bash command.

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | • System date (used to compute `querydate`). <br>• Impala cluster reachable at `mtthdoped01-ed02-impala-lb.intl.vsnl.co.in:21000`. <br>• Hive/Impala table `mnaas.kyc_inter_raw_daily`. |
| **Outputs** | • CSV file: `/Input_Data_Staging1/MNAAS_DailyFiles_v1/staging/kycmediation/config/kycfeedcount.csv` (contains two columns: count, hour substring). <br>• Email alerts (sent via `mailx`/`mail`). |
| **Side Effects** | • Writes/overwrites the CSV file on a shared staging directory. <br>• Generates email traffic to the data‑engineering team. |
| **Assumptions** | • Impala service is up and the query returns exactly two columns separated by commas. <br>• The staging directory exists and is writable by the script’s user. <br>• `mailx`/`mail` are configured with a working MTA. <br>• The expected minimum file count for a complete day is **24** (one per hour). |

---

### 4. Integration Points

| Connected Component | How `kycalert.sh` interacts |
|---------------------|------------------------------|
| **Up‑stream KYC ingestion pipelines** | Those pipelines drop raw KYC files into a location that eventually populates `mnaas.kyc_inter_raw_daily`. If they fail, `kycalert.sh` will detect the low count and raise an alarm. |
| **Down‑stream reporting / SLA dashboards** | The CSV (`kycfeedcount.csv`) may be consumed by other monitoring scripts or BI dashboards to display daily ingestion health. |
| **Alerting / ticketing system** | Emails are sent to a static list; downstream processes (e.g., an Ops ticketing webhook) may parse these messages. |
| **Scheduler (e.g., cron, Oozie, Airflow)** | The script is expected to be scheduled to run once per day (typically early morning) after the previous day’s data should be available. |
| **Configuration repository** | Hostname, output path, and email list are hard‑coded; other scripts in the repo use similar patterns (see `hadoop_move_script.sh`, `compression.sh`). |

---

### 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Impala connectivity failure** | No query → false alarm or missed alert. | Add retry logic with exponential back‑off; monitor Impala health separately; log the exact error code. |
| **CSV path permission / disk full** | Script cannot write results → downstream alerts may be inaccurate. | Verify directory existence and write permission before query; add a pre‑flight disk‑space check. |
| **Hard‑coded thresholds & recipients** | Changes require script edit → risk of outdated alerts. | Externalize threshold (`MIN_KYC_FILES=24`) and recipient list into a properties file or environment variables. |
| **Mail delivery failure** | Operators not notified. | Capture `mailx` exit status; fallback to an alternative channel (e.g., Slack webhook) if mail fails. |
| **Date calculation edge cases (DST, leap seconds)** | Wrong `querydate` → query on wrong partition. | Use UTC consistently (`date -u -d '-1 day'`) and document the timezone expectation. |
| **Unexpected query result format** | `cut -d',' -f1` may pick wrong field if schema changes. | Explicitly select only the count column (`SELECT COUNT(DISTINCT filename) AS cnt …`) and output CSV with header disabled (`-B` already does this). |

---

### 6. Running / Debugging the Script

1. **Manual execution**  
   ```bash
   $ cd move-mediation-scripts/bin
   $ ./kycalert.sh
   ```
   - Verify the printed “Running for …” line shows the expected date.
   - After execution, inspect the CSV:
     ```bash
     cat /Input_Data_Staging1/MNAAS_DailyFiles_v1/staging/kycmediation/config/kycfeedcount.csv
     ```
   - Check mail logs (`/var/log/maillog` or equivalent) for sent alerts.

2. **Dry‑run / verbose mode (add for debugging)**  
   - Temporarily prepend `set -x` at the top of the script to trace each command.
   - Capture the Impala query output directly:
     ```bash
     impala-shell ... -q "SELECT …" -B -o /tmp/debug.csv
     ```

3. **Common failure checks**  
   - **Impala unreachable:** `impala-shell -i <host> -q "SELECT 1"` should return `1`.  
   - **Directory writable:** `touch /Input_Data_Staging1/.../kycfeedcount.csv` as the script user.  
   - **Mail command functional:** `echo test | mailx -s "test" you@example.com`.

---

### 7. External Configuration / Environment Variables

| Item | Current usage | Suggested externalization |
|------|---------------|---------------------------|
| Impala host (`mtthdoped01-ed02-impala-lb.intl.vsnl.co.in:21000`) | Hard‑coded in the `impala-shell` command. | Move to `KYCA_IMPALA_HOST` env var or a properties file. |
| Output CSV path (`/Input_Data_Staging1/.../kycfeedcount.csv`) | Hard‑coded. | Parameterize as `KYCA_OUTPUT_FILE`. |
| Email recipients | Inline list in `mailx` commands. | Store in a config file (`kyc_alert.recipients`) or env var (`KYCA_ALERT_TO`). |
| Minimum expected file count (`24`) | Inline numeric comparison. | Configurable `KYCA_MIN_FILES`. |
| Date calculation timezone | Uses system `date`. | Explicit `date -u` or a TZ env var (`KYCA_TZ`). |

---

### 8. Suggested Improvements (TODO)

1. **Parameterize all hard‑coded values** (host, output path, thresholds, recipients) via a single external configuration file or environment variables to simplify maintenance and promote reuse across environments (dev / prod).

2. **Add structured logging** (e.g., write JSON lines to `/var/log/kycalert.log`) that records start time, query date, query status, file count, and any error messages. This will aid in post‑mortem analysis and enable integration with centralized log aggregation tools (Splunk, ELK).