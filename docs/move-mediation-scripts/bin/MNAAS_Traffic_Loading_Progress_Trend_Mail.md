**File:** `move-mediation-scripts/bin/MNAAS_Traffic_Loading_Progress_Trend_Mail.sh`

---

### 1. Summary
This Bash script generates a daily status report of the “traffic” CDR files that have been staged for loading into the MNAAS mediation platform. It reads a list of directory paths from a property file, counts and lists the traffic files present in each location, writes a formatted summary to a log file, and emails that log to a predefined distribution list. It is typically invoked by a cron job after the nightly traffic‑file ingestion jobs have run, providing operations staff with a quick visual check of file‑landing progress and any anomalies.

---

### 2. Key Elements & Responsibilities

| Element | Type | Responsibility |
|---------|------|-----------------|
| `MNAAS_CDRs_trend_check_mail.properties` | External property file | Supplies the array `MNAAS_Traffic_File_Paths` (list of directories to scan) and any other environment‑specific variables. |
| `Traffic_Files_Loading_Progress_log` | Variable (path) | Destination log file that receives the formatted report (`/app/hadoop_users/MNAAS/MNAASCronLogs/Traffic_Files_Loading_Progress.log`). |
| `today_date` | Variable | Holds the current date string used in the report header. |
| `for path in ${MNAAS_Traffic_File_Paths[@]}` | Loop | Iterates over each configured traffic directory. |
| `count=$(ls $path | grep -i traffic | grep -v non | wc -l)` | Command | Counts files whose names contain “traffic” (case‑insensitive) but exclude those containing “non”. |
| `ls -lrth ${path}*01*Traffic*` | Command | Lists detailed file information for files matching the pattern `*01*Traffic*` (used for a sample view). |
| `mailx -s "MNAAS Traffic Loading progress Trend Mail" …` | Command | Sends the generated log as the email body to the hard‑coded recipient list. |
| `set -x` | Bash option | Enables command‑trace debugging (useful for manual runs). |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | • Property file: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CDRs_trend_check_mail.properties`  <br>• Filesystem directories defined in `MNAAS_Traffic_File_Paths` (must be readable). |
| **Outputs** | • Log file: `/app/hadoop_users/MNAAS/MNAASCronLogs/Traffic_Files_Loading_Progress.log` (overwritten each run). <br>• Email sent via `mailx` to a static list of Tata Communications addresses. |
| **Side Effects** | • Reads directory listings (potentially large I/O). <br>• Generates network traffic for email delivery. |
| **Assumptions** | • `mailx` is correctly configured (SMTP relay, authentication). <br>• The property file defines `MNAAS_Traffic_File_Paths` as a Bash array. <br>• The script runs with a user that has read permission on all traffic directories and write permission on the log directory. |

---

### 4. Integration Points & Call Flow

| Connected Component | Interaction |
|---------------------|-------------|
| **Up‑stream ingestion scripts** (e.g., `MNAAS_Traffic_tbl_Load.sh`, `MNAAS_TrafficDetails_*`) | These scripts place traffic files into the directories scanned by this script. |
| **Cron scheduler** | Typically scheduled (e.g., nightly at 02:00) to run after ingestion jobs complete. |
| **Monitoring/Alerting platform** | The email recipients may forward the report to a ticketing system or dashboard; the script itself does not push metrics elsewhere. |
| **Property management** | The same property file is shared with other “trend‑check” scripts (e.g., `MNAAS_Tolling_seq_check.sh`) to keep directory lists consistent. |
| **Log rotation** | External log‑rotation (e.g., `logrotate`) may act on `Traffic_Files_Loading_Progress.log`. |

---

### 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Missing or malformed property file** | Script aborts, no report generated. | Add a pre‑run check: `if [ ! -f "$PROP_FILE" ]; then echo "Missing properties"; exit 1; fi`. |
| **Empty `MNAAS_Traffic_File_Paths` array** | Loop does nothing; email contains only header. | Validate array length before entering loop and warn if zero. |
| **Large directory listings** (hundreds of thousands of files) | `ls` and `wc -l` become slow, may exceed cron timeout. | Use `find -maxdepth 1 -type f -iname '*traffic*' ! -iname '*non*' | wc -l` for faster counting; optionally limit listing to recent files. |
| **Mail delivery failure** (SMTP down, auth error) | Stakeholders never receive the status. | Capture `mailx` exit code; on failure, write to a fallback log and optionally trigger an alert. |
| **Hard‑coded recipient list** | Maintenance overhead when personnel change. | Externalize recipient list to a property file or environment variable. |
| **Log file overwritten each run** | Loss of historical trend data. | Rotate logs (e.g., `mv $log ${log}.$(date +%Y%m%d)` before writing) or append with timestamps. |

---

### 6. Running / Debugging the Script

1. **Manual execution** (useful for troubleshooting):  
   ```bash
   cd /app/hadoop_users/MNAAS/
   ./bin/MNAAS_Traffic_Loading_Progress_Trend_Mail.sh
   ```
   The `set -x` flag will print each command as it runs.

2. **Check environment** before running:  
   ```bash
   source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CDRs_trend_check_mail.properties
   echo "${MNAAS_Traffic_File_Paths[@]}"
   ```
   Verify that the array contains the expected directories.

3. **Validate mail**:  
   ```bash
   echo "test body" | mailx -s "Test MNAAS Traffic Mail" you@example.com
   ```
   Ensure `mailx` can send mail from the host.

4. **Inspect log** after execution:  
   ```bash
   less /app/hadoop_users/MNAAS/MNAASCronLogs/Traffic_Files_Loading_Progress.log
   ```

5. **Cron verification**:  
   Check the cron entry (e.g., `crontab -l | grep Traffic_Loading_Progress`) and confirm the user context matches the script’s permissions.

---

### 7. External Config / Environment Variables

| Variable / File | Purpose | Typical Location |
|-----------------|---------|------------------|
| `MNAAS_CDRs_trend_check_mail.properties` | Defines `MNAAS_Traffic_File_Paths` (Bash array) and possibly other shared settings. | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/` |
| `MNAAS_Traffic_File_Paths` | Array of absolute directory paths to scan for traffic files. | Populated by the property file. |
| `MAILX` environment (e.g., `SMTPHOST`, `MAILRC`) | Controls mail delivery behavior. | System‑wide or user‑specific (`~/.mailrc`). |
| `PATH` | Must include standard utilities (`ls`, `grep`, `wc`, `mailx`). | Inherited from the user’s shell. |

---

### 8. Suggested Improvements (TODO)

1. **Robust error handling & exit codes** – wrap the main loop in a function that returns a non‑zero status on any failure (missing directory, `ls` error, mail send failure) and log the cause.
2. **Externalize the recipient list** – move the comma‑separated email addresses to the same property file (or a dedicated `mail_recipients` variable) to simplify updates and avoid hard‑coding.

---