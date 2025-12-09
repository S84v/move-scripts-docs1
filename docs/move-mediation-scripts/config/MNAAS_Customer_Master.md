# Summary
`MNAAS_Customer_Master.properties` supplies runtime configuration for the customer‑master ingestion pipeline. It imports common MNAAS properties and defines the log directory, HDFS staging path, and temporary Hive table name used by the Sqoop job that extracts the customer master dataset during a network‑move operation.

# Key Components
- **Source of common properties** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  *Loads shared variables (e.g., `MNAASLocalLogPath`).*
- **`cust_master_SqoopLog`** – Path for Sqoop job logs specific to the customer‑master load.  
- **`cust_HDFSSqoopDir`** – HDFS directory where Sqoop imports the raw customer‑master files.  
- **`cust_master_tblname`** – Temporary Hive/Impala table name (`customer_master_temp`) used for downstream transformations.

# Data Flow
| Stage | Input | Process | Output | Side Effects |
|-------|-------|---------|--------|--------------|
| 1 | `MNAAS_CommonProperties.properties` | Variable import | Populates environment for downstream scripts | None |
| 2 | Sqoop job (triggered by downstream Bash script) | Uses `cust_HDFSSqoopDir` as target HDFS path; writes logs to `cust_master_SqoopLog` | Raw CSV/TSV files in HDFS; log files on local FS | HDFS write, local FS write |
| 3 | Hive/Impala job | Loads data from `cust_HDFSSqoopDir` into `cust_master_tblname` | Temporary table `customer_master_temp` | Table creation, metadata registration |

# Integrations
- **Bash pipeline** `MNAAS_Customer_backup_files_process.sh` sources this file to obtain the three variables.  
- **Sqoop** command constructed in the pipeline references `cust_HDFSSqoopDir` and `cust_master_SqoopLog`.  
- **Hive/Impala** scripts reference `cust_master_tblname` for downstream joins with CDR data.  
- **Common properties file** provides base paths (`MNAASLocalLogPath`) and Hadoop configuration (e.g., `HADOOP_CONF_DIR`).

# Operational Risks
- **Path mismatch** – If `MNAASLocalLogPath` changes, log directory may become invalid. *Mitigation*: Validate existence of `$cust_master_SqoopLog` before job start.
- **HDFS permission errors** – Sqoop may lack write permission on `cust_HDFSSqoopDir`. *Mitigation*: Enforce ACLs and include pre‑flight HDFS `hdfs dfs -test -d` check.
- **Table name collision** – Re‑using `customer_master_temp` across concurrent runs can cause data overwrite. *Mitigation*: Append run‑timestamp or UUID to table name in the pipeline.

# Usage
```bash
# Source the properties
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source /path/to/move-mediation-scripts/config/MNAAS_Customer_Master.properties

# Verify variables
echo "Log dir: $cust_master_SqoopLog"
echo "HDFS dir: $cust_HDFSSqoopDir"
echo "Temp table: $cust_master_tblname"

# Example Sqoop import (executed by the pipeline)
sqoop import \
  --connect jdbc:mysql://dbhost/customerdb \
  --username $DB_USER --password $DB_PASS \
  --table customer_master \
  --target-dir $cust_HDFSSqoopDir \
  --as-textfile \
  --null-string '\\N' \
  --null-non-string '\\N' \
  --verbose 2>&1 | tee $cust_master_SqoopLog/sqoop_$(date +%Y%m%d%H%M%S).log
```

# Configuration
- **Environment variables** (in `MNAAS_CommonProperties.properties`):  
  - `MNAASLocalLogPath` – Base directory for all MNAAS logs.  
  - Hadoop/HDFS configuration variables (`HADOOP_CONF_DIR`, `HADOOP_USER_NAME`).  
- **Referenced config files**:  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` (must be present and readable).  

# Improvements
1. **Parameterize table name** – Replace static `customer_master_temp` with a pattern that includes a run identifier to avoid collisions in parallel executions.  
2. **Add validation block** – Insert a sanity‑check function that confirms `$cust_master_SqoopLog` and `$cust_HDFSSqoopDir` exist and are writable before any Sqoop job is launched.