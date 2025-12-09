# Summary
Defines Bash‑sourced runtime constants for the **MNAAS_Usage_Trend_Aggr** job. The constants configure process‑status tracking, logging, Hive database/table names, HDFS aggregation paths, Java aggregation/loading class names, and the location of a static usage‑trend data file used in the telecom mediation pipeline’s daily usage‑trend aggregation step.

# Key Components
- **MNAAS_usage_trend_Aggr_ProcessStatusFileName** – HDFS file that records job status (STARTED, SUCCESS, FAIL).  
- **MNAAS_usage_trend_AggrLogPath** – Local log file path with daily timestamp.  
- **Dname_MNAAS_Load_Daily_user_trend_daily_aggr_temp** – Hive staging table for intermediate aggregation.  
- **Dname_MNAAS_usage_trend_aggregation** – Hive table that stores aggregated usage‑trend data.  
- **Dname_MNAAS_usage_trend_load** – Hive table populated by the loading phase.  
- **MNAAS_Daily_Aggregation_Usertrend_PathName** – HDFS directory for temporary user‑trend files.  
- **MNAAS_usage_trend_aggregation_classname** – Fully‑qualified Java class implementing the aggregation logic.  
- **MNAAS_usage_trend_load_classname** – Fully‑qualified Java class implementing the loading logic.  
- **MNAAS_User_Trend_Data_filename** – HDFS file containing reference user‑trend data used by the job.

# Data Flow
1. **Input**: Raw daily usage records in HDFS (outside scope).  
2. **Aggregation Phase**: Java class `com.tcl.mnass.aggregation.usage_trend_aggregation` reads raw data, writes intermediate results to `Dname_MNAAS_Load_Daily_user_trend_daily_aggr_temp` (Hive) and to `MNAAS_Daily_Aggregation_Usertrend_PathName`.  
3. **Loading Phase**: Java class `com.tcl.mnass.aggregation.usage_trend_loading` reads intermediate Hive table and `MNAAS_User_Trend_Data_filename`, writes final rows to `Dname_MNAAS_usage_trend_aggregation` and `Dname_MNAAS_usage_trend_load`.  
4. **Side Effects**: Updates `MNAAS_usage_trend_Aggr_ProcessStatusFileName` with job state; appends logs to `MNAAS_usage_trend_AggrLogPath`.  
5. **External Services**: Hive Metastore, HDFS, optional YARN container for Java jobs.

# Integrations
- **MNAAS_CommonProperties.properties** – imported for base paths (`MNAASConfPath`, `MNAASLocalLogPath`, `MNAAS_Daily_Aggregation_PathName`).  
- **Other mediation jobs** – downstream jobs consume `Dname_MNAAS_usage_trend_load`; upstream jobs populate raw usage data consumed by this job.  
- **Lock/Coordination** – not defined here but follows same HDFS lock convention used across mediation scripts.

# Operational Risks
- **Stale Process‑Status File** – may cause duplicate runs; mitigate by atomic rename on job start/completion.  
- **Log Rotation Failure** – daily log file may grow unchecked; implement log‑rotate policy.  
- **Hive Table Schema Drift** – changes to staging or target tables break Java classes; enforce schema versioning and CI validation.  
- **Missing Reference Data** (`MNAAS_User_Trend_Data_filename`) – job aborts; add pre‑flight existence check.

# Usage
```bash
# Source common properties and this job’s constants
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
. /path/to/move-mediation-scripts/config/MNAAS_Usage_Trend_Aggr.properties

# Export constants for downstream scripts (if needed)
export MNAAS_usage_trend_Aggr_ProcessStatusFileName
export MNAAS_usage_trend_AggrLogPath
# ...

# Execute aggregation
java -cp $MNAAS_JarPath $MNAAS_usage_trend_aggregation_classname \
     --output-table $Dname_MNAAS_usage_trend_aggregation \
     --temp-table $Dname_MNAAS_Load_Daily_user_trend_daily_aggr_temp

# Execute loading
java -cp $MNAAS_JarPath $MNAAS_usage_trend_load_classname \
     --source-table $Dname_MNAAS_usage_trend_aggregation \
     --target-table $Dname_MNAAS_usage_trend_load
```

# Configuration
- **Referenced Config File**: `MNAAS_CommonProperties.properties` (defines `$MNAASConfPath`, `$MNAASLocalLogPath`, `$MNAAS_Daily_Aggregation_PathName`, `$MNAAS_JarPath`, etc.).  
- **Environment Variables**: None beyond those exported by the common properties file.  

# Improvements
1. **Add Validation Block** – script should verify existence and permissions of all HDFS paths and reference files before launching Java jobs.  
2. **Parameterize Log Rotation** – introduce a configurable log‑retention policy (e.g., `MNAAS_Usage_Trend_Aggr_LogRetentionDays`).