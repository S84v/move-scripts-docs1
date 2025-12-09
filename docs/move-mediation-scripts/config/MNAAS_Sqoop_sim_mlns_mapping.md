# Summary
`MNAAS_Sqoop_sim_mlns_mapping.properties` is a Bash‑sourced configuration fragment that defines all runtime constants required by the **SIM MLNS Mapping** Sqoop import step in the MNAAS daily‑processing pipeline. It supplies process‑status tracking, logging, script identification, HDFS staging paths, target Hive table names, and the Oracle `SELECT` query used by `MNAAS_Sqoop_sim_mlns_mapping.sh` to import the `sim_mlns_mapping` table into a staging Hive/Impala table.

# Key Components
- **Source of common properties** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Process‑status file** – `SIM_MLNS_Mapping_Sqoop_ProcessStatusFile`
- **Log file name** – `SIM_MLNS_Mapping_table_SqoopLogName`
- **Script identifier** – `MNAAS_Sqoop_sim_mlns_mapping_Scriptname`
- **HDFS staging directory** – `sim_mlns_mapping_table_Dir`
- **Temporary Hive table** – `sim_mlns_mapping_temp_table_name`
- **Target Hive table** – `sim_mlns_mapping_table_name`
- **Oracle query template** – `sim_mlns_mapping_table_Query`

# Data Flow
| Stage | Input | Transformation | Output | Side Effects |
|-------|-------|----------------|--------|--------------|
| 1. Configuration load | `MNAAS_CommonProperties.properties` | Variable expansion | In‑memory Bash variables | None |
| 2. Sqoop execution (in `MNAAS_Sqoop_sim_mlns_mapping.sh`) | Oracle DB `sim_mlns_mapping` (filtered by `$CONDITIONS`) | Sqoop import → HDFS staging dir (`sim_mlns_mapping_table_Dir`) | Parquet/Avro files in HDFS | Writes to process‑status file, appends to log |
| 3. Hive/Impala load (within the same script) | HDFS staging files | `INSERT OVERWRITE` into `sim_mlns_mapping` (or temp table) | Populated Hive table `sim_mlns_mapping` | May trigger Hive metastore updates |

External services:
- Oracle database (source)
- HDFS (staging)
- Hive/Impala (target)
- Local filesystem for logs and status files

# Integrations
- **Common properties** (`MNAAS_CommonProperties.properties`) provide base paths (`MNAASConfPath`, `MNAASLocalLogPath`, `SqoopPath`) and shared credentials.
- **Sqoop driver script** `MNAAS_Sqoop_sim_mlns_mapping.sh` sources this file to obtain all runtime constants.
- **Monitoring/Orchestration** systems poll the process‑status file to determine success/failure.
- **Log aggregation** tools ingest `MNAAS_Sqoop_sim_mlns_mapping.log*`.

# Operational Risks
- **Missing or stale common properties** → script fails to resolve paths. *Mitigation*: validate existence of sourced file at start.
- **Hard‑coded date in log name** may cause log rotation issues if the script runs multiple times per day. *Mitigation*: include timestamp or use log rotation.
- **Schema drift** in `sim_mlns_mapping` (column addition/removal) can break the `SELECT *` query. *Mitigation*: versioned query or explicit column list.
- **Credential leakage** if common properties contain plaintext passwords. *Mitigation*: use Kerberos or encrypted credential store.
- **Insufficient disk space on HDFS** → import aborts. *Mitigation*: pre‑run HDFS quota check.

# Usage
```bash
# 1. Source the configuration
source /app/hadoop_users/MNAAS/move-mediation-scripts/config/MNAAS_Sqoop_sim_mlns_mapping.properties

# 2. Execute the driver script (normally invoked by scheduler)
bash $MNAASScriptPath/$MNAAS_Sqoop_sim_mlns_mapping_Scriptname

# 3. Debug – print variables
echo "Log: $SIM_MLNS_Mapping_table_SqoopLogName"
echo "Query: $sim_mlns_mapping_table_Query"
```

# Configuration
- **Environment variables / paths defined in `MNAAS_CommonProperties.properties`**  
  - `MNAASConfPath` – directory for process‑status files  
  - `MNAASLocalLogPath` – base log directory  
  - `SqoopPath` – root directory for Sqoop staging directories  
- **Variables defined in this file** (see *Key Components*).  
- **Oracle connection parameters** are inherited from the common properties file (e.g., `ORACLE_JDBC_URL`, `ORACLE_USER`, `ORACLE_PASS`).

# Improvements
1. **Externalize the SQL query** to a separate `.sql` file and reference it via a variable; enables version control and easier editing.  
2. **Add a validation block** at the top of the file to assert that all required base paths and credentials are non‑empty, exiting with a clear error message if any are missing.