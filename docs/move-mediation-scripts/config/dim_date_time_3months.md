# Summary
Defines environment‑specific parameters for the **dim_date_time_3months** ingestion step of the Move‑Mediation pipeline. Supplies script name, process‑status file location, log path, Hive table identifier, Java loader class, and process name. Used by the nightly Hadoop/Hive job to populate the `dim_date_time_3months` dimension table for the last three months of data.

# Key Components
- **MNAASdim_date_time_3monthsScriptName** – name of the shell wrapper (`dim_date_time_3months.sh`) that orchestrates the load.  
- **MNAAS_dim_date_time_3months_ProcessStatusFileName** – path to the process‑status file tracking job success/failure.  
- **MNAAS_dim_date_time_3months_LogPath** – absolute log file path (date‑stamped) for execution output.  
- **Dname_MNAAS_dim_date_time_3months_tbl** – Hive table identifier used in refresh statements.  
- **dim_date_time_3months** – fully‑qualified Java class (`com.tcl.mnass.tableloading.dim_date_time_3months`) that implements the Hive load logic.  
- **processname_dim_date_time_3months** – logical process name referenced by monitoring/alerting frameworks.

# Data Flow
1. **Input**: Source data files (e.g., CSV/Parquet) located in HDFS under a predefined staging directory (not defined here but referenced by the Java loader).  
2. **Processing**: `dim_date_time_3months.sh` invokes the Java class `com.tcl.mnass.tableloading.dim_date_time_3months`, which reads source files, transforms records, and writes to Hive.  
3. **Output**: Populated Hive table `MNAAS_dim_date_time_3months_tbl`.  
4. **Side Effects**: Updates `MNAAS_dim_date_time_3months_ProcessStatusFileName` with SUCCESS/FAIL status; writes execution details to `MNAAS_dim_date_time_3months_LogPath`.  
5. **External Services**: Hadoop Distributed File System (HDFS), Hive Metastore, Java runtime, optional monitoring system that polls the process‑status file.

# Integrations
- **Common Properties**: Sources `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` for shared variables (`MNAASConfPath`, `MNAASLocalLogPath`, etc.).  
- **Shell Wrapper**: `dim_date_time_3months.sh` is invoked by the nightly orchestration script (e.g., `MoveMediationNightly.sh`).  
- **Java Loader**: Integrated with the generic table‑loading framework used across Move‑Mediation dimensions.  
- **Monitoring**: Process name `processname_dim_date_time_3months` is consumed by the production health‑check dashboard.

# Operational Risks
- **Missing Common Properties**: Failure to source `MNAAS_CommonProperties.properties` results in undefined paths → job abort. *Mitigation*: Validate file existence at script start.  
- **Log Path Collisions**: Log file name includes date only; concurrent runs on the same day could overwrite logs. *Mitigation*: Append timestamp or PID.  
- **Process‑Status Stale**: If the status file is not cleaned between runs, downstream checks may read outdated results. *Mitigation*: Truncate or delete file at job start.  
- **Java Class Regression**: Changes to `com.tcl.mnass.tableloading.dim_date_time_3months` may break schema compatibility. *Mitigation*: Unit‑test against a fixed schema and enforce versioned JAR deployment.

# Usage
```bash
# Load common properties
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Execute the dimension load
bash $MNAASConfPath/dim_date_time_3months.sh

# Debug: view log
tail -f $MNAASLocalLogPath/dim_date_time_3months.log_$(date +%F)

# Verify status
cat $MNAASConfPath/dim_date_time_3months_ProcessStatusFile
```

# Configuration
- **Environment Variables** (provided by `MNAAS_CommonProperties.properties`):  
  - `MNAASConfPath` – directory containing process‑status files.  
  - `MNAASLocalLogPath` – base directory for log files.  
- **Referenced Config Files**:  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` (global defaults).  
  - `dim_date_time_3months.sh` (shell wrapper script).  

# Improvements
1. **Add Timestamp to Log Filename** – modify `MNAAS_dim_date_time_3months_LogPath` to include `%H%M%S` or `$PID` to avoid same‑day overwrites.  
2. **Validate Required Paths at Startup** – implement a pre‑flight check that aborts with a clear error if `MNAASConfPath` or `MNAASLocalLogPath` are missing or unwritable.