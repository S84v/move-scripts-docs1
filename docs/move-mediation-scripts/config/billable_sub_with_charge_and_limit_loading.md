# Summary
Properties file that defines environment‑specific parameters for the **Billable_Sub_With_Charge_And_Limit** loading pipeline. It sets logging paths, process‑status file location, Hive database/table identifiers, Java loader class names, Hive table refresh commands, and the script name invoked by the nightly Move‑Mediation job.

# Key Components
- `MNAAS_billable_sub_with_charge_and_limit_LogPath` – target log file (includes execution date).  
- `billable_sub_with_charge_and_limit_ProcessStatusFileName` – file used to record success/failure of the load.  
- `Dname_billable_sub_with_charge_and_limit` – Hive database name for this pipeline.  
- `billable_sub_with_charge_and_limit_temp_loading_classname` – fully‑qualified Java class for loading the temporary staging table.  
- `billable_sub_with_charge_and_limit_loading_classname` – fully‑qualified Java class for loading the final table.  
- `billable_sub_with_charge_and_limit_temp_tblname_refresh` / `billable_sub_with_charge_and_limit_tblname_refresh` – Hive `REFRESH` commands for the temp and final tables.  
- `MNAAS_billable_sub_with_charge_and_limit_loading_ScriptName` – shell script that orchestrates the end‑to‑end load.  
- `setparameter='set -x'` – optional Bash debug flag (comment to disable).  

# Data Flow
1. **Input**: Source data extracted via Sqoop/Impala (not defined here) and consumed by the Java loader classes.  
2. **Processing**:  
   - `billable_sub_with_charge_and_limit_temp_loading` writes to a temporary Hive table.  
   - `billable_sub_with_charge_and_limit_loading` moves data from temp to the final Hive table.  
   - After each load, the corresponding `REFRESH` command updates Hive metastore caches.  
3. **Outputs**:  
   - Final Hive table (`$billable_sub_with_charge_and_limit_tblname`).  
   - Log file (`$MNAAS_billable_sub_with_charge_and_limit_LogPath`).  
   - Process‑status file (`$billable_sub_with_charge_and_limit_ProcessStatusFileName`).  
4. **Side Effects**: Updates Hive metastore; creates/overwrites log and status files.  

# Integrations
- Sourced by `billable_sub_with_charge_and_limit.sh` which reads these properties and invokes the Java loader classes via Hadoop `jar` execution.  
- Relies on common properties from `MNAAS_CommonProperties.properties` (e.g., `$MNAASLocalLogPath`, `$dbname`).  
- Consumes Hive/Impala services for table creation, refresh, and data insertion.  
- May be triggered by the Move‑Mediation orchestration framework (e.g., Oozie or Airflow) as part of nightly batch.  

# Operational Risks
- **Stale log file**: Log path includes date; old logs may be overwritten if script runs multiple times per day. *Mitigation*: enforce single daily run or include timestamp.  
- **Process‑status file race**: Concurrent runs could corrupt status file. *Mitigation*: lock file or use unique run identifier.  
- **Missing refresh**: Failure to execute `REFRESH` leads to stale metadata and query errors. *Mitigation*: verify exit code of refresh commands.  
- **Debug flag leakage**: `set -x` may expose sensitive parameters in logs. *Mitigation*: comment out in production.  

# Usage
```bash
# Load common properties first
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Source this file
. move-mediation-scripts/config/billable_sub_with_charge_and_limit_loading.properties

# Run the orchestrating script (debug enabled)
set -x   # optional, controlled by setparameter
bash $MNAAS_billable_sub_with_charge_and_limit_loading_ScriptName
```
To debug, uncomment `setparameter='set -x'` or export `set -x` before invoking the script.

# Configuration
- **Environment variables** (inherited from common properties):  
  - `MNAASLocalLogPath` – base directory for logs.  
  - `dbname` – Hive database name used in refresh commands.  
- **Referenced files**:  
  - `MNAAS_CommonProperties.properties` (global defaults).  
  - Java JAR containing loader classes (path defined elsewhere).  

# Improvements
1. Replace hard‑coded refresh strings with a function that validates table existence before issuing `REFRESH`.  
2. Externalize the Java class names and JAR path into a separate version‑controlled properties file to simplify upgrades without modifying the loader script.