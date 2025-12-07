**File:** `move-mediation-scripts\bin\MNAAS_Monthly_IPvProbeReports.sh`  

---

## 1. High‑Level Summary
This Bash wrapper orchestrates the generation of the monthly IPvProbe (RA) report for the MNAAS platform. It sources a central Java‑batch property file, redirects error output to a shared log, invokes the `RAReport.jar` Java application with a fixed set of arguments (including the property file path and a log‑path identifier), and writes a success message to the same log. The script is intended to be scheduled (e.g., via cron) to run once per month.

---

## 2. Key Functions / Components  

| Name | Type | Responsibility |
|------|------|----------------|
| **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Java_Batch.properties`** | source statement | Loads environment variables (e.g., `MNAAS_RAReport_LogPath`) required by the script and the Java jar. |
| **`MNAAS_Run_Monthly_Report()`** | Bash function | Executes the Java report generator (`RAReport.jar`) with hard‑coded arguments, then logs a completion message. |
| **`java -jar ... RAReport.jar … ipv N`** | Java invocation | Runs the core reporting logic (RAReport) in “ipv” mode, with mail‑notification disabled (`N`). |
| **`logger -s …`** | Bash built‑in | Writes timestamped messages to both syslog and the script‑specific log file (`$MNAAS_RAReport_LogPath`). |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | • No command‑line arguments are consumed (the function signature documents possible args but they are ignored).<br>• Implicit inputs from the sourced property file: `MNAAS_RAReport_LogPath` and any other variables required by `RAReport.jar` (DB URLs, credentials, email settings, etc.). |
| **Outputs** | • The Java process writes its own output (likely CSV/Parquet files) to locations defined inside the property file.<br>• Success/failure messages are appended to `$MNAAS_RAReport_LogPath`. |
| **Side‑effects** | • Potentially creates/updates report files in a data lake or HDFS directory (controlled by the Java jar).<br>• May trigger downstream processes that consume the generated report. |
| **Assumptions** | • `MNAAS_Java_Batch.properties` exists and is readable by the script user.<br>• `RAReport.jar` is present at the hard‑coded path and compatible with the runtime Java version.<br>• The directory referenced by `$MNAAS_RAReport_LogPath` exists and is writable.<br>• No arguments are needed for the monthly run; the Java code defaults to “previous month”. |
| **External Services** | • Java runtime (JRE).<br>• Possibly a database or Hive metastore accessed by the jar (configured in the property file).<br>• Email service (disabled in this invocation, but configured for other modes). |

---

## 4. Integration Points  

| Component | How this script connects |
|-----------|--------------------------|
| **Other MNAAS batch scripts** (e.g., `MNAAS_Monthly_Billing_Export.sh`, `MNAAS_LateLandingCDRs_Check.sh`) | All share the same property file directory and log‑path conventions; they may be scheduled sequentially to ensure data dependencies (e.g., billing data must exist before the IPvProbe report runs). |
| **`RAReport.jar`** | Core processing engine; receives the property file path and a mode flag (`ipv`). The jar is also invoked by other scripts (e.g., daily RA reports) with different mode flags (`daily`, `ra`, `new`). |
| **Logging infrastructure** | Uses `logger -s` to push messages to syslog and the script‑specific log file, enabling centralized monitoring alongside other MNAAS scripts. |
| **Scheduler (cron / Oozie / Airflow)** | Typically invoked by a monthly cron entry; the script’s idempotent design (no arguments) simplifies scheduling. |
| **Configuration repository** | The property file is version‑controlled and shared across the suite; any change propagates to this script automatically. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Missing/Corrupt property file** | Java jar may fail to start or use wrong parameters. | Add a pre‑run check: `[[ -r "$PROP_FILE" ]] || { logger "Property file missing"; exit 1; }`. |
| **Log path not writable** | No audit trail; script may silently fail. | Verify directory existence and write permission before invoking the jar. |
| **Hard‑coded `null` arguments** | Future changes to the Java API may require actual dates; current run may produce empty reports. | Replace `null null` with derived month (`$(date -d "$(date +%Y-%m-01) -1 month" +%Y-%m)`) and pass as arguments. |
| **Jar not found or incompatible Java version** | Immediate script failure. | Check `[[ -x "$(which java)" ]]` and `[[ -f "$JAR_PATH" ]]` before execution; log version info. |
| **No error handling on Java exit code** | Failures go unnoticed. | Capture `$?` after the `java` command and abort with a non‑zero exit status if non‑zero. |
| **Mail flag hard‑coded to `N`** | Operators may expect notification. | Make the mail flag configurable via the property file or a command‑line argument. |

---

## 6. Running & Debugging the Script  

1. **Standard execution** (as scheduled):  
   ```bash
   /app/hadoop_users/MNAAS/MNAAS_Scripts/bin/MNAAS_Monthly_IPvProbeReports.sh
   ```
   The script writes progress and any errors to the file referenced by `$MNAAS_RAReport_LogPath`.

2. **Manual run with verbose tracing** (useful for debugging):  
   ```bash
   set -x   # enable Bash trace
   /app/hadoop_users/MNAAS/MNAAS_Scripts/bin/MNAAS_Monthly_IPvProbeReports.sh
   set +x
   ```

3. **Check results**:  
   - Verify the log file for the “successfully completed” message and any Java stack traces.  
   - Confirm that the expected report files exist in the location defined inside `MNAAS_Java_Batch.properties`.

4. **Common troubleshooting steps**:  
   - Ensure the property file path is correct (`cat /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Java_Batch.properties`).  
   - Validate Java version: `java -version`.  
   - Test the jar directly:  
     ```bash
     java -jar /app/hadoop_users/MNAAS/MNAAS_Jar/Loader_Jars/RAReport.jar \
          null null /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Java_Batch.properties \
          MNAAS_RAReport_LogPath ipv N
     ```

---

## 7. External Configuration & Environment Variables  

| Variable / File | Purpose | Usage in Script |
|-----------------|---------|-----------------|
| **`/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Java_Batch.properties`** | Centralized batch configuration (log paths, DB connections, email settings, etc.). | Sourced at the top of the script; defines `$MNAAS_RAReport_LogPath` and other variables consumed by the Java jar. |
| **`MNAAS_RAReport_LogPath`** (defined in the property file) | Path to the log file where both Bash and Java errors are appended. | Redirected via `exec 2>>$MNAAS_RAReport_LogPath` and used in `logger -s` calls. |
| **`JAVA_HOME` / `PATH`** (implicit) | Locate the Java runtime. | Required for `java -jar` to succeed. |
| **`RAReport.jar`** (hard‑coded path) | The executable that generates the IPvProbe report. | Invoked directly with fixed arguments. |

If any of these items are missing or mis‑configured, the script will fail early or produce incomplete output.

---

## 8. Suggested Improvements (TODO)

1. **Parameterize date/month arguments** – Replace the hard‑coded `null null` with actual values derived from script arguments or the current calendar (e.g., previous month). This will make the script robust to future changes in the Java API.

2. **Add error handling and exit‑code checks** – Capture the Java process exit status, log failures, and exit the script with a non‑zero code to allow the scheduler to raise alerts.

*(Both changes can be implemented without impacting existing scheduled runs, but will improve observability and future‑proof the script.)*