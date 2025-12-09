**# Summary**  
The `Move_Invoice_Register.properties` file defines Bash‑sourced runtime constants for the **Move_Invoice_Register** job, which stages raw invoice‑register files in HDFS, loads them into temporary Hive tables, refreshes the tables, and finally persists the data via the `mnaas_invoice.jar` Java loader. It also configures process‑status tracking, logging, backup locations, and script/JAR references used by the driver `Move_Invoice_Register.sh`.

**# Key Components**  

- `Move_Invoice_Register` – HDFS directory containing raw invoice‑register input files.  
- `Move_Invoice_Register_ProcessStatusFileName` – HDFS file used to record job status (STARTED, SUCCESS, FAIL).  
- `Move_Invoice_Register_LogPath` – Local filesystem path for job log files.  
- `Move_Invoice_Register_BackupDir` – HDFS backup directory for processed files.  
- `MoveInvoiceRegister_dump_PathName` – HDFS staging area for intermediate dump files.  
- Hive database/table names:  
  - `Dname_MNAAS_Load_invoice_tbl_temp` – temporary Hive DB for loading.  
  - `Dname_Move_invoice_register_load_files_into_main_table` – DB containing the final load script.  
  - `SEBS_MOB_DTL_INVCE_REG_tblname` – target Hive table.  
  - `SEBS_MOB_DTL_INVCE_REG_temp_tblname` – temporary Hive table.  
  - `SEBS_MOB_DTL_INVCE_REG_temp_tblname_refresh` / `SEBS_MOB_DTL_INVCE_REG_tblname_refresh` – Hive `REFRESH` commands for metadata sync.  
- `MNAAS_Main_JarPath` – HDFS/local path to the Java JAR that performs the final load.  
- `invoice_noninvoice_table_loaading` – Fully‑qualified Java class implementing the load logic.  
- `processname_invoice` – Logical name used for monitoring/metrics.  
- `MoveInvoiceRegisterScriptName` – Bash driver script invoked by the scheduler.

**# Data Flow**  

| Stage | Input | Transformation | Output | Side Effects |
|-------|-------|----------------|--------|--------------|
| 1. Ingestion | Files in `/Input_Data_Staging/Move_Invoice_Register/SEBS_CHN_DTL_INVCE_REG` | Copied to HDFS staging (`MoveInvoiceRegister_dump_PathName`) | Raw files in HDFS | Status file set to **STARTED** |
| 2. Hive Load | Raw files | `Load` into `SEBS_MOB_DTL_INVCE_REG_temp_tblname` (temp DB) | Temporary Hive table populated | Refresh metadata (`SEBS_MOB_DTL_INVCE_REG_temp_tblname_refresh`) |
| 3. Refresh & Merge | Temp table | `INSERT/OVERWRITE` into `SEBS_MOB_DTL_INVCE_REG_tblname` (main DB) | Main Hive table updated | Refresh metadata (`SEBS_MOB_DTL_INVCE_REG_tblname_refresh`) |
| 4. Java Persist | Main Hive table data | `invoice_noninvoice_table_loaading` class in `mnaas_invoice.jar` writes to downstream MySQL VAZ/Invoice DB | Persistent relational rows | Backup raw files to `Move_Invoice_Register_BackupDir` |
| 5. Completion | – | Update status file to **SUCCESS** or **FAIL** | Log entry in `Move_Invoice_Register_LogPath` | None |

External services: HDFS, Hive Metastore, MySQL (target DB), YARN (job execution), Scheduler (e.g., Oozie/Airflow).

**# Integrations**  

- Sources common properties from `MNAAS_CommonProperties.properties`.  
- Invoked by the driver script `Move_Invoice_Register.sh`, which orchestrates the steps above.  
- Shares Hive DB names with other MNAAS jobs (e.g., usage‑trend, VAZ ingestion) for consistent partitioning.  
- Uses the same status‑file convention as other jobs (`*_ProcessStatusFileName`).  
- Java loader class may be reused by other invoice‑related pipelines.

**# Operational Risks**  

- **Stale status file** – If a previous run crashes, the status may remain **STARTED**, causing duplicate processing. *Mitigation*: Idempotent checks, cleanup step before start.  
- **Hive table refresh failure** – Missing metadata can lead to empty reads downstream. *Mitigation*: Verify refresh command exit code; retry on failure.  
- **Jar/class version mismatch** – Deploying a new JAR without updating the class name leads to `ClassNotFoundException`. *Mitigation*: CI pipeline validates JAR checksum and class existence.  
- **Backup directory saturation** – Unlimited accumulation in `/backup1/MNAAS/Customer_CDRS/InvoiceRegister`. *Mitigation*: Retention policy (e.g., 30 days) via cron cleanup.  
- **Insufficient HDFS permissions** – Job may fail to write to staging or backup paths. *Mitigation*: Pre‑run ACL verification script.

**# Usage**  

```bash
# Load common properties
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Export the job‑specific properties
source /path/to/Move_Invoice_Register.properties

# Execute driver (normally via scheduler)
bash $MNAASScriptPath/Move_Invoice_Register.sh
```

For debugging, set `set -x` in the driver script and inspect the status file at `$Move_Invoice_Register_ProcessStatusFileName`.

**# Configuration**  

- **Environment variables** (inherited from `MNAAS_CommonProperties.properties`):  
  - `MNAASConfPath`, `MNAASLocalLogPath`, `MNAASPathName`, `MNAASScriptPath`, `dbname`.  
- **Referenced config files**: `MNAAS_CommonProperties.properties`.  
- **Hard‑coded paths**:  
  - Input HDFS: `/Input_Data_Staging/Move_Invoice_Register/SEBS_CHN_DTL_INVCE_REG`  
  - Backup HDFS: `/backup1/MNAAS/Customer_CDRS/InvoiceRegister`  
  - Dump/staging: `$MNAASPathName/MoveInvoiceRegister_dump`  
  - JAR: `/app/hadoop_users/Lavanya/MOVE_Invoice_Register/mnaas_invoice.jar`

**# Improvements**  

1. **Externalize paths to a central YAML/JSON** – Allows environment‑specific overrides without editing the `.properties` file.  
2. **Add automated cleanup of the backup directory** – Implement a scheduled HDFS purge based on file age to prevent storage exhaustion.