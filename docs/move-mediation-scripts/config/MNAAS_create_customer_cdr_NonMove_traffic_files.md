# Summary
`MNAAS_create_customer_cdr_NonMove_traffic_files.properties` is a Bash‑style property script that defines associative arrays and scalar variables required by `MNAAS_create_customer_cdr_NonMove_traffic_files.sh`. It loads common MNAAS properties, specifies the script name, log and status file locations, column definitions, column positions, output/backup directories, feed‑file patterns, and per‑customer AWK filter conditions for generating non‑move CDR files for each customer (currently only **SIA**) in the Move‑Network‑As‑A‑Service production pipeline.

# Key Components
- **Source inclusion** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` loads shared configuration.
- **Scalar variables**
  - `MNAAS_create_customer_cdr_NonMove_traffic_files_scriptname` – target shell script.
  - `MNAAS_create_customer_cdr_NonMove_traffic_files_ProcessStatusFilename` – path to process‑status file.
  - `MNAAS_create_customer_cdr_NonMove_traffic_files_LogPath` – log file path with date suffix.
- **Associative arrays**
  - `MNASS_Create_Customer_NonMove_traffic_files_columns` – column header list per customer.
  - `MNAAS_Create_Customer_NonMove_traffic_files_column_position` – comma‑separated list of input‑field positions to extract.
  - `MNAAS_Create_Customer_NonMove_traffic_files_out_dir` – output directory per customer.
  - `MNAAS_Create_Customer_NonMove_traffic_files_backup_dir` – backup directory per customer.
  - `FEED_FILE_PATTERN` – glob pattern for source feed files (uses `MNAAS_dtail_extn` from common properties).
  - `MNAAS_Create_Customer_NonMove_traffic_files_filter_condition` – AWK filter expression per customer (formats field 45 and applies header/row checks).

# Data Flow
| Stage | Input | Processing | Output |
|-------|-------|------------|--------|
| **Load common props** | `MNAAS_CommonProperties.properties` | Bash `source` | Populates shared variables (`MNAAS_dtail_extn`, `MNAAS_Customer_SECS`, etc.) |
| **Read feed files** | Files matching `HOL*02${MNAAS_dtail_extn['nonmovecdr']}` in source directory | `awk` invoked by downstream script using `MNAAS_Create_Customer_NonMove_traffic_files_filter_condition` and column positions | Filtered, reformatted CDR records written to `${out_dir}/${customer}` |
| **Backup** | Original feed files | Copy/move after successful processing (logic in downstream script) | Files stored under `${backup_dir}/${customer}` |
| **Status & logging** | None | Writes status to `${ProcessStatusFilename}` and logs to `${LogPath}` | Status file, log file |

Side effects: file system writes (output, backup, logs, status file). No external services, DBs, or message queues are referenced directly.

# Integrations
- **Downstream script**: `MNAAS_create_customer_cdr_NonMove_traffic_files.sh` consumes all variables/arrays defined here.
- **Common properties**: `MNAAS_CommonProperties.properties` provides shared constants (`MNAAS_dtail_extn`, `MNAAS_Customer_SECS`, `MNAASConfPath`, `MNAASLocalLogPath`).
- **Potential upstream**: Scheduler/orchestrator (e.g., Oozie, Airflow) that triggers the downstream script.

# Operational Risks
- **Hard‑coded customer key** (`SIA`) – adding new customers requires manual script edit; risk of omission.
- **Column position mismatch** – if source feed schema changes, the position list becomes invalid, leading to malformed output.
- **AWK filter syntax** – embedded single‑quotes and variable expansion can break if customer identifiers contain special characters.
- **File pattern reliance on `MNAAS_dtail_extn`** – missing or incorrect extension mapping causes no input files to be found.

Mitigations: implement schema validation, externalize customer list to a CSV, add unit tests for AWK expressions, and enforce version control of `MNAAS_dtail_extn`.

# Usage
```bash
# Source the property file to export variables into the current shell
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source move-mediation-scripts/config/MNAAS_create_customer_cdr_NonMove_traffic_files.properties

# Run the processing script (typically invoked by a scheduler)
bash $MNAAS_create_customer_cdr_NonMove_traffic_files_scriptname
```
For debugging, echo associative arrays or run the downstream script with `set -x` to trace variable expansion.

# Configuration
- **Environment variables** (populated by common properties):
  - `MNAASConfPath` – directory for process‑status files.
  - `MNAASLocalLogPath` – base log directory.
  - `MNAAS_dtail_extn` – associative array mapping file types to extensions.
  - `MNAAS_Customer_SECS` – associative array mapping customer keys to SECS identifiers.
- **Referenced config files**
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
  - This property file itself (`MNAAS_create_customer_cdr_NonMove_traffic_files.properties`).

# Improvements
1. **Externalize customer definitions** – move per‑customer associative entries (columns, positions, dirs, filters) to a CSV or JSON file and generate the Bash arrays programmatically to simplify onboarding of new customers.
2. **Schema‑driven column mapping** – replace hard‑coded position strings with a lookup that reads a header line from the feed file, making the script resilient to column order changes.