# Summary
`MNAAS_Sqoop_Cbf_GNV_Invoice_generation.properties` is a Bash‑sourced configuration fragment for the **CBF GNV Invoice Generation** Sqoop step in the MNAAS daily‑processing pipeline. It supplies Oracle connection details, process‑status and log file locations, HDFS staging directory, target Hive table name, and the SQL query used by `MNAAS_Sqoop_Cbf_GNV_Invoice_generation.sh` to import invoice records from the `cbf.cbf_auto_move_details_mview` view into a staging Hive/Impala table.

# Key Components
- **Source statement** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` (imports shared constants).  
- **Oracle connection variables** – `OrgDetails_ServerName`, `OrgDetails_PortNumber`, `OrgDetails_Service`, `OrgDetails_Username`, `OrgDetails_Password`.  
- **Process‑status file** – `CBF_GNV_Invoice_generation_Sqoop_ProcessStatusFile`.  
- **Log file definition** – `CBF_GNV_Invoice_generation_table_SqoopLogName` (includes current date).  
- **Script name** – `MNAAS_Sqoop_CBF_GNV_Invoice_generation_Scriptname`.  
- **HDFS staging directory** – `CBF_GNV_Invoice_generation_table_Dir`.  
- **Target Hive table name** – `CBF_GNV_Invoice_generation_table_name`.  
- **Sqoop import query** – `CBF_GNV_Invoice_generation_table_Query` (SELECT … FROM `cbf.cbf_auto_move_details_mview` with `$CONDITIONS` placeholder).

# Data Flow
| Stage | Input | Transformation | Output |
|-------|-------|----------------|--------|
| **Sqoop import** | Oracle DB (`CBF_TRANS` service) via credentials above | Executes `CBF_GNV_Invoice_generation_table_Query` with `$CONDITIONS` supplied by Sqoop (splits, incremental bounds) | HDFS files under `$SqoopPath/CBF_GNV_Invoice_generation` |
| **Hive load** (performed by the calling script) | HDFS staging files | `LOAD DATA` / `INSERT OVERWRITE` into Hive table `CBF_GNV_Invoice_generation` | Hive/Impala table populated |
| **Logging** | Script execution context | Writes to `$MNAASLocalLogPath/CBF_GNV_Invoice_generation_Sqoop.log_YYYY-MM-DD` | Log file |
| **Process status** | Script start/end | Writes status (e.g., SUCCESS/FAIL) to `$MNAASConfPath/CBF_GNV_Invoice_generation_Sqoop_ProcessStatusFile` | Status file |

Side effects: network I/O to Oracle, HDFS writes, Hive metadata updates, creation of log and status files.

# Integrations
- **MNAAS_CommonProperties.properties** – provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`, `SqoopPath`, etc.).  
- **MNAAS_Sqoop_Cbf_GNV_Invoice_generation.sh** – consumes all variables defined herein to construct the Sqoop command and post‑processing steps.  
- **Hive/Impala** – target of the loaded data; downstream reporting jobs depend on the populated table.  
- **Scheduler (e.g., Oozie/Cron)** – triggers the shell script as part of the daily move‑mediation workflow.

# Operational Risks
- **Plain‑text credentials** – risk of credential leakage; mitigate by using a credential vault or OS‑level file permissions.  
- **Hard‑coded server IP/port** – changes require file edit and redeploy; mitigate with DNS alias or external config service.  
- **Log filename includes `date` at source definition time** – if the file is sourced once and reused, log may retain stale date; ensure the script re‑evaluates the variable at runtime.  
- **Missing `$CONDITIONS` handling** – improper Sqoop split may cause full table scan; validate that the calling script supplies appropriate `--split-by` and `--where` arguments.  
- **No validation of connection parameters** – script may fail silently; add pre‑flight connectivity checks.

# Usage
```bash
# Source the configuration (usually done inside the script)
source /app/hadoop_users/MNAAS/move-mediation-scripts/config/MNAAS_Sqoop_Cbf_GNV_Invoice_generation.properties

# Run the Sqoop import (example command constructed by the wrapper script)
sqoop import \
  --connect "jdbc:oracle:thin:@${OrgDetails_ServerName}:${OrgDetails_PortNumber}/${OrgDetails_Service}" \
  --username "${OrgDetails_Username}" \
  --password "${OrgDetails_Password}" \
  --query "${CBF_GNV_Invoice_generation_table_Query}" \
  --target-dir "${CBF_GNV_Invoice_generation_table_Dir}" \
  --as-avrodatafile \
  --split-by org_no \
  --num-mappers 4 \
  --verbose \
  2>&1 | tee "${CBF_GNV_Invoice_generation_table_SqoopLogName}"
```
To debug, set `set -x` in the wrapper script and verify that all variables resolve correctly.

# Configuration
- **Environment variables / paths** (provided by `MNAAS_CommonProperties.properties`):  
  - `MNAASConfPath` – base directory for status files.  
  - `MNAASLocalLogPath` – base directory for log files.  
  - `SqoopPath` – base HDFS staging root.  
- **Properties file itself** – must be readable by the executing user.  
- **Oracle JDBC driver** – must be on the classpath of the Sqoop command.

# Improvements
1. **Externalize secrets** – replace `OrgDetails_Username` and `OrgDetails_Password` with references to a secure credential store (e.g., Hadoop Credential Provider, HashiCorp Vault).  
2. **Parameterize date handling** – change `CBF_GNV_Invoice_generation_table_SqoopLogName` to compute the date at script execution time (`$(date +_%F)`) rather than at source time, ensuring each run generates a fresh log file.