# Summary  
Defines all runtime constants for the **MNAAS_Daily_Recon** batch job. The properties are sourced by the `MNAAS_Daily_Recon.sh` script and drive file‑system locations, status‑file handling, logging, staging/backup directories, file‑name patterns, Hive table reference, Java JAR execution, and CSV export settings used in the daily reconciliation reporting pipeline of the telecom Move production environment.

# Key Components  
- **Property imports** – sources `MNAAS_CommonProperties.properties` and `MNAAS_Java_Batch.properties` for base paths and Java batch defaults.  
- **Debug toggle** – `setparameter='set -x'` (commented to disable).  
- **Process‑status file** – `MNAAS_Daily_Recon_ProcessStatusFileName`.  
- **Shell script name** – `MNAAS_Daily_Recon_ScriptName`.  
- **Log path** – `MNAAS_DailyRecon_LogPath` (includes date suffix).  
- **Staging & backup directories** – `MNAASReconStagingDir`, `MNAASReconBackupDir`.  
- **File‑name glob patterns** – `recon_file_format`, `hol_recon_file_format`, `sng_recon_file_format`.  
- **Hive table identifier** – `Dname_MNAAS_Daily_Recon_Load`.  
- **Java JAR & class** – `MNAAS_Recon_JarPath`, `Export_File_To_CSV`.  
- **CSV export location** – `MNAAS_output_csv_path`, `MNAAS_output_csv_file`.  
- **Java logging** – `MNAAS_Java_Log_File`.  
- **HDFS report base** – `MNAAS_hdfs_report_path`.  
- **Process identifiers** – `load_process_name`, `recon_process_name`.

# Data Flow  
| Stage | Input | Transformation | Output | Side Effects |
|-------|-------|----------------|--------|--------------|
| **1. Init** | `MNAAS_CommonProperties.properties`, `MNAAS_Java_Batch.properties` | Variable substitution | Populated environment for script | None |
| **2. File discovery** | Files in `${MNAASReconStagingDir}` matching the three glob patterns | Selection of relevant reconciliation Excel files | List of files passed to Java job | None |
| **3. Java processing** | Selected Excel files, `DailyRecon.jar`, class `Export_File_To_CSV` | Parse Excel → generate CSV (`recordsToLoad.csv`) | CSV in `${MNAAS_output_csv_path}` | Writes to Java log `${MNAAS_Java_Log_File}` |
| **4. Hive load** (implicit via `Dname_MNAAS_Daily_Recon_Load`) | Generated CSV | Hive `LOAD DATA` into table `MNAAS_Daily_Recon_Load` | Table populated | HDFS write to `${MNAAS_hdfs_report_path}` |
| **5. Archival** | Processed Excel files | Move to `${MNAASReconBackupDir}` | Backup retained | File system state change |
| **6. Status & logging** | Process status file path | Write success/failure flag | Updated `${MNAAS_Daily_Recon_ProcessStatusFileName}` | Log entry in `${MNAAS_DailyRecon_LogPath}` |

# Integrations  
- **Common property library** – provides base directories (`MNAASConfPath`, `MNAASLocalLogPath`, etc.).  
- **Java batch framework** – `MNAAS_Java_Batch.properties` supplies JVM options, classpath, and generic Java logging conventions.  
- **Hive** – target table `MNAAS_Daily_Recon_Load` is referenced for downstream analytics.  
- **HDFS** – `${MNAAS_hdfs_report_path}` is the root for report storage accessed by downstream consumers.  
- **Cron scheduler** – the script is invoked by a daily cron entry (not shown) that sources this properties file.  
- **Email/notification** – implied by `recon_process_name` (likely triggers mail step in downstream script).  

# Operational Risks  
- **Hard‑coded date in log filename** – may create duplicate log files if script runs multiple times per day. *Mitigation*: use timestamp with seconds or rotate logs.  
- **Missing or malformed property imports** – failure to locate common property files aborts the job. *Mitigation*: validate existence before sourcing.  
- **Glob pattern mismatches** – if source files do not conform to expected naming, job may process zero files silently. *Mitigation*: add pre‑run validation and alert on empty file set.  
- **File permission issues** on staging, backup, or HDFS paths can cause job failure. *Mitigation*: enforce ACLs and monitor via the process‑status file.  
- **Debug flag left enabled** (`set -x`) can flood logs and expose sensitive paths. *Mitigation*: enforce comment before production deployment.  

# Usage  
```bash
# Source the properties (required by the shell script)
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_Recon.properties

# Optional: enable Bash tracing for debugging
# set -x   # or uncomment the setparameter line in the properties file

# Execute the batch script
bash $MNAAS_Daily_Recon_ScriptName
```
To debug, uncomment the `setparameter='set -x'` line, re‑source the file, and re‑run the script; trace output will appear in the console and in `${MNAAS_DailyRecon_LogPath}`.

# Configuration  
- **External property files**  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Java_Batch.properties`  
- **Environment variables** (populated by the imported files)  
  - `MNAASConfPath`, `MNAASLocalLogPath`, `MNAASJava_Log_File`, etc.  
- **Hard‑coded paths** (may be overridden in common properties)  
  - `/Input_Data_Staging/MNAAS_DailyFiles_Reporting` (staging)  
  - `/backup1/MNAAS/reporting/movenl_ra_recon_report/` (