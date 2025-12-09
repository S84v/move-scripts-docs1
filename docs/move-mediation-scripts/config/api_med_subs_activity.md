# Summary
`api_med_subs_activity.properties` defines environment‑specific parameters for the **API_MED_SUBS_ACTIVITY** ingestion pipeline. It supplies log locations, process‑status file paths, Hive/Impala connection details, database and table names, and the Java class names used for temporary and final table loading. The properties are sourced by the `api_med_subs_activity_loading.sh` script and the associated Java table‑loading classes to orchestrate nightly extraction, transformation, and loading (ETL) of subscriber activity data into Hive/Impala for downstream Move‑Mediation analytics.

# Key Components
- **MNAAS_api_med_subs_activity_LogPath** – Log file path with date suffix.  
- **api_med_subs_activity_ProcessStatusFileName** – HDFS/local file tracking job status.  
- **Dname_api_med_subs_activity** – Logical name for the process (used in monitoring).  
- **api_med_subs_activity_temp_loading_classname** – Fully‑qualified Java class for loading into the temporary staging table.  
- **api_med_subs_activity_loading_classname** – Fully‑qualified Java class for loading into the final production table.  
- **api_med_subs_activity_temp_tblname_refresh** – Hive `REFRESH` command template for the temp table.  
- **api_med_subs_activity_tblname_refresh** – Hive `REFRESH` command template for the final table.  
- **api_med_subs_activity_loading_ScriptName** – Shell script invoked to run the load.  
- **IMPALAD_HOST / IMPALAD_JDBC_PORT** – Impala service endpoint.  
- **HIVE_HOST / HIVE_JDBC_PORT** – HiveServer2 endpoint.  
- **dbname** – Hive database name (`mnaas`).  
- **api_med_subs_activity_temp_tblname** – Temporary staging Hive table.  
- **api_med_subs_activity_tblname** – Production Hive table.  
- **traffic_details_daily_tblname** – Reference table name used by downstream jobs (not directly in this script).  

# Data Flow
1. **Input**: Source data extracted from Oracle (or other upstream system) by Sqoop/Java loader classes.  
2. **Processing**:  
   - `api_med_subs_activity_temp_loading_classname` writes rows to `${dbname}.${api_med_subs_activity_temp_tblname}`.  
   - `api_med_subs_activity_loading_classname` moves data from the temp table to `${dbname}.${api_med_subs_activity_tblname}`.  
3. **Side Effects**:  
   - Writes operational logs to `MNAAS_api_med_subs_activity_LogPath`.  
   - Updates process‑status file (`api_med_subs_activity_ProcessStatusFileName`).  
   - Executes Hive `REFRESH` commands to invalidate metadata caches for both tables.  
4. **External Services**:  
   - HiveServer2 (`HIVE_HOST:HIVE_JDBC_PORT`).  
   - Impala daemon (`IMPALAD_HOST:IMPALAD_JDBC_PORT`).  
   - HDFS for status file and logs.  

# Integrations
- **Shell Wrapper**: `api_med_subs_activity_loading.sh` reads this properties file to set environment variables before invoking Java loaders.  
- **Java Loaders**: `com.tcl.mnass.tableloading.api_med_subs_activity_temp` and `com.tcl.mnass.tableloading.api_med_subs_activity` implement `DBWritable`/`Writable` to perform Sqoop‑style imports.  
- **Common Properties**: Sourced at the top (`MNAAS_CommonProperties.properties`) for shared paths, credentials, and Hadoop configuration.  
- **Downstream Jobs**: Tables populated here are consumed by nightly Move‑Mediation analytics (balance‑update, KYC reporting).  

# Operational Risks
- **Stale Credentials**: If common properties are updated without redeploying, authentication to Hive/Impala may fail. *Mitigation*: Centralize credential rotation and enforce reload on change.  
- **Metadata Inconsistency**: Missing `REFRESH` may cause stale query results. *Mitigation*: Ensure both refresh commands execute in a finally block.  
- **Log Disk Exhaustion**: Daily log files accumulate. *Mitigation*: Implement log rotation/compression.  
- **Process‑Status File Contention**: Concurrent runs may overwrite status file. *Mitigation*: Use unique run identifiers or lock file semantics.  

# Usage
```bash
# Export properties file location
export PROP_FILE=/path/to/api_med_subs_activity.properties

# Run the loading script (debug mode prints env)
bash api_med_subs_activity_loading.sh -p $PROP_FILE -d   # -d enables debug logging
```
To debug Java loaders, set `JAVA_TOOL_OPTIONS="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005"` before invoking the script.

# Configuration
- **File**: `move-mediation-scripts/config/api_med_subs_activity.properties`  
- **Included File**: `MNAAS_CommonProperties.properties` (provides `$MNAASLocalLogPath`, Hadoop classpath, etc.)  
- **Environment Variables** (populated by the script):  
  - `MNAAS_api_med_subs_activity_LogPath`  
  - `api_med_subs_activity_ProcessStatusFileName`  
  - `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`  
  - `HIVE_HOST`, `HIVE_JDBC_PORT`  
  - `dbname`, `api_med_subs_activity_temp_tblname`, `api_med_subs_activity_tblname`  

# Improvements
1. **Externalize Secrets** – Move Hive/Impala credentials to a secure vault (e.g., Hadoop Credential Provider) and reference via token instead of plain‑text properties.  
2. **Idempotent Refresh** – Wrap Hive `REFRESH` statements in a transaction‑safe wrapper that retries on transient failures and logs outcomes.  