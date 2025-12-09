# Summary
`MNAAS_Telena_backup_files_process.properties` is a Bash‑sourced configuration fragment for the **Telena file‑backup step** of the MNAAS daily‑processing pipeline. It imports shared defaults, defines all runtime constants required by `MNAAS_Telena_backup_files_process.sh`, and maps file‑type patterns to source locations, HDFS staging paths, Hive tables, status‑tracking files, and log destinations. The file drives the discovery, classification, backup, and post‑processing of Telena‑derived CSV extracts (Activations, Tolling, SimInventory, TrafficDetails, Actives).

---

# Key Components
- **Sourcing**
  - `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` – pulls global defaults (e.g., `MNAASConfPath`, `MNAASLocalLogPath`, `dbname`).

- **Scalar Variables**
  - `MNAAS_backup_filename_telena` – absolute path to the master backup‑file list.
  - `MNAAS_backup_filename_telena_temp` – temporary working copy of the list.
  - `MNAAS_Telena_files_backup_ProcessStatusFile` – status‑file path for the job.
  - `MNAAS_backup_files_logpath_Telena` – log file (date‑suffixed) for the process.
  - `MNAAS_telena_backup_files_Scriptname` – script that consumes this config.
  - `MNAAS_backup_filename_hdfs_Telena` – HDFS target directory for Telena feeds.
  - `telena_file_record_count_inter_tblname` / `telena_file_record_count_tblname` – Hive/Impala tables used for intermediate and final record‑count tracking.
  - `telena_file_record_count_refresh` – Hive `REFRESH` command string.
  - `backup_server` – IP of the remote backup host.
  - `Daily_Telena_BackupDir` – local filesystem staging directory for successful backups.
  - `MNAAS_files_backup_dups_telena` – location for duplicate‑file detection.
  - `parent_backup_dir` – root of the backup hierarchy.

- **Associative Arrays**
  - `MNAAS_backup_files_process_file_pattern` – maps logical file‑type keys to glob patterns for source CSV discovery.
  - `MNAAS_backup_files_edgenode2_backup_dir` – maps file‑type keys to sub‑directory names under the edge‑node backup root.
  - `MNAAS_telena_type_location_mapping` – normalises lower‑case type identifiers to the canonical directory names used by the backup process.
  - `MNAAS_File_processed_location_dir` – resolves processing outcome (`NonEmpty`, `Empty`, `Reject`) to target directories (`$Daily_Telena_BackupDir/$file_type`, `$EmptyFileDir`, `$MNAASRejectedFilePath`).

---

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| **Configuration Load** | `MNAAS_CommonProperties.properties` | `source` → populates global vars (`MNAASConfPath`, `MNAASLocalLogPath`, `dbname`, etc.) | In‑memory Bash variables |
| **File Discovery** | Files on edge node matching patterns in `MNAAS_backup_files_process_file_pattern` | Loop in `MNAAS_Telena_backup_files_process.sh` expands globs, classifies by type | List of absolute file paths |
| **Classification** | Discovered file path, its type | Uses `MNAAS_telena_type_location_mapping` to map to logical type; checks size | Determines destination (`NonEmpty`, `Empty`, `Reject`) |
| **Staging** | Classified file | Copies/moves to `$Daily_Telena_BackupDir/$file_type` (or `$EmptyFileDir` / `$MNAASRejectedFilePath`) | Physical file placement |
| **HDFS Load** | Staged file | `hdfs dfs -put` → `$MNAAS_backup_filename_hdfs_Telena` | HDFS replica |
| **Hive/Impala Update** | Record count per file | Executes `telena_file_record_count_refresh` and INSERT/UPDATE statements (via `beeline`/`impala-shell`) | Updated table `telena_file_record_count` |
| **Status & Logging** | Job start/end, per‑file success/failure | Writes to `MNAAS_Telena_files_backup_ProcessStatusFile` and `$MNAAS_backup_files_logpath_Telena` | Persistent audit trail |
| **Remote Backup** | Optional rsync to `$backup_server:$Daily_Telena_BackupDir` (not shown but implied) | `rsync`/`scp` invoked by the script | Remote copy for DR |

External services touched:
- **HDFS** (`/user/MNAAS/FileCount/telena_feed_data`)
- **Hive/Impala** (`$dbname.telena_file_record_count*` tables)
- **Remote backup server** (`192.168.124.28`) via network file copy
- **Local filesystem** (multiple staging & archive directories)

---

# Integrations
- **`MNAAS_Telena_backup_files_process.sh`** – primary driver script; sources this properties file and implements the workflow described above.
- **`MNAAS_CommonProperties.properties`** – provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`, `EmptyFileDir`, `MNAASRejectedFilePath`, `dbname`, etc.).
- **Hive/Impala** – tables referenced by `telena_file_record_count_*` are refreshed/updated by the script.
- **HDFS CLI** – used to place CSV extracts into the data lake.
- **Backup Server** – reachable via SSH; the script may invoke `rsync`