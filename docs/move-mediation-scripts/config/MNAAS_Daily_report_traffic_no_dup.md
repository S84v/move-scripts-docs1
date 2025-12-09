# Summary
Defines runtime variables for the **MNAAS_Daily_report_traffic_no_dup** batch job. The properties are sourced by the corresponding shell script to locate status files, logs, Hive partitions, HDFS staging, and SFTP export paths used in the daily traffic‑report generation pipeline of the telecom Move production environment.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – imports shared configuration (base paths, Hadoop settings, etc.).
- **`MNAAS_Daily_report_traffic_no_dup_ProcessStatusFileName`** – full path to the process‑status file for idempotency and monitoring.
- **`MNAAS_Daily_report_traffic_no_dup_log_file_path`** – log file location with date suffix.
- **`MNAAS_Daily_report_traffic_no_dup_scriptName`** – name of the executable shell script.
- **`MNAAS_Traffic_Partitions_Daily_Uniq_latest_3_FileName`** – reference file for the latest three unique traffic partitions in Hive.
- **`SSH_MOVE_DAILY_PATH`** – remote SFTP directory for daily CDR upload.
- **`STAGING_DIR`** – local HDFS staging directory for intermediate “no‑dup” report files.

# Data Flow
| Phase | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| **Init** | `MNAAS_CommonProperties.properties` | Source shared variables (`MNAASConfPath`, `MNAASLocalLogPath`, etc.) | Populated environment variables | None |
| **Script Execution** (`MNAAS_Daily_report_traffic_no_dup.sh`) | Process‑status file, Hive partition file, raw CDR files in `SSH_MOVE_DAILY_PATH` | Extract, deduplicate, aggregate traffic data; write to `STAGING_DIR` | CSV/Parquet report files, updated status file, log file at `*_log_file_path*` | HDFS writes, possible SFTP upload |
| **Post‑process** | Generated reports | Archive to backup, notify recipients (outside this file) | Archived copies, alerts | Network I/O, email/SMS |

# Integrations
- **MNAAS_CommonProperties.properties** – provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`) and Hadoop configuration.
- **MNAAS_Daily_report_traffic_no_dup.sh** – consumes all variables defined herein.
- **Hive Metastore** – accessed via `MNAAS_Traffic_Partitions_Daily_Uniq_latest_3_FileName` for partition metadata.
- **Remote SFTP server** (`SSH_MOVE_DAILY_PATH`) – source of daily CDR files.
- **HDFS** (`STAGING_DIR`) – intermediate storage for processed reports.

# Operational Risks
- **Path misconfiguration** – incorrect `MNAASConfPath` leads to missing status file; mitigate with validation script at start of job.
- **Log file rotation** – date suffix may create unlimited files; mitigate by adding log retention policy.
- **Staging directory overflow** – large daily reports can exhaust disk; mitigate with size monitoring and automatic cleanup.
- **SFTP connectivity loss** – prevents input data ingestion; mitigate with retry logic and alerting.

# Usage
```bash
# Load properties
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_report_traffic_no_dup.properties

# Verify variables
echo $MNAAS_Daily_report_traffic_no_dup_ProcessStatusFileName
echo $MNAAS_Daily_report_traffic_no_dup_log_file_path

# Execute the batch job
bash $MNAAS_Daily_report_traffic_no_dup_scriptName
```
For debugging, add `set -x` in the shell script and inspect the generated log file.

# Configuration
- **Environment variables**: None required beyond those defined in the sourced common properties file.
- **Referenced config files**:  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  - Files referenced by the variables (`*_ProcessStatusFile`, `*_latest_3`) must exist in `$MNAASConfPath`.
- **Directory prerequisites**:  
  - `$MNAASLocalLogPath` writable by the job user.  
  - `$STAGING_DIR` accessible with HDFS permissions.  
  - Remote SFTP path reachable via SSH keys.

# Improvements
1. **Add validation block** – script should verify existence and write‑access of all paths at start, exiting with clear error codes.
2. **Parameterize log rotation** – replace hard‑coded date suffix with a configurable retention policy (e.g., `logrotate` config) to prevent log bloat.