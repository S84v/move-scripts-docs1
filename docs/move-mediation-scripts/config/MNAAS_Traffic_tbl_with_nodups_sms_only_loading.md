# Summary
Defines Bash‑sourced runtime constants for the **MNAAS_Traffic_tbl_with_nodups_sms_only_loading** job. The job loads daily raw “sms‑only” traffic detail records into Hive without duplicates, using intermediate staging tables and Java loader classes. It also configures script location, process‑status tracking, logging, and HDFS lock coordination for the mediation pipeline.

# Key Components
- **MNAAS_traffic_no_dups_sms_only_ScriptName** – absolute path to the executable shell script.
- **MNAAS_Traffic_tbl_with_nodups_sms_only_loading_ProcessStatusFileName** – HDFS file used to record job status (STARTED, SUCCESS, FAILURE).
- **MNAAS_Traffic_tbl_with_nodups_sms_only_loadingLogPath** – log file path with daily timestamp suffix.
- **traffic_details_raw_daily_with_no_dups_sms_only_tblname_refresh** – Hive `REFRESH` command string for the target table.
- **Dname_MNAAS_Load_Daily_Traffic_tbl_with_nodups_sms_only_inter** – name of the intermediate Hive database/table.
- **Dname_MNAAS_Load_Daily_Traffic_tbl_with_nodups_sms_only** – name of the final Hive database/table.
- **MNAAS_Traffic_tbl_with_nodups_sms_only_inter_loading_load_classname** – fully‑qualified Java class for loading the intermediate table.
- **MNAAS_Traffic_tbl_with_nodups_sms_only_loading_load_classname** – fully‑qualified Java class for loading the final table.
- **setparameter** – optional `set -x` for Bash debugging.

# Data Flow
1. **Input** – Daily raw SMS‑only traffic files (HDFS location defined in shared `MNAAS_CommonProperties.properties`).
2. **Process** – Shell script invokes Java loader classes:
   - Intermediate loader (`*_inter_loading`) writes to staging Hive table.
   - Final loader (`*_loading`) writes to target Hive table, deduplicating records.
3. **Side Effects** – 
   - Updates process‑status file on HDFS.
   - Writes execution log to `MNAAS_Traffic_tbl_with_nodups_sms_only_loadingLogPath`.
   - Issues Hive `REFRESH` on the target table.
4. **External Services** – Hive Metastore, HDFS, Java runtime (JARs containing the loader classes).

# Integrations
- **MNAAS_CommonProperties.properties** – provides `$dbname`, `$traffic_details_raw_daily_with_no_dups_sms_only_tblname`, HDFS base paths, and Java classpath.
- **MNAAS_Traffic_tbl_with_nodups_sms_only_loading.sh** – the orchestrating script that sources this properties file.
- **Other mediation jobs** – downstream jobs consume the refreshed Hive table; upstream jobs populate the raw SMS‑only files.

# Operational Risks
- **Missing shared properties** – job fails if `MNAAS_CommonProperties.properties` is unavailable or corrupted. *Mitigation*: health‑check script before execution.
- **Duplicate detection logic failure** – could lead to data quality issues. *Mitigation*: unit‑test Java loader classes; add post‑load row‑count validation.
- **Log file path collision** – daily log file may exceed filesystem limits. *Mitigation*: rotate logs and enforce size limits.
- **Process‑status file stale** – stale status may block re‑runs. *Mitigation*: cleanup task that removes status files older than 48 h.

# Usage
```bash
# Source constants
. /app/hadoop_users/MNAAS/MNAAS_Config/MNAAS_Traffic_tbl_with_nodups_sms_only_loading.properties

# Enable Bash tracing (optional)
set $setparameter   # expands to `set -x` if debugging is desired

# Execute the job
bash $MNAAS_traffic_no_dups_sms_only_ScriptName
```
To debug, uncomment the `setparameter` line or run the script with `bash -x`.

# Configuration
- **Environment Variables** – inherited from `MNAAS_CommonProperties.properties` (e.g., `dbname`, `hadoop_home`, `java_home`).
- **Referenced Config Files**
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
  - Process‑status file path defined above.
  - Log directory `/app/hadoop_users/MNAAS/MNAASCronLogs/`.

# Improvements
1. **Parameterize log rotation** – add `logrotate` configuration or size‑based rollover to prevent unbounded log growth.
2. **Add explicit exit‑code handling** – wrap Java class invocations with status checks and propagate standardized error codes to the orchestration layer.