# Summary
Defines runtime constants for the **MNAAS_msisdn_jlr_rate_per_country** batch job. The job executes a Python script that calculates MSISDN‑JLR billing rates per country, writes results to a Hive table, records its execution status, and generates a dated log file.

# Key Components
- `MNAAS_Daily_msisdn_jlr_rate_per_country_ProcessStatusFileName` – absolute path for the process‑status file.  
- `MNAAS_Daily_msisdn_jlr_rate_per_country_logpath` – log file path with date suffix.  
- `MNAAS_Daily_msisdn_jlr_rate_per_country_Pyfile` – full path to the Python implementation (`MNAAS_msisdn_jlr_billing_rate.py`).  
- `MNAAS_Daily_msisdn_jlr_rate_per_country_Script` – name of the wrapper shell script (`MNAAS_msisdn_jlr_rate_per_country.sh`).  
- `MNAAS_Daily_msisdn_jlr_rate_per_country_Refresh` – Hive/Impala refresh command for the target table.  
- `MNAAS_Daily_msisdn_jlr_rate_per_country_tblname` – target Hive table (`msisdn_jlr_billing_rate`).  
- `MNAAS_retention_period_jlr_adhoc_daily` – data‑retention window (days).  
- `setparameter='set -x'` – optional Bash debugging flag (commented out to disable).  

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| 1 | Staged MSISDN‑JLR raw files (HDFS) | `MNAAS_msisdn_jlr_rate_per_country.sh` invokes the Python script | Aggregated billing‑rate records written to Hive table `msisdn_jlr_billing_rate` | Process‑status file updated; log file appended; Hive table refreshed |
| 2 | Hive table `msisdn_jlr_billing_rate` | Refresh command executed (`refresh mnaas.msisdn_jlr_billing_rate`) | Table metadata refreshed for downstream queries | None |
| 3 | Retention policy | Periodic cleanup (outside this script) uses `MNAAS_retention_period_jlr_adhoc_daily` | Old partitions older than 185 days are dropped | Data loss if mis‑configured |

# Integrations
- **Common Properties**: Sourced from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` (provides `$MNAASConfPath`, `$MNAASLocalLogPath`, etc.).  
- **Shell Wrapper**: `MNAAS_msisdn_jlr_rate_per_country.sh` reads the variables defined here to construct the execution command.  
- **Python Engine**: `MNAAS_msisdn_jlr_billing_rate.py` performs the core aggregation; relies on Spark/Hive libraries available on the cluster.  
- **Hive/Impala**: Refresh command targets the Hive metastore; downstream analytics consume the table.  
- **Monitoring**: Process‑status file is polled by the production scheduler (e.g., Oozie/Cron) to determine success/failure.

# Operational Risks
- **Incorrect path**: Mis‑typed `$MNAASConfPath` or `$MNAASLocalLogPath` leads to missing status/log files → job considered hung. *Mitigation*: Validate paths at script start; fail fast with clear error.  
- **Stale Python reference**: The property points to `MNAAS_msisdn_jlr_billing_rate.py`; if the script is updated without corresponding property change, logic mismatch may occur. *Mitigation*: Enforce version‑controlled deployment and CI check for property‑script alignment.  
- **Retention mis‑configuration**: Setting `MNAAS_retention_period_jlr_adhoc_daily` too low can purge needed data. *Mitigation*: Guard cleanup jobs with a minimum retention threshold.  
- **Debug flag exposure**: Leaving `set -x` enabled in production can flood logs with sensitive data. *Mitigation*: Keep the flag commented; enable only in a controlled debug session.

# Usage
```bash
# Load properties
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source /path/to/MNAAS_msisdn_jlr_rate_per_country.properties

# Execute wrapper script (normally invoked by scheduler)
bash $MNAAS_Daily_msisdn_jlr_rate_per_country_Script

# To enable Bash debugging for a single run
set -x   # or uncomment setparameter line in the properties file
```

# Configuration
- **Environment Variables** (provided by common properties):  
  - `MNAASConfPath` – base directory for process‑status files.  
  - `MNAASLocalLogPath` – base directory for log files.  
- **Referenced Config Files**:  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` (global defaults).  
- **Hard‑coded Paths**:  
  - Python script: `/app/hadoop_users/MNAAS/MNAAS_CronFiles/MNAAS_msisdn_jlr_billing_rate.py`  
  - Shell script: `MNAAS_msisdn_jlr_rate_per_country.sh` (expected in the same directory as the properties file or in `$PATH`).  

# Improvements
1. **Parameterize Retention**: Move `MNAAS_retention_period_jlr_adhoc_daily` to a dedicated retention config file to allow independent tuning without modifying the job properties.  
2. **Add Validation Block**: Include a Bash function that verifies existence and write permissions of all referenced files/directories at load time; abort with descriptive error if any check fails.