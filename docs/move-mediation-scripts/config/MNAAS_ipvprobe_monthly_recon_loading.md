# Summary
Defines runtime constants for the **IPVProbe Monthly Reconciliation Loading** batch job. The properties are sourced by `MNAAS_ipvprobe_monthly_recon_loading.sh` and drive script identification, process‑status tracking, logging, intermediate file locations, HDFS load path, and the temporary Hive table used to load monthly IPVProbe reconciliation data in the Move‑Mediation production environment.

# Key Components
- **MNAAS_ipvprobe_Monthly_Recon_ScriptName** – name of the executing shell script.  
- **MNAAS_ipvprobe_Monthly_Recon_ProcessStatusFileName** – full HDFS path to the process‑status file for the monthly recon job.  
- **MNAAS_ipvprobe_Monthly_recon_loadingLogName** – local log file path (date‑suffixed) for job execution details.  
- **MNAASInterFilePath_Monthly_IPVProbe_Recon_Details** – HDFS directory for intermediate daily recon detail files aggregated into the monthly run.  
- **MNAAS_Monthly_IPVProbe_Recon_Interdates_File** – local file listing the date partitions to be processed for the monthly load.  
- **MNAAS_Monthly_Recon_load_ipvprobe_PathName** – HDFS target directory where the monthly recon raw files are staged for Hive ingestion.  
- **Dname_MNAAS_Load_Monthly_ipvprobe_recon_tbl_temp** – logical Hive table name used as a temporary staging table during the load.

# Data Flow
| Stage | Input | Transformation | Output |
|-------|-------|------------------|--------|
| 1. Script start | `MNAAS_ipvprobe_monthly_recon_loading.sh` reads this properties file | Loads constants into environment variables | Variables available to script |
| 2. Status check | Process‑status file (`*_ProcessStatusFile`) on HDFS | Determines if previous run succeeded/failed | Decision branch in script |
| 3. Log init | Local log path (`*.log$(date +_%F)`) | Opens/creates log file | Execution trace |
| 4. Intermediate aggregation | Daily recon detail files under `MNAASInterFilePath_Monthly_IPVProbe_Recon_Details` | Concatenates/filters per dates listed in `MNAAS_Monthly_IPVProbe_Recon_Interdates_File` | Monthly recon file in `MNAAS_Monthly_Recon_load_ipvprobe_PathName` |
| 5. Hive load | Monthly recon file in HDFS | `INSERT OVERWRITE` into temporary Hive table `Dname_MNAAS_Load_Monthly_ipvprobe_recon_tbl_temp` | Data available for downstream processing |
| 6. Completion | Updates process‑status file | Marks job as SUCCESS/FAIL | Status persisted for next run |

Side effects: writes to HDFS directories, updates Hive temporary table, creates local log file, modifies process‑status file.

External services: HDFS, Hive Metastore, local filesystem for logs and date‑list file.

# Integrations
- **Common Properties**: Sources `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` for shared paths (`MNAASConfPath`, `MNAASLocalLogPath`).  
- **Shell Script**: Consumed by `MNAAS_ipvprobe_monthly_recon_loading.sh`.  
- **Downstream Jobs**: Subsequent Hive/Impala jobs reference the temporary table `MNAAS_Insert_Monthly_ipvprobe_Recon_temp_tbl` for final reporting or archival.  
- **Monitoring**: Process‑status file is polled by the production orchestration layer (e.g., Oozie/Airflow) to trigger next steps.

# Operational Risks
- **Stale Process‑Status File** – may cause false success/failure detection. *Mitigation*: enforce file cleanup at job start/end.  
- **Date List Mismatch** – `IPVProbe_Inter_Table_dates_monthly.txt` may contain invalid or missing dates, leading to incomplete loads. *Mitigation*: validate file content before processing.  
- **HDFS Path Permissions** – insufficient write rights to intermediate or load directories cause job abort. *Mitigation*: audit ACLs and run as dedicated service account.  
- **Log File Growth** – daily log rotation not enforced; logs may consume local disk. *Mitigation*: implement log rotation or retention policy.

# Usage
```bash
# Load common properties and this file
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ipvprobe_monthly_recon_loading.properties

# Execute the batch job
bash $MNAAS_ipvprobe_Monthly_Recon_ScriptName
```
For debugging, export `DEBUG=1` before running to increase script verbosity.

# Configuration
- **Environment Variables** (provided by `MNAAS_CommonProperties.properties`):  
  - `MNAASConfPath` – base config directory.  
  - `MNAASLocalLogPath` – local log directory.  
- **Referenced Files**:  
  - `IPVProbe_Inter_Table_dates_monthly.txt` – list of date partitions.  
  - `MNAAS_ipvprobe_monthly_recon_loading_ProcessStatusFile` – status tracking file.  
- **HDFS Directories**:  
  - `/app/hadoop_users/MNAAS/MNAAS_Intermediatefiles/Monthly/IPVProbe_Daily_Recon`  
  - `/user/MNAAS/RawtablesLoad/Monthly/IPVProbe_Recon`

# Improvements
1. **Parameterize Log Rotation** – add a property for log retention and integrate with `logrotate` to prevent disk exhaustion.  
2. **Validate Date List Atomically** – implement a pre‑run checksum or schema validation of `IPVProbe_Inter_Table_dates_monthly.txt` to abort early on malformed input.