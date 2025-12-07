**File:** `move-mediation-scripts\bin\MNAAS_Daily_RAReports.sh`

---

## 1. High‑Level Summary
This Bash wrapper launches the Java batch job `RAReport.jar` to generate the daily “RA” (Revenue Assurance) report for the MNAAS platform. It sources a common property file for configuration, redirects error output to a log file defined in that property file, and writes start/end markers to the same log via `logger`. The script is intended to be invoked as part of the nightly batch schedule.

---

## 2. Key Functions / Components  

| Name | Type | Responsibility |
|------|------|----------------|
| `MNAAS_Run_Daily_Report` | Bash function | Calls `java -jar RAReport.jar` with fixed arguments (`null null … daily Y`), then logs a success message. |
| Property source line (`. /app/hadoop_users/.../MNAAS_Java_Batch.properties`) | Bash include | Loads environment variables used throughout the script (e.g., `$MNAAS_RAReport_LogPath`). |
| `exec 2>>$MNAAS_RAReport_LogPath` | Bash redirection | Redirects *all* subsequent STDERR from the script to the designated log file. |
| Main block (logger calls + function invocation) | Bash script body | Emits start‑time markers, identifies the job, and triggers the report function. |

No classes are defined – the only executable code lives in the single Bash function.

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | • Implicit date arguments are passed as `null` to the Java jar (the jar decides the target date, typically “previous day”).<br>• Configuration values from `MNAAS_Java_Batch.properties` (e.g., `$MNAAS_RAReport_LogPath`). |
| **Outputs** | • The Java job writes its own report files (location defined inside the jar’s config).<br>• Log entries appended to `$MNAAS_RAReport_LogPath`. |
| **Side Effects** | • Potential creation of report files on HDFS / local FS (as coded inside `RAReport.jar`).<br>• Email notification (the last argument `Y` tells the jar to send mail). |
| **Assumptions** | • `MNAAS_Java_Batch.properties` exists, is readable, and defines `$MNAAS_RAReport_LogPath`.<br>• The user executing the script has execute permission on the jar and write permission on the log path.<br>• Java runtime is available and compatible with the jar.<br>• Network connectivity to any downstream systems (mail server, DB, etc.) required by the jar. |

---

## 4. Integration Points & Connectivity  

| Connected Component | How the Script Interacts |
|---------------------|--------------------------|
| **`MNAAS_Java_Batch.properties`** | Sourced at the top; provides environment variables (log path, DB connection strings, mail settings, etc.). |
| **`RAReport.jar`** | The core processing engine; invoked with arguments `null null <prop‑file> <log‑path> daily Y`. All business logic (data extraction, transformation, report generation, email) lives here. |
| **System logger (`logger`)** | Writes start/end markers to the same log file via STDERR redirection. |
| **Scheduler (e.g., cron, Oozie, Control-M)** | Not shown in the script but typical production usage: the script is scheduled to run nightly. |
| **Other “MNAAS” batch scripts** | Follow the same pattern (source same property file, redirect to a script‑specific log, invoke a dedicated jar). They share configuration and may run sequentially or in parallel as part of the nightly batch suite. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| Missing or malformed `MNAAS_Java_Batch.properties` | Job fails before launching the jar; no logs or reports. | Validate file existence and required variables at script start; abort with clear error. |
| `$MNAAS_RAReport_LogPath` not writable | STDERR and logger output lost; downstream monitoring blind. | Pre‑run `touch`/`chmod` test; alert if permission denied. |
| `RAReport.jar` throws an unhandled exception | No report generated; email may not be sent. | Capture Java exit code; if non‑zero, write explicit error to log and optionally trigger an alert. |
| Java version mismatch | Runtime errors, class‑path issues. | Pin Java version in a wrapper (e.g., `JAVA_HOME` check) and document required version. |
| Email delivery failure (argument `Y`) | Stakeholders not notified. | Ensure mail server reachable; consider adding retry logic or fallback notification (e.g., Slack). |
| Hard‑coded `null` arguments | Future changes to date handling may be missed. | Parameterize dates via script arguments or environment variables. |

---

## 6. Running & Debugging the Script  

1. **Standard execution** (as scheduled):  
   ```bash
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_Daily_RAReports.sh
   ```

2. **Manual run with debug tracing**:  
   ```bash
   set -x   # enable Bash trace
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_Daily_RAReports.sh
   set +x
   ```

3. **Check logs** (path defined in the property file):  
   ```bash
   tail -f "$MNAAS_RAReport_LogPath"
   ```

4. **Verify Java job exit status** (modify script temporarily):  
   ```bash
   java -jar ... ; RC=$?
   logger -s "RAReport.jar exit code: $RC" 2>>"$MNAAS_RAReport_LogPath"
   ```

5. **Common troubleshooting steps**  
   - Ensure the property file is readable: `ls -l /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Java_Batch.properties`  
   - Verify the jar exists and is executable: `ls -l /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars/RAReport.jar`  
   - Confirm Java runtime: `java -version`  
   - Test mail sending independently (if possible) using the same SMTP settings from the property file.

---

## 7. External Configuration & Environment Variables  

| Variable / File | Purpose | Where it is used |
|-----------------|---------|------------------|
| `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Java_Batch.properties` | Central configuration for all MNAAS batch jobs. Expected to define at least:<br>• `MNAAS_RAReport_LogPath` – path for STDERR and logger output.<br>• Database connection strings, mail server settings, etc., consumed by `RAReport.jar`. | Sourced at script start (`.` command) and referenced directly (`$MNAAS_RAReport_LogPath`). |
| `$MNAAS_RAReport_LogPath` | Destination for all log entries generated by this script and the Java jar. | `exec 2>>$MNAAS_RAReport_LogPath` and `logger -s … 2>>$MNAAS_RAReport_LogPath`. |
| Implicit environment (e.g., `JAVA_HOME`, `PATH`) | Required for the `java` command to run. | Not referenced directly but assumed to be set on the host. |

If additional variables are added to the property file (e.g., `RA_REPORT_OUTPUT_DIR`), the Java jar will read them; the Bash wrapper does not need to reference them.

---

## 8. Suggested Improvements (TODO)

1. **Parameterize date arguments** – replace the hard‑coded `null null` with optional script arguments (e.g., `--date YYYY-MM-DD` and `--month YYYY-MM`) and pass them to the jar, improving flexibility and testability.

2. **Add robust error handling** – capture the Java process exit code, log failures explicitly, and exit with a non‑zero status so that the scheduler can detect and retry or raise alerts.

---