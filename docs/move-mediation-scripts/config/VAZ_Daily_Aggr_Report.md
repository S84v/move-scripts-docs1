# Summary
`VAZ_Daily_Aggr_Report.properties` is a Bash‑style configuration file for the Move mediation pipeline’s “VAZ Daily Aggregation Report” job. It imports common properties, defines script and log locations, Hive refresh statements, retention parameters, and email recipients. The driver script `VAZ_Daily_Aggr_Report.sh` sources this file before executing the Python aggregation (`vaz_aggr_report.py`) and Hive refreshes.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – sources global environment variables (`MNAASConfPath`, `MNAASLocalLogPath`, etc.).
- **`setparameter='set -x'`** – enables Bash debugging when uncommented.
- **`VAZ_Daily_Aggr_Report_ProcessStatusFileName`** – path to the job status file.
- **`VAZ_Daily_Aggr_Report_logpath`** – daily log file path with date suffix.
- **`VAZ_Daily_Aggr_Report_Pyfile`** – absolute path to the Python aggregation script.
- **`VAZ_Daily_Aggr_Report_Script`** – name of the driver shell script.
- **Hive refresh statements** (`VAZ_Daily_Aggr_Report_application_usage_Refresh`, `VAZ_Daily_Aggr_Report_Service_Maintainece_Refresh`, `VAZ_Daily_Aggr_Report_Celltower_Performance_Refresh`, `VAZ_Daily_Aggr_Report_Top_Impacted_Device_Refresh`) – SQL commands executed post‑aggregation.
- **Retention parameters**
  - `VAZ_Daily_Aggr_Report_tblname` – target Hive table for retention cleanup.
  - `DropNthMonthOlderPartition` – Java class implementing partition drop logic.
  - `mnaas_retention_period_market_mgmt_report` – retention window (months).
- **Email configuration**
  - `SDP_ticket_from_email` – sender address for SDP tickets.
  - `T0_email` – list of recipients for job notifications.

# Data Flow
| Stage | Input | Process | Output | Side Effects |
|-------|-------|---------|--------|--------------|
| 1 | Global properties file | Source variables | Environment variables (`MNAASConfPath`, etc.) | None |
| 2 | Python script `vaz_aggr_report.py` | Executes aggregation logic on source tables | Populates Hive aggregation tables (`vaz_dm_applicationusage_aggr_summation`, etc.) | Writes to Hive, updates `ProcessStatusFileName` |
| 3 | Hive refresh statements | Executed via `hive -e` | Refreshes materialized views / tables | Commits changes in Hive Metastore |
| 4 | Retention class `DropNthMonthOlderPartition` | Invoked (likely via `spark-submit` or `hadoop jar`) | Drops partitions older than `mnaas_retention_period_market_mgmt_report` months from `market_management_report_sims` | Alters Hive table partitions |
| 5 | Email variables | Used by driver script to send notifications | Success/failure emails to `T0_email` | External SMTP traffic |

# Integrations
- **`MNAAS_CommonProperties.properties`** – provides base configuration (paths, Hadoop/YARN settings).
- **`VAZ_Daily_Aggr_Report.sh`** – driver script that sources this file, runs the Python job, executes Hive refreshes, triggers retention, and handles logging/status.
- **Hive Metastore** – target of refresh statements and retention cleanup.
- **Java class `com.tcl.retentions.DropNthMonthOlderPartition`** – invoked to manage partition deletion.
- **SMTP service** – used for email notifications.
- **Potential downstream jobs** – consume refreshed aggregation tables.

# Operational Risks
- **Missing or stale global properties** – job may fail to locate logs or status file. *Mitigation*: Validate existence of sourced file at start.
- **Debug flag left enabled** (`set -x`) – can flood logs with sensitive data. *Mitigation*: Ensure `setparameter` is commented out in production.
- **Hard‑coded email addresses** – may become outdated. *Mitigation*: Externalize recipients to a separate config or directory service.
- **Retention logic mis‑configuration** (`mnaas_retention_period_market_mgmt_report`) – could delete needed data. *Mitigation*: Add sanity check before partition drop.
- **Hive refresh failures** – downstream consumers may see stale data. *Mitigation*: Capture Hive command exit codes; abort on non‑zero status and alert.

# Usage
```bash
# Source configuration (normally done inside the driver script)
source /app/hadoop_users/MNAAS/MNAAS_CronFiles/VAZ_Daily_Aggr_Report.properties

# Run driver script with optional debug
bash -x VAZ_Daily_Aggr_Report.sh   # uncomment setparameter if needed
```
To debug a specific step, uncomment the `setparameter='set -x'` line, re‑source the file, and re‑run the driver.

# Configuration
- **Environment variables** injected by `MNAAS_CommonProperties.properties`:
  - `MNAASConfPath`
  - `MNAASLocalLogPath`
  - Hadoop/YARN related vars (e.g., `HADOOP_HOME`, `JAVA_HOME`).
- **File references**:
  - `VAZ_Daily_Aggr_Report_ProcessStatusFileName`
  - `VAZ_Daily_Aggr_Report_logpath`
  - `VAZ_Daily_Aggr_Report_Pyfile`
- **Java class**: `com.tcl.retentions.DropNthMonthOlderPartition` (must be on classpath).
- **Email**: `SDP_ticket_from_email`, `T0_email`.

# Improvements
1. **Externalize mutable parameters** (email list, retention period) to a separate properties file or key‑value store to avoid code changes for updates.
2. **Add validation block** at the top of the file to verify required variables (`MNAASConfPath`, `MNAASLocalLogPath`, Python script existence) and exit early with a clear error message.