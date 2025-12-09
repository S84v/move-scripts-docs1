# Summary
`MNAAS_Sqoop_ba_rep_data.properties` is a Bash‑sourced configuration fragment that defines runtime constants for the **Bare‑Report data Sqoop** step of the MNAAS daily‑processing pipeline. It supplies Oracle connection details, script metadata, status‑file and log locations, HDFS staging paths, target Hive table name, and the Sqoop import query used by `MNAAS_Sqoop_ba_rep_data.sh` to pull the previous month’s billing‑report records from the Oracle source into the staging Hive/Impala table `ra_calldate_gen_succ_rej`.

# Key Components
- **Source statement** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` – imports shared constants (e.g., `$MNAASConfPath`, `$MNAASLocalLogPath`, `$SqoopPath`).
- **Environment variable** – `ORACLE_HOME=/rdbms/oracle/12.1.0.2/client` – points to the Oracle client required by Sqoop.
- **Oracle connection parameters** – `OrgDetails_ServerName`, `OrgDetails_PortNumber`, `OrgDetails_Service`, `OrgDetails_Username`, `OrgDetails_Password`.
- **Process‑status file** – `ba_rep_data_Sqoop_ProcessStatusFile`.
- **Log file definition** – `ba_rep_data_table_SqoopLogName` (includes date suffix).
- **Script identifier** – `MNAAS_Sqoop_ba_rep_data_Scriptname`.
- **HDFS staging directory** – `ba_rep_data_table_Dir`.
- **Target Hive table** – `ba_rep_data_table_name`.
- **Sqoop import query** – `ba_rep_data_table_Query` (parameterized with `$CONDITIONS` and month filter).

# Data Flow
| Stage | Input | Transformation | Output | Side Effects |
|-------|-------|----------------|--------|--------------|
| 1 | Oracle DB `ba_rep_data` (host `OrgDetails_ServerName`, service `comprd`) | Sqoop executes `ba_rep_data_table_Query` (filters to previous month) | Rows written to HDFS path `$ba_rep_data_table_Dir` as text/Parquet | Creates/updates `$ba_rep_data_Sqoop_ProcessStatusFile` and log `$ba_rep_data_table_SqoopLogName`. |
| 2 | HDFS staging files | Hive/Impala `LOAD DATA` (performed by downstream script) | Populates Hive table `ra_calldate_gen_succ_rej`. | May trigger downstream data pipelines. |

External services: Oracle database, HDFS, Hive/Impala, Linux filesystem for logs/status files.

# Integrations
- **MNAAS_CommonProperties.properties** – provides base paths (`$MNAASConfPath`, `$MNAASLocalLogPath`, `$SqoopPath`) and common environment settings.
- **MNAAS_Sqoop_ba_rep_data.sh** – consumes all exported variables; orchestrates Sqoop command, status handling, and logging.
- **Downstream Hive/Impala jobs** – read the staging directory populated by this Sqoop run.
- **Monitoring/alerting** – may watch the process‑status file for success/failure.

# Operational Risks
- **Hard‑coded credentials** – exposure risk; mitigate by moving to a credential vault or Kerberos keytab.
- **Static month calculation** – relies on Oracle `add_months(trunc(sysdate),-1)`; if pipeline runs late, data may be missing. Mitigate with configurable run‑date parameter.
- **Path concatenation without validation** – malformed `$MNAASConfPath` could cause log/status files to be written to unexpected locations. Validate paths at startup.
- **Oracle client version mismatch** – `ORACLE_HOME` points to a specific client; upgrades may break connectivity. Pin version in deployment docs and test after upgrades.
- **Missing `$CONDITIONS` substitution** – Sqoop requires `--query` with `$CONDITIONS`; if the script omits it, the import will fail. Ensure script passes the placeholder unchanged.

# Usage
```bash
# Load configuration into the current shell
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Sqoop_ba_rep_data.properties

# Execute the associated script (debug mode adds -x)
bash $MNAASConfPath/../MNAAS_Sqoop_ba_rep_data.sh   # or full path to script
```
To debug, prepend `set -x` inside `MNAAS_Sqoop_ba_rep_data.sh` or run:
```bash
bash -x MNAAS_Sqoop_ba_rep_data.sh
```

# Configuration
- **Referenced files**  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` (shared constants)  
- **Exported environment variables**  
  - `ORACLE_HOME`  
  - `OrgDetails_ServerName`, `OrgDetails_PortNumber`, `OrgDetails_Service`, `OrgDetails_Username`, `OrgDetails_Password`  
  - `ba_rep_data_Sqoop_ProcessStatusFile`  
  - `ba_rep_data_table_SqoopLogName`  
  - `MNAAS_Sqoop_ba_rep_data_Scriptname`  
  - `ba_rep_data_table_Dir`  
  - `ba_rep_data_table_name`  
  - `ba_rep_data_table_Query`  

# Improvements
1. **Externalize secrets** – replace `OrgDetails_Username`/`Password` with a secure vault lookup (e.g., HashiCorp Vault, AWS Secrets Manager) and inject at runtime.
2. **Parameterize the target month** – add an optional environment variable (e.g., `TARGET_MONTH=YYYY-MM`) that overrides the hard‑coded `add_months` logic, enabling back‑fills and late‑run corrections without code changes.