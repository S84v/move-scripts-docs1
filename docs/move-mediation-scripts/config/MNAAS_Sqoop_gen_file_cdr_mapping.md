# Summary
`MNAAS_Sqoop_gen_file_cdr_mapping.properties` is a Bash‑sourced configuration fragment that defines runtime constants for the **Gen‑File CDR Mapping** Sqoop import step in the MNAAS daily‑processing pipeline. It supplies process‑status file paths, log file naming, script name, HDFS staging directory, target Hive table name, and the Oracle SELECT query used by `MNAAS_Sqoop_gen_file_cdr_mapping.sh` to import the previous month’s CDR‑mapping records into a staging Hive/Impala table.

# Key Components
- **Source of common properties**  
  `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Process‑status file**  
  `gen_file_cdr_mapping_Sqoop_ProcessStatusFileName`
- **Sqoop log file (dated)**  
  `gen_file_cdr_mapping_table_SqoopLogName`
- **Script identifier**  
  `MNAAS_Sqoop_gen_file_cdr_mapping_Scriptname`
- **HDFS staging directory**  
  `gen_file_cdr_mapping_table_Dir`
- **Target Hive table name**  
  `gen_file_cdr_mapping_table_name`
- **Oracle import query** (`$CONDITIONS` placeholder for Sqoop split)  
  `gen_file_cdr_mapping_table_Query`

# Data Flow
| Stage | Input | Transformation | Output | Side Effects |
|------|-------|----------------|--------|--------------|
| 1. Sqoop launch (via `MNAAS_Sqoop_gen_file_cdr_mapping.sh`) | Oracle DB `gen_file_cdr_mapping` view/table | Executes `gen_file_cdr_mapping_table_Query` filtered to previous month (`add_months(trunc(sysdate),-1)`) and Sqoop split conditions | HDFS directory `$SqoopPath/gen_file_cdr_mapping` containing imported files | Writes process‑status file, appends to dated log file, updates Hive external table `gen_file_cdr_mapping` |

External services: Oracle database (source), HDFS (staging), Hive/Impala (target), Linux filesystem (logs/status).

# Integrations
- **Common properties** from `MNAAS_CommonProperties.properties` (defines `$MNAASConfPath`, `$MNAASLocalLogPath`, `$SqoopPath`, etc.).
- **Shell script** `MNAAS_Sqoop_gen_file_cdr_mapping.sh` consumes all variables defined here.
- **Downstream Hive/Impala jobs** that read from the `gen_file_cdr_mapping` staging table.
- **Monitoring/alerting** that watches the process‑status file for success/failure.

# Operational Risks
- **Date logic mismatch**: Query relies on Oracle `sysdate`; if the pipeline runs outside the expected window, wrong month may be imported. *Mitigation*: Validate `$CURRENT_MONTH` before execution.
- **Missing `$CONDITIONS` substitution**: Incorrect Sqoop split configuration can cause full table scan or duplicate imports. *Mitigation*: Ensure Sqoop `--split-by` column is indexed and `$CONDITIONS` is passed unchanged.
- **Log file growth**: Log name includes date but not rotation; daily logs accumulate. *Mitigation*: Implement log rotation or compression.
- **Credential exposure**: Oracle connection details are inherited from common properties; insecure permissions could leak credentials. *Mitigation*: Restrict file mode to 600 and store passwords in a vault.

# Usage
```bash
# Load configuration
source /app/hadoop_users/MNAAS/move-mediation-scripts/config/MNAAS_Sqoop_gen_file_cdr_mapping.properties

# Run the import (example)
bash $MNAAS_Sqoop_gen_file_cdr_mapping_Scriptname \
    --status-file "$gen_file_cdr_mapping_Sqoop_ProcessStatusFileName" \
    --log-file   "$gen_file_cdr_mapping_table_SqoopLogName"
```
To debug, set `-x` on the shell script or export `SQOOP_DEBUG=1` before execution.

# Configuration
- **Environment variables** (populated by common properties):  
  `MNAASConfPath`, `MNAASLocalLogPath`, `SqoopPath`, Oracle connection strings (`ORACLE_USER`, `ORACLE_PASS`, `ORACLE_TNS`), Hadoop/Hive settings.
- **Referenced config file**:  
  `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Local overrides**: None; all values are defined directly in this file.

# Improvements
1. **Parameterize month** – replace hard‑coded `add_months(trunc(sysdate),-1)` with an external variable (e.g., `TARGET_MONTH`) to allow back‑fills and ad‑hoc runs.  
2. **Add log rotation** – integrate `logrotate` or append a timestamped suffix and purge logs older than 30 days automatically.