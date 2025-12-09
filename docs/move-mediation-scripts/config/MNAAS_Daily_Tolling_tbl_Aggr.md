# Summary
Defines runtime constants for the **MNAAS_Daily_Tolling_tbl_Aggr** batch job. The properties are sourced by `MNAAS_Daily_Tolling_tbl_Aggr.sh` to locate the process‑status file, construct the daily log filename, identify the script name, and reference the Hive insert procedure used in the daily tolling aggregation pipeline of the Move production environment.

# Key Components
- `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` – imports shared base paths and Hadoop settings.  
- `MNAAS_Daily_Tolling_Aggr_ProcessStatusFileName` – full HDFS path to the process‑status file.  
- `MNAAS_DailyTollingAggrLogPath` – local log file path, includes current date (`date +_%F`).  
- `MNAASDailyTollingAggrScriptName` – canonical script filename.  
- `Dname_MNAAS_Insert_Daily_Aggr_Tolling_tbl` – Hive stored‑procedure / table identifier for the insert operation.

# Data Flow
| Stage | Input | Output | Side Effects |
|-------|-------|--------|--------------|
| Shell script start | Sourced properties file | Environment variables populated | None |
| Process‑status check | `$MNAAS_Daily_Tolling_Aggr_ProcessStatusFileName` (HDFS) | Boolean status | Reads HDFS file |
| Job execution | Hive procedure `$Dname_MNAAS_Insert_Daily_Aggr_Tolling_tbl` | Populated `tolling_tbl` partition | Writes to Hive, updates HDFS |
| Logging | `$MNAAS_DailyTollingAggrLogPath` | Log file with timestamp | Appends to local filesystem |
| Completion | N/A | Updated process‑status file | Writes status to HDFS |

# Integrations
- **MNAAS_CommonProperties.properties** – provides `$MNAASConfPath`, `$MNAASLocalLogPath`, Hadoop configuration, and other shared constants.  
- **MNAAS_Daily_Tolling_tbl_Aggr.sh** – consumes this properties file via `source`.  
- **Hive** – invoked through `$Dname_MNAAS_Insert_Daily_Aggr_Tolling_tbl`.  
- **HDFS** – process‑status file resides under `$MNAASConfPath`.  
- **Local filesystem** – log directory defined by `$MNAASLocalLogPath`.

# Operational Risks
- Missing or unreadable `MNAAS_CommonProperties.properties` → job fails at source time.  
- Incorrect `$MNAASConfPath` or `$MNAASLocalLogPath` values → status file or logs written to wrong location.  
- Date command failure or locale mismatch → log filename may not contain expected date, causing log rotation issues.  
- Hive procedure name mismatch → runtime Hive errors, incomplete aggregation.  

Mitigations: pre‑run validation script, enforce file permissions, centralize date formatting, add health‑check for Hive procedure existence.

# Usage
```bash
# From the job’s working directory
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_Tolling_tbl_Aggr.properties
bash MNAAS_Daily_Tolling_tbl_Aggr.sh   # script uses the exported variables
```
For debugging, export `DEBUG=1` before sourcing to enable verbose logging inside the shell script.

# Configuration
- **Environment variables** (populated by common properties):  
  - `MNAASConfPath` – base HDFS configuration directory.  
  - `MNAASLocalLogPath` – base local log directory.  
- **Referenced config file**: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`.  

# Improvements
1. Add a sanity‑check block at the end of the properties file to verify that required variables (`MNAASConfPath`, `MNAASLocalLogPath`) are non‑empty and that the common properties file exists.  
2. Externalize the date format (`%F`) into a configurable variable to allow locale‑specific log naming without editing the properties file.