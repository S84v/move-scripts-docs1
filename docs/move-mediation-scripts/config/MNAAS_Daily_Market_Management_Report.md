# Summary
Defines all runtime variables required by the **MNAAS_Daily_Market_Management_Report** batch job. The variables include status‑file location, log file path, Python implementation, shell script name, Hive table reference, retention policy, notification recipients, export SQL query, and CSV export destination. The file is sourced by the daily market‑management‑report cron job to drive data extraction, Hive refresh, file export, and alerting in the telecom move production environment.

# Key Components
- **MNAAS_Daily_Market_Management_Report_ProcessStatusFileName** – full path to the process‑status flag file.  
- **MNAAS_Daily_Market_Mgmt_Report_logpath** – log file path (includes date suffix).  
- **MNAAS_Daily_Market_Mgmt_Report_Pyfile** – absolute path to the Python script that builds the report.  
- **MNAAS_Daily_Market_Mgmt_Report_Script** – name of the wrapper shell script executed by cron.  
- **MNAAS_Daily_Market_Mgmt_Report_Summation_Refresh** – Hive `REFRESH` command for the target table.  
- **MNAAS_Daily_Market_Mgmt_Report_tblname** – Hive table identifier (`market_management_report_sims`).  
- **DropNthMonthOlderPartition** – Java class used by the retention utility.  
- **mnaas_retention_period_market_mgmt_report** – retention window (months).  
- **SDP_ticket_from_email** – sender address for ticket notifications.  
- **T0_email** – comma‑separated list of primary recipients for success/failure alerts.  
- **Monthly_Markt_management_Export_Query** – HiveQL query that extracts the previous month’s data.  
- **Monthly_Markt_management_Export_Path** – filesystem path for the exported CSV file.  

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| 1. Status Init | None | Shell script creates `ProcessStatusFileName` (empty or “STARTED”). | Status file created. |
| 2. Logging | `MNAAS_Daily_Market_Mgmt_Report_logpath` | All stdout/stderr redirected. | Log file with timestamp. |
| 3. Data Extraction | Hive table `market_management_report_sims` | Python script executes `Monthly_Markt_management_Export_Query`. | CSV written to `Monthly_Markt_management_Export_Path`. |
| 4. Hive Refresh | Hive table name | Executes `MNAAS_Daily_Market_Mgmt_Report_Summation_Refresh`. | Table metadata refreshed. |
| 5. Retention | Hive partitions | Calls `DropNthMonthOlderPartition` with `mnaas_retention_period_market_mgmt_report`. | Old partitions dropped. |
| 6. Notification | Status & log | Sends email from `SDP_ticket_from_email` to `T0_email`. | Alert email (success/failure). |
| 7. Completion | None | Updates status file to “COMPLETED”. | Final status flag. |

External services:
- Hive metastore (query & refresh).  
- Email SMTP server (notification).  
- Filesystem (log, CSV export, status file).  

# Integrations
- **MNAAS_CommonProperties.properties** – provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`).  
- **MNAAS_Daily_Market_Management_Report.sh** – wrapper script that sources this properties file and invokes the Python module.  
- **market_mgmt_report.py** – consumes the query variable and writes the CSV.  
- **DropNthMonthOlderPartition** – invoked via a Java/Scala retention utility (likely a Spark job).  
- **Cron scheduler** – triggers the shell script nightly.  

# Operational Risks
- **Missing or incorrect base paths** (`MNAASConfPath`, `MNAASLocalLogPath`) → job fails to write logs/status. *Mitigation*: Validate paths at start of script.  
- **Merge conflict markers** (`<<<<<<< HEAD`, `>>>>>>>`) present in file → syntax error when sourced. *Mitigation*: Clean file in version control; add CI lint step.  
- **Hard‑coded email list** – changes require code edit. *Mitigation*: Externalize to a separate config or distribution list.  
- **Retention period mismatch** (e.g., negative value) → unintended data loss. *Mitigation*: Guard against non‑positive values before invoking retention utility.  
- **SQL query assumes previous month exists** – edge case on month‑rollover may return empty set. *Mitigation*: Add fallback handling in Python script.  

# Usage
```bash
# Source the properties (usually done inside the wrapper script)
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/config/MNAAS_Daily_Market_Management_Report.properties

# Run the wrapper script with debugging enabled
set -x   # optional, uncomment setparameter line in the properties file
./MNAAS_Daily_Market_Management_Report.sh
```
To debug, comment out `setparameter='set -x'` in the properties file, then manually execute the Python script:
```bash
python3 $MNAAS_Daily_Market_Mgmt_Report_Pyfile \
    --query "$Monthly_Markt_management_Export_Query" \
    --output "$Monthly_Markt_management_Export_Path"
```

# Configuration
- **File**: `MNAAS_Daily_Market_Management_Report.properties` (this document).  
- **Referenced file**: `MNAAS_CommonProperties.properties` (defines `MNAASConfPath`, `MNAASLocalLogPath`).  
- **Environment variables** (inherited from common properties):  
  - `MNAASConfPath` – directory for status files.  
  - `MNAASLocalLogPath` – base directory for logs.  
- **External constants**:  
  - `DropNthMonthOlderPartition` – Java class name for retention.  
  - `mnaas_retention_period_market_mgmt_report` – integer months.  

# Improvements
1. **Remove merge conflict artifacts** and enforce a lint step that rejects files containing `<<<<<<<`, `=======`, or `>>>>>>>`.  
2. **External