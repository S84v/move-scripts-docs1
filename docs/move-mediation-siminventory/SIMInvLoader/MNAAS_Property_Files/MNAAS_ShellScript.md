# Summary
`MNAAS_ShellScript.properties` is a key‑value configuration file used by the Move mediation **SIMInvLoader** job. It supplies Oracle and Impala connection parameters, runtime constants (batch sizes, directory paths, email settings) and environment‑specific toggles. The properties are loaded by Java/Scala DAO and shell scripts at job start to drive SIM inventory extraction, staging, logging and notification in production.

# Key Components
- **Oracle connection properties** (`ora_serverNameMOVE`, `ora_portNumberMOVE`, `ora_serviceNameMOVE`, `ora_usernameMOVE`, `ora_passwordMOVE`) – used by DAO classes to obtain JDBC connections to the Move database (test instance `comtst` by default).  
- **Impala connection properties** (`IMPALAD_HOST`, `IMPALAD_JDBC_PORT`) – consumed by Hive/Impala utilities for refresh statements.  
- **Batch/threshold constants** (`siminventory_backlog`, `siminventory_preactive_count`, `siminventory_partdate`) – control incremental load size and pre‑active record limits.  
- **File system paths** (`MNAASMainStagingDirDaily_*`, `Daily_SimInventory_BackupDir_Full`, `MNAASRejectedFilePathFull`) – define staging, backup and reject directories for input CSVs.  
- **Email settings** (`siminv_mail_to`, `siminv_mail_cc`, `siminv_mail_from`, `siminv_mail_host`) – used by the notification module to send job status e‑mails.  
- **Log path** (`MNAAS_daily_reports_LogPath`) – directory where the job writes its cron‑log files.

# Data Flow
1. **Input**: Raw SIM inventory files placed in `MNAASMainStagingDirDaily` (or versioned dirs).  
2. **Processing**:  
   - Java/Scala DAO reads Oracle connection props → opens JDBC session → executes SQL statements from `MOVEDAO.properties`.  
   - Impala host/port used by Hive scripts to refresh materialized views after load.  
   - Batch constants limit rows fetched/processed per run.  
3. **Output**:  
   - Populated target tables in Oracle `MNAAS` schema.  
   - Processed files moved to backup dir (`Daily_SimInventory_BackupDir_Full`).  
   - Rejected records written to `MNAASRejectedFilePathFull`.  
   - Log file written to `MNAAS_daily_reports_LogPath`.  
   - Status e‑mail sent via `siminv_mail_host`.  
4. **Side Effects**: Directory creation/removal, file moves, DB commits, email dispatch.

# Integrations
- **`MOVEDAO.properties`** – DAO loads SQL strings using the DB credentials defined here.  
- **Shell script `SIMInvLoader.sh`** – sources this properties file before invoking Java classes and Hive scripts.  
- **Log4j configuration (`log4j.properties`)** – consumes `MNAAS_daily_reports_LogPath` for file appender.  
- **External services**: Oracle DB (`comtst`), Impala/ Hive cluster, SMTP server (`mxrelay.vsnl.co.in`).  

# Operational Risks
- **Plain‑text credentials** – risk of credential leakage; mitigate by encrypting or using a secret manager.  
- **Hard‑coded test DB host** – accidental execution against non‑prod environment; enforce environment flag or CI gate.  
- **Absolute Windows paths** – job fails on Linux nodes; ensure path compatibility or use OS‑agnostic variables.  
- **Static batch sizes** – may cause OOM if data volume spikes; monitor and adjust thresholds dynamically.  

# Usage
```bash
# From the SIMInvLoader bin directory
source ../MNAAS_Property_Files/MNAAS_ShellScript.properties
./SIMInvLoader.sh   # job reads the sourced properties automatically
```
To debug, export `DEBUG=true` before sourcing; Java classes will log full stack traces to the configured log file.

# Configuration
- **Environment variables**: None required; all values are in this file.  
- **Referenced config files**: `MOVEDAO.properties`, `log4j.properties`.  
- **Override mechanism**: Comment/uncomment the appropriate Oracle block (Prod/Test/Dev) to switch environments.  

# Improvements
1. Replace clear‑text passwords with a vault lookup (e.g., HashiCorp Vault or AWS Secrets Manager).  
2. Externalize OS‑specific paths to a separate platform‑specific properties file and load conditionally based on `os.name`.