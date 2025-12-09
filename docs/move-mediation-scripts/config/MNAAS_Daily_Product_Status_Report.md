# Summary
Defines runtime constants for the **MNAAS_Daily_Product_Status_Report** batch job. The properties are sourced by the daily product‑status‑report cron job to configure script execution, logging, HDFS and SFTP locations, and backup paths used in the telecom move production environment.

# Key Components
- `MNAAS_Daily_Product_Status_Report_ProcessStatusFileName` – full path to the process‑status file used for idempotency and monitoring.  
- `MNAAS_Daily_Product_Status_Report_logpath` – log file location with date suffix.  
- `MNAAS_Product_Status_Report_Pyfile` – absolute path to the Python implementation (`product_status_report.py`).  
- `MNAAS_Product_Status_Report_Script` – shell wrapper script name (`product_status_report.sh`).  
- `MNAAS_Product_Status_Report_hdfs_path` – target HDFS directory for report output.  
- `MNAAS_Product_Status_Report_sftp_path` – final SFTP destination for successful exports.  
- `MNAAS_Product_Status_Report_sftp_path_temp` – temporary SFTP staging directory.  
- `MNAAS_Product_Status_Report_back_path` – backup directory for archived report files.  
- `setparameter='set -x'` – optional Bash debug flag (commented out to disable).  

# Data Flow
1. **Input** – Source data is read by `product_status_report.py` (not defined here) from internal databases/Hive.  
2. **Processing** – Python script generates a CSV/report and writes to `$MNAAS_Product_Status_Report_hdfs_path`.  
3. **Export** – Shell wrapper moves the file from HDFS to `$MNAAS_Product_Status_Report_sftp_path_temp`, then to `$MNAAS_Product_Status_Report_sftp_path`.  
4. **Backup** – A copy is stored in `$MNAAS_Product_Status_Report_back_path`.  
5. **Status & Logging** – Process status is written to `$MNAAS_Daily_Product_Status_Report_ProcessStatusFileName`; execution details are appended to `$MNAAS_Daily_Product_Status_Report_logpath`.  

# Integrations
- **MNAAS_CommonProperties.properties** – sourced at the top to obtain base paths (`MNAASConfPath`, `MNAASLocalLogPath`).  
- **Cron scheduler** – daily cron job sources this file before invoking `product_status_report.sh`.  
- **HDFS** – interacts with `/user/MNAAS/report/`.  
- **SFTP server** – uses `/backup1/MNAAS/Crushsftp/eLUX/actives/` and temp staging area.  
- **Backup storage** – writes to `/backup1/MNAAS/Customer_CDR/eLUX_backup/actives/`.  

# Operational Risks
- **Path misconfiguration** – incorrect base paths cause file‑not‑found errors; mitigate with validation scripts during deployment.  
- **Permission issues** – HDFS, SFTP, and backup directories must be writable by the cron user; enforce via ACL checks.  
- **Log rotation overflow** – daily log files accumulate; implement logrotate or size‑based cleanup.  
- **Debug flag leakage** – leaving `set -x` enabled in production may expose sensitive data; ensure it remains commented out.  

# Usage
```bash
# Source the properties (executed by cron)
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_Product_Status_Report.properties

# Run with debug (optional)
# set -x   # uncomment setparameter line in the properties file
./product_status_report.sh
```

# Configuration
- **Environment variables** imported from `MNAAS_CommonProperties.properties`:
  - `MNAASConfPath`
  - `MNAASLocalLogPath`
- **Referenced config files**:
  - `MNAAS_CommonProperties.properties` (base paths)
  - This file itself (`MNAAS_Daily_Product_Status_Report.properties`)  

# Improvements
1. **Validate paths at load time** – add a Bash function that checks existence and write permission of all directories; abort with clear error if any check fails.  
2. **Externalize debug flag** – replace hard‑coded `setparameter` with an environment variable (e.g., `MNAAS_DEBUG`) to avoid accidental production exposure.