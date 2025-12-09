# Summary
Defines Bash associative arrays and variables for the “create‑customer‑cdr‑actives” pipeline. It loads common properties, reads a static customer list, and builds per‑customer column definitions, column positions, output/backup directories, feed‑file patterns, and filter conditions used by `MNAAS_create_customer_cdr_actives_files.sh` to generate active‑CDR files for each customer in the Move‑Network‑As‑A‑Service production workflow.

# Key Components
- **Source of common properties** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Script metadata variables** – `MNAAS_Create_Customer_cdr_actives_scriptname`, `*_ProcessStatusFilename`, `*_LogPath`
- **Customer list source** – `MNAAS_Create_New_Customers` (points to `MNAAS_Create_New_Customers_Non_traffic.properties`)
- **Associative arrays**
  - `MNASS_Create_Customer_actives_files_columns` – column header list per customer
  - `MNAAS_Create_Customer_actives_files_column_position` – `$`‑field positions for extraction
  - `MNAAS_Create_Customer_actives_files_out_dir` – HDFS/FS output directories per customer
  - `MNAAS_Create_Customer_actives_files_backup_dir` – backup directories per customer
  - `FEED_FILE_PATTERN` – filename glob patterns for feed files
  - `MNAAS_Create_Customer_actives_files_filter_condition` – awk‑style filter expressions per customer
- **Dynamic population loops** – `for new_customer in \`cut -d'|' -f1 $MNAAS_Create_New_Customers\`` extends each associative array with entries for newly added customers.

# Data Flow
- **Input**
  - `MNAAS_CommonProperties.properties` (common config)
  - `MNAAS_Create_New_Customers_Non_traffic.properties` (static customer list, pipe‑delimited)
- **Processing**
  - Bash parses the customer list, populates associative arrays.
  - Arrays are consumed by `MNAAS_create_customer_cdr_actives_files.sh` to:
    - Read raw CDR feed files matching `FEED_FILE_PATTERN`.
    - Apply `MNAAS_Create_Customer_actives_files_filter_condition` (awk) to select rows.
    - Extract columns using `MNAAS_Create_Customer_actives_files_column_position`.
    - Write per‑customer active files to `*_out_dir`.
    - Copy originals to `*_backup_dir`.
- **Outputs**
  - Per‑customer active CDR files in designated output directories.
  - Backup copies in corresponding backup directories.
  - Process status and log files (`*_ProcessStatusFile`, `*_log`).

# Integrations
- **Downstream** – Files produced are consumed by ingestion jobs, validation pipelines, and backup/archival processes.
- **Upstream** – Relies on common property definitions (`MNAAS_CommonProperties.properties`) and the static customer list file.
- **External services** – HDFS or local filesystem paths referenced by output/backup directories; may be accessed by Hadoop jobs or other shell scripts.

# Operational Risks
- **Missing/incorrect customer entry** – New customers not present in static list cause empty array entries; mitigated by validating the customer list before execution.
- **Hard‑coded column positions** – Changes in source feed schema break extraction; mitigated by version‑controlled schema definitions and automated tests.
- **Path concatenation errors** – Relative paths may resolve incorrectly on different nodes; mitigated by using absolute paths or environment‑driven base directories.
- **Large associative arrays** – Bash memory limits on very large customer sets; mitigated by splitting into multiple property files or migrating to a more scalable language (e.g., Python).

# Usage
```bash
# Load the property file (sets up arrays)
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/config/MNAAS_create_customer_cdr_actives_files.properties

# Execute the generation script
/app/hadoop_users/MNAAS/MNAAS_Scripts/MNAAS_create_customer_cdr_actives_files.sh \
    --date $(date +%Y-%m-%d)   # optional args as defined by the script
```
To debug, enable Bash tracing:
```bash
set -x
source ...properties
```

# Configuration
- **Environment variables** (expected to be set in `MNAAS_CommonProperties.properties`):
  - `MNAASConfPath`
  - `MNAASLocalLogPath`
  - `MNAAS_dtail_extn` (associative array for file extensions)
  - `MNAAS_Customer_SECS` (customer‑to‑SECSID mapping)
- **Referenced config files**
  - `MNAAS_CommonProperties.properties`
  - `MNAAS_Create_New_Customers_Non_traffic.properties`

# Improvements
1. **Externalize schema definitions** – Move column header lists and position strings to a version‑controlled JSON/YAML file and load them at runtime to simplify maintenance.
2. **Add validation step** – Implement a pre‑run check that verifies all referenced customers exist in `MNAAS_Customer_SECS` and that feed files match expected patterns; abort with clear error messages if mismatches are detected.