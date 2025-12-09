# Summary
Defines runtime constants for the **IPVProbe Daily Reconciliation Loading** batch job. The properties are sourced by `MNAAS_ipvprobe_daily_recon_loading.sh` and provide script name, process‑status file, log file (date‑suffixed), intermediate file locations, HDFS load path, and the temporary Hive table used to load daily IPVProbe reconciliation data in the Move‑Mediation production environment.

# Key Components
- **MNAAS_ipvprobe_Daily_Recon_ScriptName** – name of the executing shell script.  
- **MNAAS_ipvprobe_Daily_Recon_ProcessStatusFileName** – absolute path to the job’s process‑status file.  
- **MNAAS_ipvprobe_daily_recon_loadingLogName** – absolute path to the log file, includes current date (`%F`).  
- **MNAASInterFilePath_Daily_IPVProbe_Recon_Details** – HDFS directory for intermediate recon detail files.  
- **MNAAS_Daily_IPVProbe_Recon_Interdates_File** – local file listing partition dates for the intermediate table.  
- **MNAAS_Daily_Recon_load_ipvprobe_PathName** – HDFS target directory for the final load.  
- **Dname_MNAAS_Load_Daily_ipvprobe_recon_tbl_temp** – logical name of the temporary Hive table used during load.

# Data Flow
1. **Input** – Raw IPVProbe daily files placed in `${MNAASInterFilePath_Daily_IPVProbe_Recon_Details}` (HDFS).  
2. **Processing** – Shell script reads the above constants, creates/updates the process‑status file, writes operational logs to `${MNAAS_ipvprobe_daily_recon_loadingLogName}`.  
3. **Transformation** – Data staged in the intermediate HDFS path is loaded into Hive temporary table `${Dname_MNAAS_Load_Daily_ipvprobe_recon_tbl_temp}`.  
4. **Output** – Populated Hive table is persisted to `${MNAAS_Daily_Recon_load_ipvprobe_PathName}` (HDFS) and made available to downstream analytics.  
5. **Side Effects** – Updates process‑status file, generates log file, may trigger Hive metastore commits.  

# Integrations
- **MNAAS_CommonProperties.properties** – provides base variables (`MNAASConfPath`, `MNAASLocalLogPath`, `MNAASInterFilePath_Daily`).  
- **MNAAS_ipvprobe_daily_recon_loading.sh** – consumes all properties defined herein.  
- **Hive Metastore** – temporary table creation and insert operations.  
- **HDFS** – source, intermediate, and target directories.  
- Potential downstream batch jobs that consume the loaded Hive table.

# Operational Risks
- **Missing/incorrect base variables** (`MNAASConfPath`, etc.) → job fails to locate files. *Mitigation*: validate existence of all derived paths at script start.  
- **Date‑suffix log collision** if script runs multiple times per day. *Mitigation*: enforce single execution lock or include timestamp.  
- **Process‑status file stale** after abnormal termination → subsequent runs may skip processing. *Mitigation*: implement cleanup/recovery logic.  
- **Hive temporary table name conflict** across concurrent runs. *Mitigation*: append unique run identifier to temp table name.  

# Usage
```bash
# Source common properties and job‑specific properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ipvprobe_daily_recon_loading.properties

# Execute the batch job
bash $MNAAS_ipvprobe_Daily_Recon_ScriptName
```
*Debug*: add `set -x` in the shell script or `echo` each variable after sourcing to verify paths.

# Configuration
- **External config file**: `MNAAS_CommonProperties.properties` (defines `MNAASConfPath`, `MNAASLocalLogPath`, `MNAASInterFilePath_Daily`).  
- **Environment variables**: None required beyond those defined in the common properties file.  
- **Derived variables**: All constants in this file are derived from the common base paths.

# Improvements
1. **Parameterize temporary Hive table** with a run‑specific UUID to avoid name collisions in parallel executions.  
2. **Add pre‑execution validation** routine that checks existence and write permissions of all referenced HDFS and local paths; abort with clear error codes if any check fails.