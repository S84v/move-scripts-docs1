# Summary
Defines runtime constants for the **GBS Average Rate** batch job. It imports common properties, then sets log, HDFS, and Hive table locations used by the GBS rate Sqoop import process in the Move‑Mediation production environment.

# Key Components
- `gbs_rate_SqoopLog` – Local filesystem path for Sqoop job log files.  
- `rate_HDFSSqoopDir` – HDFS directory where the Sqoop‑imported GBS rate data is staged.  
- `gbs_rate_tblname` – Hive table name that will be populated with the current average conversion rate.

# Data Flow
1. **Input**: Source relational table (Oracle/SQL) containing GBS rate data (referenced indirectly by the Sqoop job).  
2. **Processing**: Sqoop reads the source table and writes CSV/Parquet files to `rate_HDFSSqoopDir`.  
3. **Output**: Files are loaded into Hive table `gbs_curr_avgconv_rate`.  
4. **Side Effects**: Log entries written to `gbs_rate_SqoopLog`; HDFS directory populated; Hive metastore updated.  
5. **External Services**: Hadoop HDFS, Hive, Sqoop, underlying RDBMS (Oracle/SQL Server).

# Integrations
- **MNAAS_CommonProperties.properties** – Provides base variables such as `MNAASLocalLogPath`.  
- **MNAAS_GBS_Avg_Rate.sh** (or similarly named driver script) – Sources this properties file to obtain paths before invoking Sqoop and Hive commands.  
- **Job Scheduler (e.g., Oozie/Airflow/Cron)** – Executes the driver script on a defined cadence, relying on these constants.

# Operational Risks
- **Path Misconfiguration** – Incorrect `MNAASLocalLogPath` leads to log write failures; mitigate by validating directory existence at job start.  
- **HDFS Permission Issues** – Sqoop may lack write permission to `rate_HDFSSqoopDir`; mitigate with pre‑run ACL checks.  
- **Hive Table Schema Drift** – Changes to source table not reflected in Hive table `gbs_curr_avgconv_rate`; mitigate with schema version control and automated validation.  
- **Resource Contention** – Large Sqoop imports can saturate network/HDFS; mitigate by throttling or scheduling during off‑peak windows.

# Usage
```bash
# Source common and job‑specific properties
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_GBS_Avg_Rate.properties

# Run Sqoop import (example)
sqoop import \
  --connect $ORACLE_JDBC_URL \
  --username $ORACLE_USER \
  --password $ORACLE_PWD \
  --table GBS_RATE_TABLE \
  --target-dir $rate_HDFSSqoopDir \
  --as-parquetfile \
  --m 4 \
  --verbose \
  2>&1 | tee $gbs_rate_SqoopLog/$(date +%Y%m%d_%H%M%S).log

# Load into Hive (example)
hive -e "USE $HiveDB; LOAD DATA INPATH '$rate_HDFSSqoopDir' OVERWRITE INTO TABLE $gbs_rate_tblname;"
```

# Configuration
- **Environment Variables** (provided by `MNAAS_CommonProperties.properties`):  
  - `MNAASLocalLogPath` – Base directory for all job logs.  
  - `HiveDB` – Target Hive database.  
  - Database connection strings/credentials (`ORACLE_JDBC_URL`, `ORACLE_USER`, `ORACLE_PWD`).  
- **Referenced Config Files**:  
  - `MNAAS_CommonProperties.properties` (global constants).  
  - This file (`MNAAS_GBS_Avg_Rate.properties`) for job‑specific paths.

# Improvements
1. **Parameterize Source Table** – Add a property for the source relational table name to avoid hard‑coding within the driver script.  
2. **Add Validation Block** – Include a pre‑execution check that verifies existence and write permissions of `gbs_rate_SqoopLog` and `rate_HDFSSqoopDir`, aborting with a clear error if validation fails.