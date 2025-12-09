# Summary
Defines runtime constants for the **MNAAS_msisdn_non_jlr_rate_per_country** batch job. The variables are sourced by the corresponding shell script (`MNAAS_msisdn_non_jlr_rate_per_country.sh`) which executes a Python ETL that calculates non‑JLR MSISDN billing rates per country, writes results to Hive, updates a process‑status file, and creates a dated log file.

# Key Components
- **MNAAS_Daily_msisdn_non_jlr_rate_per_country_ProcessStatusFileName** – absolute path of the process‑status file.  
- **MNAAS_Daily_msisdn_non_jlr_rate_per_country_logpath** – log file location with date suffix.  
- **MNAAS_Daily_msisdn_non_jlr_rate_per_country_Pyfile** – full path to the Python implementation (`MNAAS_msisdn_non_jlr_rate_per_country.py`).  
- **MNAAS_Daily_msisdn_non_jlr_rate_per_country_Script** – name of the wrapper shell script (`MNAAS_msisdn_non_jlr_rate_per_country.sh`).  
- **MNAAS_Daily_msisdn_non_jlr_rate_per_country_Refresh** – Hive `REFRESH` command for the target table.  
- **MNAAS_Daily_msisdn_non_jlr_rate_per_country_tblname** – Hive table name (`msisdn_tcl_rate_customer`).  
- **MNAAS_retention_period_non_jlr_adhoc_daily** – data retention period (days) for ad‑hoc cleanup.  
- **setparameter** – optional debugging flag (`set -x`).  

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| 1 | Staged source files (non‑JLR MSISDN usage) | Python script reads, aggregates, computes rates | Hive table `msisdn_tcl_rate_customer` (INSERT/OVERWRITE) | Writes to process‑status file, appends to dated log |
| 2 | Hive table after load | `REFRESH` command executed | Updated metadata cache | None |
| 3 | Retention policy job (outside scope) | Uses `MNAAS_retention_period_non_jlr_adhoc_daily` to purge old partitions | Deleted old partitions | None |

External services: Hadoop/Hive cluster, HDFS for logs and status file.

# Integrations
- **MNAAS_CommonProperties.properties** – provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`).  
- **MNAAS_msisdn_non_jlr_rate_per_country.sh** – sources this file, uses defined variables to invoke the Python script and manage status/logging.  
- **Hive Metastore** – target for `REFRESH` and table writes.  
- **Cron scheduler** – triggers the wrapper script on a daily basis.  

# Operational Risks
- **Missing/incorrect base paths** – leads to failure to write status or logs. *Mitigation*: Validate paths at script start; abort with clear error.  
- **Python script runtime errors** – may leave stale status file. *Mitigation*: Capture exit code, update status file to “FAILED”, and send alert.  
- **Retention period mismatch** – accidental deletion of recent data. *Mitigation*: Enforce a minimum retention threshold and log the purge action.  
- **Debug flag left enabled** – excessive log volume. *Mitigation*: Default `setparameter` to empty; require explicit uncomment for debugging.  

# Usage
```bash
# Source the properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_msisdn_non_jlr_rate_per_country.properties

# Run the job (normally invoked by the wrapper script)
bash $MNAAS_Daily_msisdn_non_jlr_rate_per_country_Script
```
To enable step‑by‑step debugging, uncomment the `setparameter='set -x'` line before sourcing.

# Configuration
- **Environment variables** supplied by `MNAAS_CommonProperties.properties`:
  - `MNAASConfPath` – directory for process‑status files.  
  - `MNAASLocalLogPath` – base directory for log files.  
- **External config files**:
  - `MNAAS_CommonProperties.properties` (shared constants).  
  - `MNAAS_msisdn_non_jlr_rate_per_country.sh` (wrapper script).  
  - `MNAAS_msisdn_non_jlr_rate_per_country.py` (Python ETL).  

# Improvements
1. **Parameter validation block** – add a function that checks existence and write permissions of all derived paths before job execution.  
2. **Centralised status handling** – abstract status file updates into a reusable library to ensure consistent SUCCESS/FAILURE semantics across all MNAAS jobs.