# Summary
Defines environment‑specific parameters for the **Customers_With_Overage_Flag** nightly Move‑Mediation ingestion job. Supplies log path, process‑status file, Hive database/table identifiers, Java loader class names, Hive refresh statements, and the shell script to invoke. Used by the Hadoop/Hive pipeline that flags customers with usage overage.

# Key Components
- `MNAAS_customers_with_overage_flag_LogPath` – absolute path for the execution log (date‑stamped).  
- `customers_with_overage_flag_ProcessStatusFileName` – file used by the orchestrator to record job status.  
- `Dname_customers_with_overage_flag` – logical name of the Hive database (passed to Hive scripts).  
- `customers_with_overage_flag_temp_classname` – fully‑qualified Java class for the temporary staging table load.  
- `customers_with_overage_flag_classname` – fully‑qualified Java class for the final table load.  
- `customers_with_overage_flag_temp_tblname` / `customers_with_overage_flag_tblname` – Hive table names for staging and production.  
- `customers_with_overage_flag_temp_tblname_refresh` / `customers_with_overage_flag_tblname_refresh` – Hive `REFRESH` commands to invalidate metadata caches after load.  
- `MNAAS_customers_with_overage_flag_ScriptName` – shell script (`customers_with_overage_flag.sh`) that drives the end‑to‑end process.  
- `setparameter='set -x'` – optional Bash debugging flag (commented out to disable).  

# Data Flow
1. **Input**: Source data extracted by Java loader classes (`*_temp_classname`, `*_classname`) from Oracle/other RDBMS.  
2. **Processing**:  
   - Data written to Hive staging table (`*_temp_tblname`).  
   - `REFRESH` executed on staging table.  
   - Transformation logic (within Java classes) applied.  
   - Result written to production Hive table (`*_tblname`).  
   - `REFRESH` executed on production table.  
3. **Output**: Populated Hive tables used by downstream reporting/balance‑update jobs.  
4. **Side Effects**:  
   - Log file appended at `MNAAS_customers_with_overage_flag_LogPath`.  
   - Process status file updated (`customers_with_overage_flag_ProcessStatusFileName`).  
5. **External Services**: Hive metastore, Oracle (or source DB), Hadoop HDFS for logs, Linux shell environment.

# Integrations
- Sourced by `customers_with_overage_flag.sh` which reads these properties and invokes the Java loader JARs.  
- Shares common properties via `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`.  
- Integrated with the generic Move‑Mediation framework that schedules nightly jobs, monitors `*_ProcessStatusFileName`, and aggregates logs.  
- Hive refresh commands rely on `$dbname` variable defined in the common properties file.

# Operational Risks
- **Log path overflow**: Unbounded log growth may fill local disk. *Mitigation*: Log rotation or size‑based cleanup.  
- **Stale Hive metadata**: Failure to execute `REFRESH` leads to query inconsistencies. *Mitigation*: Enforce exit‑code check after each `REFRESH`.  
- **Process status file race**: Concurrent runs could overwrite status. *Mitigation*: Use job‑level locking or unique status filenames per run timestamp.  
- **Debug flag leakage**: Leaving `setparameter='set -x'` enabled may expose sensitive data in logs. *Mitigation*: Ensure it is commented out in production.

# Usage
```bash
# Load common properties first
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Source this file
source /app/hadoop_users/MNAAS/MNAAS_Configuration_Files/customers_with_overage_flag.properties

# Execute the driver script
bash $MNAAS_customers_with_overage_flag_ScriptName   # typically invoked by cron or Oozie
```
To debug, uncomment `setparameter='set -x'` before sourcing the file.

# Configuration
- **Referenced files**: `MNAAS_CommonProperties.properties` (defines `$MNAASLocalLogPath`, `$dbname`, etc.).  
- **Environment variables**: Must be exported by the common properties file (`MNAASLocalLogPath`, `dbname`).  
- **Properties defined herein**: All listed under *Key Components*.  

# Improvements
1. **Parameterize log rotation** – add `MNAAS_customers_with_overage_flag_LogRetentionDays` and implement automated cleanup in the driver script.  
2. **Externalize Hive refresh commands** – move `*_tblname_refresh` strings to a dedicated Hive‑DDL properties file to simplify updates when database naming conventions change.