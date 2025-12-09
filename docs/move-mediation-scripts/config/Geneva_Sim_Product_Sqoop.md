# Summary
Defines environment‑specific parameters for the **Geneva Sim Product Sqoop** ingestion step of the Move‑Mediation pipeline. Sources common MNAAS properties, sets log location, HDFS staging directory, Sqoop import query, and post‑import split‑insert query used by nightly Hadoop jobs to load `gen_sim_product` data into the `mnaas.gen_sim_product_split` Hive table.

# Key Components
- **MNAAS_CommonProperties.properties** – sourced file providing base paths (`MNAASLocalLogPath`, `MNAASConfPath`, etc.).
- **Geneva_Sim_Product_SqoopLog** – full path of the date‑stamped Sqoop log file.
- **Geneva_Sim_Product_SqoopDir** – HDFS directory where Sqoop writes raw `gen_sim_product` files.
- **Geneva_Sim_Product_SqoopQuery** – Sqoop free‑form query with `$CONDITIONS` placeholder for split‑by‑column handling.
- **Geneva_Sim_Product_SplitQuery** – Hive/Impala `INSERT` statement that normalises multi‑value `PROPOSITION` field using `lateral view posexplode(split(...,'!'))`.

# Data Flow
1. **Input**: Relational source table `gen_sim_product` (presumably Oracle/SQL Server).  
2. **Sqoop Import**: Executes `Geneva_Sim_Product_SqoopQuery` → writes CSV/Avro files to `Geneva_Sim_Product_SqoopDir` on HDFS.  
3. **Log**: Writes operational details to `Geneva_Sim_Product_SqoopLog`.  
4. **Post‑Import**: Hive/Impala executes `Geneva_Sim_Product_SplitQuery` → populates `mnaas.gen_sim_product_split`.  
5. **Side Effects**: HDFS directory populated; Hive table updated; log file appended.

# Integrations
- **Shell Wrapper** (e.g., `Geneva_Sim_Product_Sqoop.sh`) reads this properties file to construct the Sqoop command and subsequent Hive query.  
- **MNAAS_CommonProperties.properties** supplies shared environment variables used across all Move‑Mediation scripts.  
- **Hive/Impala** engine consumes `Geneva_Sim_Product_SplitQuery`.  
- **Scheduler** (Oozie/Airflow/cron) triggers the wrapper nightly.

# Operational Risks
- **Date‑stamp in log path** may create a new log file each run, leading to log proliferation. *Mitigation*: rotate/compress old logs.  
- **Hard‑coded column list** in Sqoop query may break if source schema changes. *Mitigation*: schema versioning or dynamic column discovery.  
- **`$CONDITIONS` misuse** can cause uneven splits or job failures if split column not indexed. *Mitigation*: verify split column suitability and monitor Sqoop job exit codes.  
- **`coalesce(cast(... as int), -999)`** introduces sentinel values that may propagate to downstream analytics. *Mitigation*: document sentinel usage and handle in downstream validation.

# Usage
```bash
# Load common properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Source this file
. /path/to/Geneva_Sim_Product_Sqoop.properties

# Execute Sqoop import (example wrapper)
sqoop import \
  --connect "$MNAAS_JDBC_URL" \
  --username "$MNAAS_DB_USER" \
  --password "$MNAAS_DB_PASS" \
  --query "$Geneva_Sim_Product_SqoopQuery" \
  --target-dir "$Geneva_Sim_Product_SqoopDir" \
  --split-by "PRODUCT_CODE" \
  --as-textfile \
  --null-string '\\N' \
  --null-non-string '\\N' \
  --verbose \
  2>&1 | tee -a "$Geneva_Sim_Product_SqoopLog"

# Run Hive split query
hive -e "$Geneva_Sim_Product_SplitQuery"
```

# Configuration
- **Referenced Config Files**:  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Environment Variables Expected** (from common properties):  
  - `MNAASLocalLogPath` – base local log directory.  
  - `MNAAS_JDBC_URL`, `MNAAS_DB_USER`, `MNAAS_DB_PASS` – DB connection credentials.  
  - `MNAAS_HIVE_DB` – default Hive database (if used in wrapper).  

# Improvements
1. **Parameterize Split Column** – expose `split-by` column as a configurable property to avoid hard‑coding `PRODUCT_CODE`.  
2. **Centralised Log Management** – replace per‑run log file naming with a rotating logger (e.g., log4j) and archive old logs automatically.