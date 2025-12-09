# Summary
Defines runtime constants for the **IPVProbe Hive table load** batch job (`MNAAS_ipvprobe_tbl_Load.sh`). Supplies process‑status file path, log file name, script identifier, intermediate file locations, Hive table names, and HDFS paths used to ingest daily raw IPVProbe data into temporary and final Hive tables in the Move‑Mediation production environment.

# Key Components
- `MNAAS_Daily_ipvprobe_Load_Raw_ProcessStatusFileName` – full path to the process‑status file for the load job.  
- `MNAAS_Daily_ipvprobe_Load_Raw_LogPath` – log file path with date suffix.  
- `MNAASDailyIPVProbeLoadScriptName` – name of the executing shell script.  
- `MNASS_IPVProbe_Inter_removedups_withdups_filepath` – directory for intermediate files that retain duplicate records.  
- `MNASS_IPVProbe_Inter_removedups_withoutdups_filepath` – directory for intermediate files after duplicate removal.  
- `Dname_MNAAS_Insert_Daily_Reject_ipvprobe_tbl` – Hive table for rejected rows.  
- `Dname_MNAAS_Insert_Daily_ipvprobe_tbl` – Hive table for successfully inserted rows.  
- `Dname_MNAAS_Load_Daily_ipvprobe_tbl_temp` – temporary Hive staging table.  
- `MNAAS_Daily_Rawtablesload_ipvprobe_PathName` – HDFS root for raw IPVProbe data.  
- `MNAASInterFilePath_Daily_IPVProbe_Details` – local intermediate directory for daily detail files.  
- `MNAASInterFilePath_Dups_Check` – path to the master file used for duplicate‑check logic.  
- `MNAAS_IPVProbe_Error_FileName` – file that aggregates load‑time errors.  
- `MNNAS_IPVProbe_OnlyHeaderFile` – file containing only the header line for validation.

# Data Flow
1. **Input** – Raw IPVProbe files landed in `$MNAAS_Daily_Rawtablesload_ipvprobe_PathName` (HDFS).  
2. **Processing** – Shell script reads the above constants, copies raw files to intermediate directories, performs duplicate removal, validates headers, and writes error records to `$MNAAS_IPVProbe_Error_FileName`.  
3. **Hive Load** – Data loaded into `$Dname_MNAAS_Load_Daily_ipvprobe_tbl_temp`; successful rows moved to `$Dname_MNAAS_Insert_Daily_ipvprobe_tbl`, rejected rows to `$Dname_MNAAS_Insert_Daily_Reject_ipvprobe_tbl`.  
4. **Output** – Process‑status file updated, log file written, and optional error/validation files persisted.  
5. **Side Effects** – Creation/modification of HDFS directories, Hive tables, and local filesystem artifacts.

# Integrations
- **Common Properties** – Sources `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` for base paths (`$MNAASConfPath`, `$MNAASLocalLogPath`, etc.).  
- **Shell Script** – Consumed by `MNAAS_ipvprobe_tbl_Load.sh`.  
- **Hive** – Uses Hive CLI/Beeline to execute `INSERT` statements against the tables defined above.  
- **HDFS** – Reads raw files from `$MNAAS_Daily_Rawtablesload_ipvprobe_PathName`.  
- **Logging/Monitoring** – Process‑status file and dated log are read by external monitoring dashboards.

# Operational Risks
- **Path Misconfiguration** – Incorrect base path leads to file not found errors; mitigate with validation of `$MNAASConfPath` at script start.  
- **Duplicate Removal Logic Failure** – May cause data loss or duplicate rows; add checksum verification after dedup step.  
- **Hive Table Schema Drift** – Table definitions may diverge from source file format; enforce schema versioning and automated DDL checks.  
- **Log File Overwrite** – Date suffix prevents overwrite, but clock skew could create duplicate logs; synchronize system time via NTP.  
- **Process‑Status Staleness** – Stale status file may mask failures; implement timeout and alerting on status age.

# Usage
```bash
# Source common properties
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Load the IPVProbe constants
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ipvprobe_tbl_Load.properties

# Execute the load script (debug mode)
bash -x $MNAASDailyIPVProbeLoadScriptName
```
To debug, inspect `$MNAAS_Daily_ipvprobe_Load_Raw_LogPath` and the process‑status file.

# Configuration
- **Environment Variables** – `$MNAASConfPath`, `$MNAASLocalLogPath`, `$MNAASInterFilePath_Daily`, `$MNAAS_Daily_RAWtable_PathName` defined in `MNAAS_CommonProperties.properties`.  
- **Referenced Config Files** – `MNAAS_CommonProperties.properties` (base paths, Hadoop/Hive connection settings).  

# Improvements
1. Convert flat variable definitions to a structured properties file (key=value) and parse with a dedicated config loader to reduce shell parsing errors.  
2. Add checksum generation for each intermediate file and store in a manifest to enable automated integrity verification before Hive load.