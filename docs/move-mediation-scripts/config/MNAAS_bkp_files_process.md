# Summary
`MNAAS_bkp_files_process.properties` is a shell‑sourced configuration file that centralises environment‑specific constants for the MNAAS backup‑file‑processing jobs. It loads shared properties, optionally enables Bash debugging, and defines paths, filenames, server addresses, and script names used by the edge‑node backup processing pipelines (`MNAAS_edgenode2_backup_files_process.sh`, `MNAAS_edgenode1_bkp_files_process.sh`, and the Python driver `MNAAS_backup_table.py`). The constants drive file‑pruning, archiving, and status‑tracking for table‑level backups in the Move‑Mediation production environment.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – sources common global constants (e.g., `MNAASConfPath`, `MNAASLocalLogPath`).
- **`setparameter='set -x'`** – Bash debugging toggle; when exported, scripts run with trace output.
- **Status & Log Paths**
  - `MNAAS_files_backup_ProcessStatusFile` – status file location.
  - `MNAAS_backup_files_logpath` – per‑table process log (date‑suffixed).
  - `MNAAS_backup_files_logpath_common` – common process log (date‑suffixed).
- **Backup Server**
  - `backup_server=192.168.124.28` – target edge‑node for file transfer.
- **HDFS / Storage Paths**
  - `MNAAS_Rawtablesload_PathName` – HDFS source directory for raw table backups.
  - `MNAAS_File_processed_location_ZIP` – local archive root for zipped backups.
  - `MNAAS_File_processed_location_enode1` – local archive root for edge‑node‑1.
- **Script References**
  - `MNAAS_backup_enode2_files_Scriptname` – edge‑node‑2 processing script.
  - `MNAAS_backup_enode1_files_Scriptname` – edge‑node‑1 processing script.
  - `MNAAS_Script_name` – Python driver invoked by cron.
  - `MNAAS_BackupScript_Enode2` – absolute path to edge‑node‑2 script.
- **Metadata**
  - `MNAAS_Table_List` – file containing list of tables to process.
  - `MNAAS_Table_Name` – placeholder populated at runtime for each table.

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| **Cron Trigger** | `MNAAS_Table_List` (list of tables) | Loop over each `table_name` | Calls Python driver (`MNAAS_backup_table.py`) | Writes to status file |
| **Python Driver** | `MNAAS_Rawtablesload_PathName` (HDFS) | Copies raw backup files to local staging | Files staged under `MNAAS_File_processed_location_*` | HDFS read, local FS write |
| **Edge‑Node Script (Enode2)** | Staged files, `backup_server` | Compresses, transfers to edge‑node‑2 | Archived ZIPs in `MNAAS_File_processed_location_ZIP` | Network I/O, remote SSH |
| **Logging** | Variables above | Writes timestamped logs to `MNAAS_backup_files_logpath*` | Log files on local log directory | Disk I/O |
| **Status Update** | `MNAAS_files_backup_ProcessStatusFile` | Append success/failure per table | Updated status file | Disk I/O |

# Integrations
- **Common Properties** – `MNAAS_CommonProperties.properties` provides base paths, user credentials, and environment flags.
- **Edge‑Node Scripts** – `MNAAS_edgenode2_backup_files_process.sh` and `MNAAS_edgenode1_bkp_files_process.sh` are invoked directly by the Python driver or cron.
- **Hadoop/HDFS** – Reads raw backup tables from `/user/MNAAS/table_bkp/${table_name}/`.
- **Remote Backup Server** – SSH/SCP to `192.168.124.28` for final storage.
- **Cron Scheduler** – Typically scheduled via `/etc/cron.d` or user crontab to run the Python driver.

# Operational Risks
- **Hard‑coded IP** – `backup_server` may become stale; use DNS or config management.
- **Debug Toggle Leakage** – Leaving `setparameter='set -x'` enabled in production floods logs; ensure it is disabled or controlled via env flag.
- **Path Inconsistencies** – Date suffix in log paths (`$(date +_%F)`) evaluated at source time; if scripts source the file multiple times, log filenames may diverge.
- **Missing Table List** – Empty or malformed `table_name.txt` aborts the pipeline; add validation step.
- **Network Failure** – Transfer to edge‑node may hang; implement timeout/retry logic.

# Usage
```bash
# Source configuration
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_bkp_files_process.properties

# Enable debugging (optional)
export setparameter='set -x'

# Run the Python driver for a specific table (example)
TABLE=customer_orders
export table_name=$TABLE
$MNAAS_Script_name   # invokes MNAAS_backup_table.py with env vars
```
For full pipeline execution, schedule the driver via cron:
```cron
0 2 * * *   hdfs_user  source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_bkp_files_process.properties && \
            /app/hadoop_users/MNAAS/MNAAS_CronFiles/MNAAS_backup_table.py >> $MNAAS_backup_files_logpath 2>&1
```

# Configuration
- **Sourced Files**
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Environment Variables (populated at runtime)**
  - `table_name` – current table being processed.
- **Constants Defined in this File**
  - `setparameter`
  - `MNAAS_files_backup_ProcessStatusFile`
  - `MNAAS_backup_files_logpath`
  - `MNAAS_backup_files_logpath_common`
  - `backup_server`
  - `MNAAS_Rawtablesload_PathName`
  - `MNAAS_File_processed_location_ZIP`
  - `MNAAS_backup_enode2_files_Scriptname`
  - `MNAAS_backup_enode1_files_Scriptname`
  - `MNAAS_File_processed_location_enode1