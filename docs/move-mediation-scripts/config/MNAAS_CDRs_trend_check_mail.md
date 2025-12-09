# Summary
`MNAAS_CDRs_trend_check_mail.properties` is a Bash‑sourced configuration file that supplies environment‑specific constants, file‑name patterns, and directory mappings for the MNAAS backup‑file‑processing pipeline (edge‑node 2). It centralises paths, customer‑to‑feed mappings, and traffic‑file locations used by downstream scripts that ingest, validate, and archive CDR backup files.

# Key Components
- **Sourced common properties** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
- **Static file paths** – `MNAAS_backup_filename_telena`, `MNAAS_backup_filename_telena_temp`, `Daily_Telena_BackupDir`, `MNAAS_edge2_backup_path`, `edgenode2`, `MNAAS_Create_New_Customers`
- **Associative arrays**  
  - `MNAAS_backup_files_process_file_pattern` – maps logical feed names to glob patterns for source CSV files.  
  - `FEED_CUSTOMER_MAPPING` – maps feed identifiers to one‑or‑more customer codes.  
  - `MNAAS_backup_files_edgenode2_backup_dir` – maps feed names to sub‑directory names on the edge node.  
  - `MNAAS_telena_type_location_mapping` – maps Telena type strings to feed names.  
  - `MNAAS_edge2_customer_backup_dir` – maps customer codes to their dedicated backup sub‑directories.  
  - `MNAAS_Traffic_File_Paths` – indexed list of HDFS staging directories for daily traffic files.
- **Commented‑out entries** – placeholders for future customers (e.g., SKYROAM, uCloudlink).

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| **File discovery** | Raw backup files under `$Daily_Telena_BackupDir` and `$MNAAS_edge2_backup_path` | Glob patterns from `MNAAS_backup_files_process_file_pattern` filter files per feed | Lists of matching CSV files |
| **Customer routing** | Feed name + file list | `FEED_CUSTOMER_MAPPING` resolves target customers; `MNAAS_edge2_customer_backup_dir` resolves destination sub‑dir | Files copied/moved to `$MNAAS_edge2_backup_path/<customer_dir>/` |
| **Telena type handling** | Telena type identifier (e.g., `activations`) | `MNAAS_telena_type_location_mapping` maps to feed name | Correct feed pattern applied |
| **Traffic staging** | Daily traffic files in HDFS | `MNAAS_Traffic_File_Paths` provides candidate staging directories for Spark/Hadoop jobs | Files read by downstream aggregation jobs |
| **Status & logging** | Not defined here – consumed by calling scripts | Scripts generate status files and dated logs using paths derived from common properties | Persistent audit trail |

External services touched indirectly:
- HDFS (via `$MNAAS_Traffic_File_Paths`)
- Local filesystem (backup directories)
- Potentially remote edge node (`edgenode2`) via NFS/SMB mount.

# Integrations
- **`MNAAS_edgenode2_backup_files_process.sh`** – sources this file to obtain patterns, mappings, and paths for its processing loop.  
- **`MNAAS_backup_table.py`** – Python driver reads the same environment variables (exported by the sourcing script) to load backup metadata into Hive/Impala tables.  
- **Common property files** – `MNAAS_CommonProperties.properties` provides base variables (`$MNAASConfPath`, `$MNAASLocalLogPath`, etc.).  
- **`MNAAS_Create_New_Customers.properties`** – used to augment customer‑specific logic (e.g., dynamic directory creation).  

# Operational Risks
- **Missing/incorrect mapping** – a feed not present in `FEED_CUSTOMER_MAPPING` results in files being dropped. *Mitigation*: validation script that checks completeness against a master customer list.  
- **Path divergence** – hard‑coded absolute paths may become stale after storage re‑architecture. *Mitigation*: centralise all base paths in `MNAAS_CommonProperties.properties` and reference them.  
- **Glob pattern collisions** – overlapping patterns could cause duplicate processing. *Mitigation*: enforce unique prefixes in `MNAAS_backup_files_process_file_pattern`.  
- **Debug toggle absent** – no built‑in Bash `set -x` control; debugging requires manual edit. *Mitigation*: add `MNAAS_DEBUG` flag and conditional `set -x`.  

# Usage
```bash
# Load configuration in the processing script
#!/usr/bin/env bash
set -euo pipefail

# Optional: enable debug
# export MNAAS_DEBUG=1

source /app/hadoop_users/MNAAS/move-mediation-scripts/config/MNAAS_CDRs_trend_check_mail.properties

# Example: list all Activation files for all customers
for pattern in "${MNAAS_backup_files_process_file_pattern[Activations]}"; do
    find "$Daily_Telena_BackupDir" -type f -name "$pattern"
done
```
To debug, run the calling script with `bash -x script.sh` after ensuring the file is sourced.

# Configuration
- **Environment variables** (populated by common properties):  
  - `MNAASConfPath` – root config directory.  
  - `MNAASLocalLogPath` – base log directory.  
- **External config files**:  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Create_New_Customers.properties`  
- **Constants defined in this file**: all associative arrays and path variables listed under *Key Components*.

# Improvements
1. **Externalise mappings** – move associative arrays to a JSON/YAML file and load via `jq` or `yq` to simplify updates and enable schema validation.  
2. **Add integrity checks** – implement a startup validation routine that verifies that every key in `MNAAS_backup_files_process_file_pattern` has a corresponding entry in `MNAAS_backup_files_edgenode2_backup_dir`