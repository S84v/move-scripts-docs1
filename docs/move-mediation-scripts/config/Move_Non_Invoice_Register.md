# Summary
Defines Bash‑sourced runtime constants for the **Move_Non_Invoice_Register** job. The job stages non‑invoice register CSV files in HDFS, loads them into temporary Hive tables, refreshes the tables, and persists the data via the `mnaas_invoice.jar` Java loader. It also configures process‑status tracking, logging, backup locations, and driver script references.

# Key Components
- **Move_Non_Invoice_Register** – HDFS input directory for raw non‑invoice register files.  
- **Move_Non_Invoice_Register_ProcessStatusFileName** – HDFS file used to record job status (STARTED, SUCCESS, FAIL).  
- **Move_Non_Invoice_Register_LogPath** – Local filesystem path for job logs.  
- **Move_Non_Invoice_Register_BackupDir** – HDFS backup directory for processed files.  
- **MoveNonInvoiceRegister_dump_PathName** – HDFS staging area for intermediate dumps.  
- **Dname_MNAAS_Load_non_invoice_tbl_temp** – Hive database name for temporary load tables.  
- **Dname_Move_Non_Invoice_Register_load_files_into_main_table** – Hive database name for final load tables.  
- **SEBS_CHN_DTL_INVCE_REG_tblname** – Hive table name for the persistent non‑invoice register.  
- **SEBS_CHN_DTL_INVCE_REG_temp_tblname** – Hive temporary table name.  
- **SEBS_CHN_DTL_INVCE_REG_temp_tblname_refresh** – Hive `REFRESH` command for the temporary table.  
- **SEBS_CHN_DTL_INVCE_REG_tblname_refresh** – Hive `REFRESH` command for the persistent table.  
- **MNAAS_Main_JarPath** – HDFS path to the Java loader JAR (`mnaas_invoice.jar`).  
- **invoice_noninvoice_table_loaading** – Fully‑qualified Java class implementing the load logic.  
- **processname_invoice** – Logical name passed to the Java loader (`non_invoice_data`).  
- **MoveInvoiceRegisterScriptName** – Driver shell script (`Move_Non_Invoice_Register.sh`).

# Data Flow
1. **Input**: CSV files placed in `Move_Non_Invoice_Register` HDFS directory.  
2. **Staging**: Files copied to `MoveNonInvoiceRegister_dump_PathName`.  
3. **Hive Load**:  
   - Load into temporary table `SEBS_CHN_DTL_INVCE_REG_temp` (DB `Dname_MNAAS_Load_non_invoice_tbl_temp`).  
   - Execute `REFRESH` on temporary table.  
   - Insert into persistent table `SEBS_CHN_DTL_INVCE_REG` (DB `Dname_Move_Non_Invoice_Register_load_files_into_main_table`).  
   - Execute `REFRESH` on persistent table.  
4. **Java Loader**: `mnaas_invoice.jar` invoked with class `invoice_noninvoice_table_loaading` and process name `non_invoice_data` to persist data to downstream MySQL (or other target).  
5. **Backup**: Processed source files moved to `Move_Non_Invoice_Register_BackupDir`.  
6. **Status & Logging**: Job status written to `Move_Non_Invoice_Register_ProcessStatusFileName`; logs written to `Move_Non_Invoice_Register_LogPath`.

# Integrations
- **MNAAS_CommonProperties.properties** – Provides shared environment variables (`MNAASConfPath`, `MNAASLocalLogPath`, `MNAASPathName`, etc.).  
- **Hive** – Used for temporary and final table creation, refresh, and insert operations.  
- **Java Loader (`mnaas_invoice.jar`)** – Persists data to the target relational store (MySQL).  
- **Driver Script (`Move_Non_Invoice_Register.sh`)** – Orchestrates the above steps; invoked by scheduler (e.g., Oozie/Cron).  
- **Backup System** – HDFS backup directory for audit/recovery.

# Operational Risks
- **Missing Input Files** – Job will stall; mitigate with pre‑run file existence check and alert.  
- **Hive Table Schema Drift** – Incompatible schema causes load failures; enforce schema versioning and automated validation.  
- **Jar/Class Mismatch** – Incorrect class name leads to Java runtime errors; add sanity check before execution.  
- **Insufficient HDFS Space** – Staging and backup may fill storage; monitor disk usage and enforce retention policies.  
- **ProcessStatusFile Stale** – Stale status may mislead downstream jobs; implement TTL cleanup.

# Usage
```bash
# Source common properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Execute driver script (typically via scheduler)
bash /app/hadoop_users/Lavanya/MOVE_Invoice_Register/Move_Non_Invoice_Register.sh
```
For debugging, enable `set -x` in the driver script and inspect logs at `$Move_Non_Invoice_Register_LogPath`.

# Configuration
- **Environment Variables** (provided by `MNAAS_CommonProperties.properties`):  
  - `MNAASConfPath`  
  - `MNAASLocalLogPath`  
  - `MNAASPathName`  
- **File Paths** defined in this properties file (see Key Components).  
- **Java JAR**: `/app/hadoop_users/Lavanya/MOVE_Invoice_Register/mnaas_invoice.jar`  
- **Driver Script**: `Move_Non_Invoice_Register.sh`

# Improvements
1. **Parameterize Hive Database Names** – Move DB names to a dedicated config file to avoid hard‑coding across scripts.  
2. **Add Idempotent Status Handling** – Implement atomic write/rename for the process‑status file to prevent race conditions during concurrent runs.