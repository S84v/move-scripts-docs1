# Summary
Defines Bash‑sourced runtime constants for the **Table Statistics Call‑Date Aggregation** step of the MNAAS daily‑processing pipeline. Provides process‑status tracking, logging, Hive/Impala table identifiers, Java class names, and HDFS staging paths used by the aggregation and loading jobs that compute per‑call‑date statistics.

# Key Components
- `MNAAS_table_statistics_calldate_aggr_ProcessStatusFileName` – file tracking job status.  
- `MNAAS_table_statistics_calldate_aggr_LogPath` – daily log file path (includes execution date).  
- `Dname_Load_table_statistics_calldate_aggr_tbl` – Hive staging table name for the load step.  
- `Dname_MNAAS_table_statistics_calldate_aggr` / `Dname_MNAAS_table_statistics_calldate_aggr_tbl` – logical names for the aggregation Hive table and its physical counterpart.  
- `Dname_table_statistics_calldate_aggr_load` – identifier for the load operation.  
- `MNAAS_table_statistics_calldate_aggr_classname` – fully‑qualified Java class implementing the aggregation logic.  
- `MNAAS_table_statistics_calldate_aggregation_data_filename` – HDFS file that holds intermediate aggregation data.  
- `MNAAS_table_statistics_calldate_aggr_PathName` – HDFS directory for aggregation output.  
- `MNAAS_table_statistics_calldate_loading_classname` – Java class responsible for loading aggregated data into Hive.  

# Data Flow
1. **Input**: Oracle/Hive source tables consumed by `com.tcl.mnass.aggregation.table_statistics_calldate_aggregation`.  
2. **Processing**: Java aggregation class writes intermediate results to `$MNAAS_table_statistics_calldate_aggregation_data_filename`.  
3. **Staging**: Files are placed under `$MNAAS_table_statistics_calldate_aggr_PathName`.  
4. **Load**: `com.tcl.mnass.tableloading.table_statistics_calldate_loading` reads the staged files and inserts into Hive table `$Dname_MNAAS_table_statistics_calldate_aggr_tbl`.  
5. **Side Effects**: Updates `$MNAAS_table_statistics_calldate_aggr_ProcessStatusFileName`, writes logs to `$MNAAS_table_statistics_calldate_aggr_LogPath`.  

# Integrations
- **MNAAS_CommonProperties.properties** – imported for base paths (`MNAASConfPath`, `MNAASLocalLogPath`, `MNAAS_Daily_Aggregation_PathName`).  
- **MNAAS_table_statistics_calldate_aggr.sh** (or equivalent driver script) – sources this file and invokes the Java classes via Sqoop/Hadoop.  
- **Hive/Impala** – target tables referenced by the `Dname_*` variables.  
- **HDFS** – staging directories and data files referenced by the `*_PathName` and `*_data_filename` variables.  

# Operational Risks
- **Missing common properties** → job fails to resolve base paths. *Mitigation*: validate inclusion of `MNAAS_CommonProperties.properties` at script start.  
- **Incorrect date suffix in log path** → log rotation may overwrite or create malformed filenames. *Mitigation*: enforce `date +_%F` format and test on edge dates (e.g., leap year).  
- **Classpath misconfiguration** → Java aggregation/loading classes not found. *Mitigation*: centralize JAR version in a shared lib directory and verify at deployment.  
- **HDFS permission errors** → inability to write staging files. *Mitigation*: pre‑run a permission check on `$MNAAS_table_statistics_calldate_aggr_PathName`.  

# Usage
```bash
# Source configuration
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/config/MNAAS_table_statistics_calldate_aggr.properties

# Run driver script (example)
bash /app/hadoop_users/MNAAS/MNAAS_Scripts/MNAAS_table_statistics_calldate_aggr.sh
```
For debugging, echo key variables:
```bash
echo "Log: $MNAAS_table_statistics_calldate_aggr_LogPath"
echo "Agg class: $MNAAS_table_statistics_calldate_aggr_classname"
```

# Configuration
- **External config**: `MNAAS_CommonProperties.properties` (provides `MNAASConfPath`, `MNAASLocalLogPath`, `MNAAS_Daily_Aggregation_PathName`).  
- **Environment variables**: None beyond those defined in the common properties file.  

# Improvements
1. **Parameterize log date format** – expose a variable (e.g., `MNAAS_LogDateFormat`) to allow custom suffixes without editing the file.  
2. **Add validation block** – script should verify existence and write permission of all paths and the presence of required Java classes before job start.