# Summary
Defines runtime constants for the **Monthly Traffic No‑Duplication Summation** batch job. Supplies the process‑status file path, log file name, Python implementation, shell script name, Hive refresh command, retention policy parameters, and notification email addresses for the Move‑Mediation production environment.

# Key Components
- **`MNAAS_Monthly_Traffic_Nodups_Summation_ProcessStatusFileName`** – absolute path to the job’s process‑status file.  
- **`MNAAS_Monthly_Traffic_Nodups_Summation_logpath`** – log file location with date suffix.  
- **`MNAAS_Monthly_Traffic_Nodups_Pyfile`** – full path to the Python script that performs the aggregation (`traffic_nodups_summation.py`).  
- **`MNAAS_Monthly_Traffic_Nodups_Script`** – shell wrapper script name (`MNAAS_Monthly_traffic_nodups_summation.sh`).  
- **`MNAAS_Monthly_Traffic_Nodups_Summation_Refresh`** – Hive `REFRESH` statement for table `mnaas.traffic_details_raw_daily_with_no_dups_summation`.  
- **Retention parameters** – table name, retention period (months), and Java class for partition dropping (`DropNthMonthOlderPartition`).  
- **Notification emails** – `SDP_ticket_from_email` and `T0_email`.  
- **`setparameter`** – optional Bash debugging flag (`set -x`).  

# Data Flow
1. **Input** – Common properties sourced from `MNAAS_CommonProperties.properties`.  
2. **Processing** – Shell script (`MNAAS_Monthly_traffic_nodups_summation.sh`) invokes the Python file, which reads raw traffic data, removes duplicate records, and writes aggregated results to Hive table `traffic_details_raw_daily_with_no_dups_summation`.  
3. **Side Effects** –  
   - Updates process‑status file.  
   - Appends execution details to dated log file.  
   - Executes Hive `REFRESH` to make new partitions visible.  
   - Triggers retention job (`DropNthMonthOlderPartition`) to purge partitions older than the configured period (6 months).  
   - Sends email notifications to the addresses defined.  
4. **Outputs** – Updated Hive table, log file, process‑status file, and optional email alerts.

# Integrations
- **Hive Metastore** – Table `mnaas.traffic_details_raw_daily_with_no_dups_summation`.  
- **Retention Service** – Java class `com.tcl.retentions.DropNthMonthOlderPartition` invoked via Hive/Impala to drop old partitions.  
- **Email System** – Uses local mail client (e.g., `mailx`) to send alerts to `SDP_ticket_from_email` / `T0_email`.  
- **Common Property Repository** – Shared configuration (`MNAAS_CommonProperties.properties`) provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`).  
- **Cron Scheduler** – Typically executed as a nightly/monthly cron job referencing this property file.

# Operational Risks
- **Missing/Corrupt CommonProperties** → job fails to resolve base paths. *Mitigation*: validate existence of sourced file before execution.  
- **Incorrect log path or permission** → log file not written, obscuring troubleshooting. *Mitigation*: enforce directory ownership and write permissions.  
- **Retention period mismatch** → premature data deletion. *Mitigation*: lock retention parameters behind a change‑control process and add sanity checks.  
- **Hard‑coded email addresses** → notification failures if addresses change. *Mitigation*: externalize emails to a separate config or secret store.  
- **Hive REFRESH failure** → downstream queries see stale data. *Mitigation*: capture Hive command exit status and abort on non‑zero return.

# Usage
```bash
# Source the property file to export variables
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source /path/to/MNAAS_Monthly_traffic_nodups_summation.properties

# Optional: enable Bash debugging
set -x   # or uncomment setparameter line in the file

# Execute the wrapper script
/app/hadoop_users/MNAAS/MNAAS_CronFiles/MNAAS_Monthly_traffic_nodups_summation.sh
```
Check log:
```bash
cat "${MNAAS_Monthly_Traffic_Nodups_Summation_logpath}"
```
Verify process‑status:
```bash
cat "${MNAAS_Monthly_Traffic_Nodups_Summation_ProcessStatusFileName}"
```

# Configuration
- **Environment Variables** (populated by `MNAAS_CommonProperties.properties`):  
  - `MNAASConfPath` – root config directory.  
  - `MNAASLocalLogPath` – base log directory.  
- **Referenced Files**:  
  - `MNAAS_CommonProperties.properties` (shared constants).  
  - `traffic_nodups_summation.py` (Python aggregation logic).  
  - `MNAAS_Monthly_traffic_nodups_summation.sh` (shell wrapper).  
- **Hard‑coded values**:  
  - Hive table name (`traffic_details_raw_daily_with_no_dups_summation`).  
  - Retention period (`6` months).  
  - Email addresses (`Mohan.koripalli@tatacommunications.com`).  

# Improvements
1. **Externalize mutable parameters** (email addresses, retention period, Hive table name) into a dedicated environment‑specific config file or secret manager to avoid code changes for operational updates.  
2. **Add validation logic** at the start of the wrapper script to verify that all required paths exist, Hive table is reachable, and the retention class is loadable; abort with clear error codes if any check fails.