# Summary
Defines runtime constants for the **MNAAS_move_files_from_staging** batch job. The file sources common properties, then declares the shell‑script name, process‑status file location, log‑file naming convention, staging‑to‑realtime file‑matching patterns, and the target real‑time ingestion directory. These constants are consumed by `MNAAS_move_files_from_staging.sh` to move staged CDR/TAP‑error files into the production ingestion path while recording execution status and logs.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – imports shared environment variables (e.g., `MNAASConfPath`, `MNAASLocalLogPath`).
- **`MNAAS_move_files_from_staging_ScriptName`** – name of the executable shell script (`MNAAS_move_files_from_staging.sh`).
- **`MNAAS_move_files_from_staging_ProcessStatusFileName`** – absolute path to the job’s process‑status file (`$MNAASConfPath/...`).
- **`MNAAS_move_files_from_staging_LogPath`** – log file path with date suffix (`$MNAASLocalLogPath/..._YYYY-MM-DD.log`).
- **`Holland01_tap_error_extn`**, **`Singapore01_tap_error_extn`**, **`Holland01_failed_events_extn`**, **`Singapore01_failed_events_extn`** – glob patterns for locating TAP‑error and Failed‑Events files per customer.
- **`SSH_MOVE_REALTIME_PATH`** – destination directory for real‑time CDR ingestion (`/quest/voicesqm/Move_CDR/RealTime_CDR`).

# Data Flow
| Phase | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| **Initialization** | `MNAAS_CommonProperties.properties` | Source common variables (`MNAASConfPath`, `MNAASLocalLogPath`, etc.) | Populated environment for downstream script |
| **File Selection** | Staged directory (implicit) | Shell script expands the four glob patterns to locate matching files | List of files to move |
| **File Transfer** | Selected files | `scp`/`mv` (implemented in `MNAAS_move_files_from_staging.sh`) to `SSH_MOVE_REALTIME_PATH` | Files placed in real‑time ingestion path |
| **Status Recording** | Job execution result | Write status (`SUCCESS`/`FAILURE`) to `MNAAS_move_files_from_staging_ProcessStatusFileName` | Process‑status file updated |
| **Logging** | Execution details | Append timestamped entries to `MNAAS_move_files_from_staging_LogPath` | Persistent log file per run |

# Integrations
- **Common Property Library** – `MNAAS_CommonProperties.properties` supplies base paths and Hadoop environment settings.
- **MNAAS_move_files_from_staging.sh** – consumes all constants defined herein; invoked by the production scheduler (e.g., Oozie, cron).
- **Monitoring/Alerting** – external health‑check scripts may read the process‑status file to trigger alerts.
- **Downstream Ingestion Pipelines** – files placed in `SSH_MOVE_REALTIME_PATH` are consumed by real‑time CDR processing jobs (Hive, Spark, etc.).

# Operational Risks
- **Missing/Corrupt Common Properties** – job fails to resolve `$MNAASConfPath` or `$MNAASLocalLogPath`. *Mitigation*: validate existence of sourced file at start of script.
- **Pattern Mismatch** – glob patterns may not match actual file names after schema changes. *Mitigation*: unit‑test patterns against a sample staging directory; maintain versioned pattern list.
- **Permission Errors** – insufficient rights on staging or real‑time directories. *Mitigation*: enforce least‑privilege ACLs; log permission failures explicitly.
- **Date Suffix Collision** – log file name uses `date +_%F`; if the script runs multiple times within the same day, logs may be overwritten. *Mitigation*: append a timestamp (`%T`) or unique run identifier.

# Usage
```bash
# Load properties (normally done by the scheduler)
source /path/to/MNAAS_move_files_from_staging27042021.properties

# Execute the batch job
bash $MNAAS_move_files_from_staging_ScriptName

# Debug: print resolved variables
echo "Status file: $MNAAS_move_files_from_staging_ProcessStatusFileName"
echo "Log file:    $MNAAS_move_files_from_staging_LogPath"
echo "Patterns:   $Holland01_tap_error_extn $Singapore01_tap_error_extn ..."
```

# Configuration
- **Environment Variables (inherited from common properties)**
  - `MNAASConfPath` – base config directory.
  - `MNAASLocalLogPath` – base log directory.
- **Referenced Config File**
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Static Constants (defined in this file)**
  - Script name, process‑status file name, log path template, file‑match patterns, destination path.

# Improvements
1. **Parameterize Date Format** – replace hard‑coded `date +_%F` with a configurable variable to allow sub‑daily granularity or alternative formats.
2. **Add Validation Block** – implement a pre‑execution check that verifies:
   - Existence and readability of common properties file.
   - Write permission on `SSH_MOVE_REALTIME_PATH`.
   - Non‑empty result set for each glob pattern.  
   Abort with a clear error code if any check fails.