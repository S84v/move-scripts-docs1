# Summary
Defines environment‑specific parameters for the **Geneva Usage Product Sqoop** ingestion step of the Move‑Mediation pipeline. It sources shared MNAAS properties, sets the log file path (date‑stamped), HDFS staging directory, target Hive table name, and the Sqoop import query used to extract `v_gen_usage_product` rows into the `gen_usage_product` Hive table.

# Key Components
- **Source statement** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  Loads common base paths (`MNAASLocalLogPath`, etc.).
- **Geneva_Usage_Product_SqoopLog** – Log file location with daily suffix.  
- **Geneva_Usage_Product_SqoopDir** – HDFS staging directory for Sqoop output.  
- **gen_usage_product_tblname** – Target Hive table identifier.  
- **Geneva_Usage_Product_SqoopQuery** – Parameterised SQL query executed by Sqoop; uses `$CONDITIONS` placeholder for split‑by logic.

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| Sqoop import | Oracle view `v_gen_usage_product` (via JDBC) | Executes `Geneva_Usage_Product_SqoopQuery` with `$CONDITIONS` for parallelism | Files in `Geneva_Usage_Product_SqoopDir` (HDFS) → later loaded into Hive table `gen_usage_product` | Writes to `Geneva_Usage_Product_SqoopLog`; may generate temporary staging files. |

# Integrations
- **MNAAS_CommonProperties.properties** – Provides base directories (`MNAASLocalLogPath`, `MNAASConfPath`).  
- **Sqoop execution scripts** (e.g., `run_geneva_usage_product_sqoop.sh`) source this file to obtain variables for the `sqoop import` command.  
- **Hive** – Subsequent Hive scripts consume the HDFS files to populate `mnaas.gen_usage_product_split`.  
- **Oracle DB** – Source of `v_gen_usage_product` view.

# Operational Risks
- **Missing/incorrect common properties** → job fails to locate log or HDFS paths. *Mitigation*: Validate source file existence before execution.  
- **Date‑stamp format in log path** may produce illegal filenames on locale changes. *Mitigation*: Enforce POSIX date format (`%F`).  
- **SQL query syntax errors or schema changes** in `v_gen_usage_product`. *Mitigation*: Unit‑test query against a dev DB; version‑control schema.  
- **Insufficient split‑by column** leading to skewed parallelism. *Mitigation*: Choose a high‑cardinality column for `$CONDITIONS`.  
- **HDFS permission issues** on `/user/MNAAS/sqoop/usage_product`. *Mitigation*: Pre‑run ACL check; alert on permission failures.

# Usage
```bash
# Load common properties and this config
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source /path/to/move-mediation-scripts/config/Geneva_Usage_Product_Sqoop.properties

# Example Sqoop command (as used by wrapper script)
sqoop import \
  --connect jdbc:oracle:thin:@//dbhost:1521/ORCL \
  --username $DB_USER --password $DB_PASS \
  --query "$Geneva_Usage_Product_SqoopQuery" \
  --target-dir "$Geneva_Usage_Product_SqoopDir" \
  --split-by product_code \
  --num-mappers 8 \
  --as-avrodatafile \
  --compress \
  --log-dir "$(dirname $Geneva_Usage_Product_SqoopLog)" \
  --outdir $MNAASConfPath \
  2>&1 | tee -a $Geneva_Usage_Product_SqoopLog
```
To debug, echo variables after sourcing and run the Sqoop command with `--verbose`.

# Configuration
- **Environment variables** (provided by `MNAAS_CommonProperties.properties`):  
  - `MNAASLocalLogPath` – Base directory for logs.  
  - `MNAASConfPath` – Directory for generated Java source files.  
- **Properties defined in this file**: `Geneva_Usage_Product_SqoopLog`, `Geneva_Usage_Product_SqoopDir`, `gen_usage_product_tblname`, `Geneva_Usage_Product_SqoopQuery`.  
- **External dependencies**: Oracle JDBC driver on classpath, Hadoop/HDFS client, Sqoop binary.

# Improvements
1. **Externalise the SQL query** to a separate `.sql` file and reference it via `$(cat ...)` to simplify version control and allow DBAs to edit without redeploying scripts.  
2. **Parameterise split‑by column and mapper count** via environment variables to adapt to data volume changes without code modification.