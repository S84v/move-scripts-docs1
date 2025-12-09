# Summary
Defines Bash‑sourced runtime constants for the **Customer‑Level Table Statistics Aggregation** step of the MNAAS daily‑processing pipeline. Supplies process‑status tracking, logging, Hive/Impala table identifiers, Java class names, HDFS staging paths, and data‑file locations used by the aggregation and loading jobs that compute per‑customer statistics.

# Key Components
- **MNAAS_table_statistics_customer_aggr_ProcessStatusFileName** – path to the status‑file that records job completion/failure.  
- **MNAAS_table_statistics_customer_aggr_LogPath** – daily log file location for the aggregation script.  
- **Dname_Load_table_statistics_customer_aggr_tbl** – identifier for the Hive/Impala load‑table step.  
- **Dname_MNAAS_table_statistics_customer_aggr** – logical name of the aggregation job.  
- **Dname_MNAAS_table_statistics_customer_aggr_tbl** – target Hive/Impala table name for aggregated data.  
- **Dname_table_statistics_customer_aggr_load** – name of the loading sub‑process.  
- **MNAAS_table_statistics_customer_aggr_classname** – fully‑qualified Java class implementing the aggregation logic.  
- **MNAAS_table_statistics_customer_aggregation_data_filename** – local file that holds intermediate aggregation data before HDFS upload.  
- **MNAAS_table_statistics_customer_aggr_PathName** – HDFS staging directory for the aggregation output.  
- **MNAAS_table_statistics_customer_loading_classname** – Java class responsible for loading the staged data into the target Hive/Impala table.  

# Data Flow
1. **Input**: Oracle source tables (not defined here) accessed by `MNAAS_table_statistics_customer_aggr_classname`.  
2. **Processing**: Java aggregation job writes intermediate results to `$MNAAS_table_statistics_customer_aggregation_data_filename`.  
3. **Staging**: Files are copied to `$MNAAS_table_statistics_customer_aggr_PathName` on HDFS.  
4. **Loading**: `MNAAS_table_statistics_customer_loading_classname` reads staged files and inserts into `$Dname_MNAAS_table_statistics_customer_aggr_tbl`.  
5. **Outputs**:  
   - Hive/Impala table `$Dname_MNAAS_table_statistics_customer_aggr_tbl`.  
   - Log file `$MNAAS_table_statistics_customer_aggr_LogPath`.  
   - Status file `$MNAAS_table_statistics_customer_aggr_ProcessStatusFileName`.  
6. **Side Effects**: Updates process‑status file, creates/overwrites HDFS staging directory, writes logs.

# Integrations
- **MNAAS_CommonProperties.properties** – imported for shared variables (`MNAASConfPath`, `MNAASLocalLogPath`, `MNAAS_Daily_Aggregation_PathName`).  
- **MNAAS_table_statistics_customer_aggr.sh** (not shown) – sources this file and invokes the Java classes.  
- **MNAAS_Table_Space_Monitoring**, **MNAAS_Sqoop_sim_mlns_mapping**, etc., – sibling pipeline steps that may depend on the same common properties and share HDFS base paths.  

# Operational Risks
- **Missing Common Properties**: Failure to source `MNAAS_CommonProperties.properties` results in undefined paths. *Mitigation*: Verify file existence before sourcing; fail fast with clear error.  
- **Stale Status File**: Re‑using an old status file may cause false‑positive success reports. *Mitigation*: Script must truncate or delete status file at start.  
- **HDFS Path Collisions**: Concurrent runs could overwrite `$MNAAS_table_statistics_customer_aggr_PathName`. *Mitigation*: Include run‑time identifier (e.g., timestamp) in path.  
- **Java Class Version Mismatch**: Incompatible JARs cause job failure. *Mitigation*: Pin JAR versions and validate at deployment.  

# Usage
```bash
# Source common properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Source this configuration
. move-mediation-scripts/config/MNAAS_table_statistics_customer_aggr.properties

# Execute aggregation (example)
hadoop jar /path/to/aggregation.jar \
    ${MNAAS_table_statistics_customer_aggr_classname} \
    --output ${MNAAS_table_statistics_customer_aggr_PathName} \
    > ${MNAAS_table_statistics_customer_aggr_LogPath} 2>&1

# Update status file on success/failure
if [ $? -eq 0 ]; then
    echo "SUCCESS" > ${MNAAS_table_statistics_customer_aggr_ProcessStatusFileName}
else
    echo "FAILURE" > ${MNAAS_table_statistics_customer_aggr_ProcessStatusFileName}
fi
```

# Configuration
- **Environment Variables / Files**:  
  - `MNAAS_CommonProperties.properties` (provides `MNAASConfPath`, `MNAASLocalLogPath`, `MNAAS_Daily_Aggregation_PathName`).  
  - `MNAAS_table_statistics_customer_aggr_ProcessStatusFile` (defined in common properties).  
- **Referenced Paths**:  
  - `$MNAASConfPath` – directory for configuration and status files.  
  - `$MNAASLocalLogPath` – base directory for log files.  
  - `$MNAAS_Daily_Aggregation_PathName` – HDFS root for daily aggregation outputs.  

# Improvements
1. **Parameterize Date in Log Path** – currently appends `$(date +_%F)` at source time; move date handling to the invoking script to avoid log file name changes on re‑source.  
2. **Add Validation Block** – include a function that checks existence of all required variables and directories, exiting with a non‑zero code if any are missing.