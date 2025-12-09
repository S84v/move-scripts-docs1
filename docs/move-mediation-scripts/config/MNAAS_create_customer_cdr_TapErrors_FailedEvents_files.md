# Summary
Defines Bash associative arrays, scalar variables, and file‑pattern mappings required by `MNAAS_create_customer_cdr_TapErrors_FailedEvents_files.sh`. It loads common MNAAS properties, configures per‑customer output/backup directories, column‑position placeholders, and AWK filter conditions for processing TAP‑Error and Failed‑Event CDR feeds in the Move‑Network‑As‑A‑Service production pipeline.

# Key Components
- **Source inclusion** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
- **Debug toggle** – `setparameter='set -x'` (comment to enable Bash tracing)  
- **Script metadata** – `MNAAS_Create_Customer_cdr_TapErrors_FailedEvents_scriptname`  
- **Process‑status & log paths** – `*_ProcessStatusFilename`, `*_LogPath`  
- **Backup directory variable** – `backup_zip_dir` (points to `$BackupDir`)  
- **File‑type extensions** – `Tap_Errors_extn`, `Failed_Events_extn`  
- **Customer‑to‑feed mapping** – `declare -A FEED_CUSTOMER_MAPPING_TAPERRORS`  
- **Column‑position placeholder** – `declare -A MNAAS_Create_Customer_taperrors_files_column_position` (currently empty for “MyRep”)  
- **Output directory mapping** – `declare -A MNAAS_Create_Customer_taperrors_files_out_dir`  
- **Backup directory mapping** – `declare -A MNAAS_Create_Customer_taperrors_files_backup_dir`  
- **AWK filter condition mapping** – `declare -A MNAAS_Create_Customer_taperrors_files_filter_condition`  
- **Failed‑Events static output/backup dirs** – `MNAAS_Create_Customer_failedevents_files_out_dir`, `*_backup_dir`  
- **Feed‑file pattern mappings** – `declare -A FEED_FILE_PATTERN_TAPERRORS`, `declare -A FEED_FILE_PATTERN_FAILEDEVENTS`

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| 1 | `MNAAS_CommonProperties.properties` (global vars) | Source into current shell | Populates env vars (`MNAASConfPath`, `MNAASLocalLogPath`, `BackupDir`, `MNAAS_dtail_extn`, `MNAAS_Customer_SECS`) | None |
| 2 | Customer list (implicit via mappings) | Populate associative arrays for mapping, directories, filters | In‑memory Bash arrays/variables | None |
| 3 | Feed files matching `${MNAAS_dtail_extn['TAPErrors']}` & `${MNAAS_dtail_extn['failedevents']}` in source ingest locations | Consumed by `MNAAS_create_customer_cdr_TapErrors_FailedEvents_files.sh` (AWK filtering, column selection) | Per‑customer TAP‑Error CSV files, Failed‑Event CDR files in designated output dirs | Files written to `$MNAAS_Create_Customer_taperrors_files_out_dir[...]` and backup dirs; status file updated; log appended |
| 4 | Optional debug flag (`setparameter`) | Enables Bash `set -x` tracing when uncommented | Verbose execution trace in log | Increased I/O, potential exposure of sensitive values |

# Integrations
- **Common property repository** – `MNAAS_CommonProperties.properties` supplies base paths, customer SEC identifiers, and extension maps (`MNAAS_dtail_extn`).  
- **Main processing script** – `MNAAS_create_customer_cdr_TapErrors_FailedEvents_files.sh` sources this file to obtain configuration.  
- **Hadoop / HDFS** – Implicitly referenced by `$MNAASLocalLogPath` and `$BackupDir` which are typically HDFS locations in the production environment.  
- **External monitoring** – Process‑status file (`*_ProcessStatusFilename`) consumed by orchestration/monitoring tools to track job health.  

# Operational Risks
- **Missing customer mapping** – If a new customer is not added to `FEED_CUSTOMER_MAPPING_TAPERRORS`, files will be skipped silently. *Mitigation*: Validate mappings at script start; fail fast on unknown customers.  
- **Empty column‑position definition** – `MNAAS_Create_Customer_taperrors_files_column_position["MyRep"]` is empty; downstream AWK may produce malformed output. *Mitigation*: Enforce non‑empty definition or provide default column list.  
- **Hard‑coded paths** – Changes to `$BackupDir` or `$MNAASLocalLogPath` require file edit and redeployment. *Mitigation*: Externalize via environment variables or a dedicated config service.  
- **Debug flag leakage** – Accidentally enabling `set -x` in production can flood logs with sensitive data. *Mitigation*: Guard with an explicit `DEBUG` env var and document usage.  

# Usage
```bash
# 1. Ensure common properties are up‑to‑date
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# 2. Source this configuration (optional: enable debug)
#   Uncomment the line below to turn on Bash tracing
# setparameter='set -x'
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_create_customer_cdr_TapErrors_FailedEvents_files.properties

# 3. Execute the processing script
bash /app/hadoop_users/MNAAS/MNAAS_create_customer_cdr_TapErrors_FailedEvents_files.sh
```
To debug a single customer, export `CUSTOMER=MyRep` before invoking the script (if the script respects the variable).

# Configuration
- **Environment variables** (populated by common properties):  
  - `MNAASConfPath` – directory for process‑status files  
  - `MNAASLocalLogPath` – base log directory  
  - `BackupDir` – root backup location  
  - `MNAAS_dtail_extn` – associative array of file‑type extensions (`['TAPErrors']`, `['failedevents']`)  
 