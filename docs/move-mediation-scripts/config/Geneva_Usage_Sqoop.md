# Summary
`Geneva_Usage_Sqoop.properties` defines environment‑specific variables for the Geneva Usage product ingestion step. It sources shared MNAAS properties, sets a date‑stamped log path, HDFS staging directory, target Hive table name, and the Sqoop import query that extracts rows from the `usage_geneva` source table for a fixed billing month and cut‑off timestamp. Down‑stream scripts use these variables to execute a Sqoop import and load the data into the `gen_usage_product` Hive table.

# Key Components
- **Source statement** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  *Loads base paths and common configuration variables.*
- **Geneva_Usage_SqoopLog** – `$MNAASLocalLogPath/usage_geneva_Sqoop.log$(date +_%F)`  
  *Log file location with daily suffix.*
- **Geneva_Usage_SqoopDir** – `/user/MNAAS/sqoop/usage_geneva`  
  *HDFS staging directory for Sqoop import files.*
- **gen_usage_product_tblname** – `gen_usage_product`  
  *Target Hive table (used by downstream Hive load script).*
- **Geneva_Usage_SqoopQuery** – Full SELECT statement with hard‑coded `bill_month='2025-01'` and `insert_date` cut‑off, ending with `\$CONDITIONS` placeholder for Sqoop partitioning.

# Data Flow
| Stage | Input | Process | Output | Side Effects |
|-------|-------|---------|--------|--------------|
| 1. Property load | `MNAAS_CommonProperties.properties` | Bash `source` | Environment variables (`MNAASLocalLogPath`, etc.) | None |
| 2. Sqoop import | `usage_geneva` table in source RDBMS (Oracle/SQL) | Sqoop executes `Geneva_Usage_SqoopQuery` with `$CONDITIONS` for parallelism | Files (e.g., `part-m-00000`) in `$Geneva_Usage_SqoopDir` on HDFS | Log entry written to `$Geneva_Usage_SqoopLog` |
| 3. Hive load (external script) | HDFS files from step 2 | `LOAD DATA INPATH ... INTO TABLE gen_usage_product` | Data available in Hive `gen_usage_product` table | May trigger downstream analytics pipelines |

# Integrations
- **MNAAS_CommonProperties.properties** – provides base log path and any shared Hadoop/YARN settings.  
- **Sqoop** – invoked by a wrapper script that reads `Geneva_Usage_SqoopQuery` and other variables.  
- **HDFS** – staging directory is part of the MNAAS HDFS namespace.  
- **Hive** – downstream Hive DDL/ETL scripts reference `gen_usage_product_tblname`.  
- **Logging subsystem** – writes to a date‑stamped file under `$MNAASLocalLogPath`.  
- **Scheduler (e.g., Oozie/Airflow)** – schedules the wrapper script that sources this properties file.

# Operational Risks
- **Hard‑coded dates** (`bill_month='2025-01'`, cut‑off timestamp) → stale data if not updated each run. *Mitigation: externalize dates to variables or pass as parameters.*  
- **Missing source properties file** → undefined `$MNAASLocalLogPath` causing script failure. *Mitigation: add existence check and fail‑fast error handling.*  
- **Log file growth** – daily log files never rotated. *Mitigation: implement logrotate or size‑based cleanup.*  
- **Sqoop `$CONDITIONS` misuse** – if wrapper script does not supply `--split-by` correctly, import may fall back to single mapper, causing performance bottlenecks. *Mitigation: enforce split column and mapper count in wrapper.*  
- **Permission drift on HDFS staging dir** – write failures if ACLs change. *Mitigation: verify/repair permissions before import.*

# Usage
```bash
# 1. Source the properties
source /path/to/move-mediation-scripts/config/Geneva_Usage_Sqoop.properties

# 2. Run the Sqoop import (example wrapper)
sqoop import \
  --connect jdbc:oracle:thin:@//dbhost:1521/PROD \
  --username $DB_USER --password $DB_PASS \
  --query "$Geneva_Usage_SqoopQuery" \
  --target-dir "$Geneva_Usage_SqoopDir" \
  --split-by secs_id \
  --num-mappers 4 \
  --as-avrodatafile \
  --null-string '\\N' --null-non-string '\\N' \
  --verbose \
  2>&1 | tee -a "$Geneva_Usage_SqoopLog"
```
*Debug*: echo variables after sourcing; run Sqoop with `--dry-run` (if supported) or add `--verbose`.

# Configuration
- **Environment variables** (populated by `MNAAS_CommonProperties.properties`):  
  - `MNAASLocalLogPath` – base directory for logs.  
  - Any Hadoop/YARN configuration overrides (e.g., `HADOOP_CONF_DIR`).  
- **External config files**:  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` (must be present and readable).  
- **Database credentials**: typically supplied via separate secure mechanism (e.g., Kerberos ticket, credential store) – not defined in this file.

# Improvements
1. **Parameterize temporal filters** – replace hard‑coded `bill_month` and `insert_date` values with `${BILL_MONTH}` and `${CUT_OFF_TS}` variables, allowing the scheduler to inject the correct period per run.  
2. **Add validation block** – at the end of the file, include a Bash function that checks required variables (`MNAASLocalLogPath`, `Geneva_Usage_SqoopDir`, etc.) and exits with a clear error code if any are missing or malformed.