# Summary
Defines Bash scalar variables and file‑pattern mappings for the **TapErrors/FailedEvents** CDR generation pipeline. It sources common MNAAS properties, sets script identifiers, log/status file locations, backup directory, and filename extensions used by `MNAAS_create_customer_cdr_TapErrors_FailedEvents_files.sh` to process TAP‑Error and Failed‑Event feeds per customer.

# Key Components
- **Source inclusion** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
- **Script identifier** – `MNAAS_Create_Customer_cdr_TapErrors_FailedEvents_scriptname`  
- **Process status file** – `MNAAS_Create_Customer_cdr_TapErrors_FailedEvents_ProcessStatusFilename`  
- **Log file path** – `MNAAS_Create_Customer_cdr_TapErrors_FailedEvents_LogPath` (includes current date)  
- **Backup directory variable** – `backup_zip_dir` (points to `$BackupDir`)  
- **Feed‑file extensions** – `Tap_Errors_extn=*TAPErrors*.csv` and `Failed_Events_extn=*FailedEvents*.cdr`

# Data Flow
| Stage | Input | Processing | Output / Side Effect |
|-------|-------|------------|----------------------|
| 1. Load common config | `MNAAS_CommonProperties.properties` | Bash `source` imports global variables (e.g., `$MNAASConfPath`, `$MNAASLocalLogPath`, `$BackupDir`) | Populates environment for downstream script |
| 2. Define script metadata | Hard‑coded literals | Assigns script name, status file, log file (date‑stamped) | Variables consumed by the main processing script |
| 3. Set backup location | `$BackupDir` | Direct assignment to `backup_zip_dir` | Used by the main script to archive processed files |
| 4. Define feed patterns | None (static glob patterns) | Assigns glob strings for TAP Errors and Failed Events | Main script expands patterns to locate source CDR files |

# Integrations
- **`MNAAS_CommonProperties.properties`** – provides base paths and shared configuration.  
- **`MNAAS_create_customer_cdr_TapErrors_FailedEvents_files.sh`** – sources this file to obtain all required variables before execution.  
- **Log aggregation** – writes to `$MNAASLocalLogPath/...log_YYYY-MM-DD`.  
- **Process status tracking** – writes status to `$MNAASConfPath/...ProcessStatusFile`.  
- **Backup subsystem** – archives processed files to `$BackupDir` via `backup_zip_dir`.

# Operational Risks
- **Missing common properties file** → script fails to resolve base paths. *Mitigation*: verify existence and readability before sourcing.  
- **Incorrect `$BackupDir`** → backup archive not created, leading to data loss. *Mitigation*: add runtime check for directory write permission.  
- **Date format in log filename** may produce illegal characters on non‑GNU `date`. *Mitigation*: enforce POSIX‑compatible date command or fallback.  
- **Glob pattern mismatch** → no TAPErrors/FailedEvents files found, causing empty output. *Mitigation*: log warning and exit with non‑zero status if pattern expands to zero files.

# Usage
```bash
# Source the property file (usually done inside the main script)
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
. move-mediation-scripts/config/MNAAS_create_customer_cdr_TapErrors_FailedEvents_files_backup.properties

# Run the processing script
bash /app/hadoop_users/MNAAS/MNAAS_create_customer_cdr_TapErrors_FailedEvents_files.sh
```
*Debug*: `set -x` before sourcing to echo variable assignments; `echo $Tap_Errors_extn $Failed_Events_extn` to verify patterns.

# Configuration
- **Environment variables / paths defined in `MNAAS_CommonProperties.properties`**  
  - `MNAASConfPath` – directory for process‑status files.  
  - `MNAASLocalLogPath` – base log directory.  
  - `BackupDir` – root backup location.  
- **Local variables defined in this file** (see *Key Components*).  

# Improvements
1. **Add validation block** – after sourcing, verify that `$MNAASConfPath`, `$MNAASLocalLogPath`, and `$BackupDir` exist and are writable; abort with clear error if not.  
2. **Parameterize date format** – expose a variable (e.g., `LOG_DATE_FMT='%Y-%m-%d'`) to allow consistent formatting across environments and simplify unit testing.