# Summary
Defines runtime constants for the **MNAAS_MSISDN_Level_Daily_Usage_Aggr** batch job. The properties are sourced by `MNAAS_MSISDN_Level_Daily_Usage_Aggr.sh` to control process‑status tracking, logging, Hive table targets, and control‑file location for daily MSISDN usage aggregation.

# Key Components
- **MNAAS_MSISDN_level_daily_usage_aggr_ProcessStatusFileName** – absolute path of the process‑status file.  
- **MNAAS_MSISDN_Level_Daily_Usage_Aggr_ScriptName** – name of the executable shell script.  
- **MNAAS_MSISDN_level_daily_usage_aggrLogPath** – log file path with date suffix.  
- **Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl** – Hive table for aggregated usage.  
- **Dname_MNAAS_retention_msisdn_level_daily_usage_aggr_tbl** – Hive table for retention‑aged data.  
- **MNAAS_MSISDN_level_daily_usage_aggrCntrlFileName** – path to the control‑file properties used by the script.  

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| 1 | Common properties (`MNAAS_CommonProperties.properties`) | Variable substitution | Fully qualified paths & names | None |
| 2 | Control file (`MNAAS_MSISDN_level_daily_usage_aggrCntrlFile.properties`) | Script reads job parameters (e.g., source directories) | – | May affect HDFS reads |
| 3 | Staged MSISDN usage files (HDFS) | Aggregation logic in the shell‑script / invoked Hive/Impala queries | Populated `Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl` | Writes to Hive, updates process‑status file |
| 4 | Completed aggregation | Retention logic (optional) | Populated `Dname_MNAAS_retention_msisdn_level_daily_usage_aggr_tbl` | May delete/archieve old partitions |
| 5 | Execution result | Log generation | `MNAAS_MSISDN_Level_Daily_Usage_Aggr.log_YYYY-MM-DD` | Log rotation, alerting |

# Integrations
- **MNAAS_CommonProperties.properties** – provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`, etc.).  
- **MNAAS_MSISDN_Level_Daily_Usage_Aggr.sh** – consumes all variables defined herein.  
- **Hive / Impala** – target tables for aggregated usage and retention data.  
- **HDFS** – source staging area for raw MSISDN usage files.  
- **Process‑status monitoring** – external orchestration (e.g., Oozie/Airflow) polls the status file.

# Operational Risks
- **Missing or incorrect base paths** → job fails to write status/log files. *Mitigation*: validate existence of `$MNAASConfPath` and `$MNAASLocalLogPath` at script start.  
- **Date suffix generation (`date +_%F`)** may produce unexpected format on non‑GNU `date`. *Mitigation*: enforce GNU coreutils or fallback to `$(date +"_%Y-%m-%d")`.  
- **Hive table schema drift** → aggregation queries break. *Mitigation*: version control table DDL and run schema validation step.  
- **Concurrent executions** → race condition on status file. *Mitigation*: implement file locking or use a unique run identifier.  

# Usage
```bash
# Source the property file
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_MSISDN_Level_Daily_Usage_Aggr.properties

# Execute the batch job
bash $MNAAS_MSISDN_Level_Daily_Usage_Aggr_ScriptName
```
For debugging, export `DEBUG=1` before sourcing to enable verbose logging inside the shell script.

# Configuration
- **Environment variables** (provided by `MNAAS_CommonProperties.properties`):  
  - `MNAASConfPath` – directory for configuration files.  
  - `MNAASLocalLogPath` – base directory for log files.  
- **Referenced config files**:  
  - `MNAAS_CommonProperties.properties` (global defaults).  
  - `MNAAS_MSISDN_level_daily_usage_aggrCntrlFile.properties` (job‑specific control parameters).  

# Improvements
1. **Externalize date format** – add a property `MNAAS_LogDateFormat=%Y-%m-%d` and construct `MNAAS_MSISDN_level_daily_usage_aggrLogPath` using it to avoid hard‑coded `date` command.  
2. **Add validation block** – script should verify that all required variables (paths, table names) are non‑empty and that target Hive tables exist before processing.