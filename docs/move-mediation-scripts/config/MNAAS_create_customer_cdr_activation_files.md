# Summary
Defines Bash associative arrays and variables for the “create‑customer‑cdr‑activation” pipeline. It loads common properties, selects the customer list from a dynamic property file, and builds per‑customer column definitions, column positions, output/backup directories, feed‑file patterns, and filter conditions used by the `MNAAS_create_customer_cdr_activation_files.sh` script to generate activation files for each customer in the Move‑Network‑As‑A‑Service production workflow.

# Key Components
- **Source of common properties** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
- **Script name & status/log paths** – `MNAAS_Create_Customer_cdr_activation_scriptname`, `*_ProcessStatusFilename`, `*_LogPath`  
- **Customer list** – `MNAAS_Create_New_Customers` (points to a non‑traffic customer definition file)  
- **`MNASS_Create_Customer_activation_files_columns`** – associative array mapping each customer to a semicolon‑delimited column list.  
- **`MNAAS_Create_Customer_activation_files_column_position`** – associative array mapping each customer to the positional `$` field list used by `awk`/`cut`.  
- **`MNAAS_Create_Customer_activation_files_out_dir`** – output directory per customer (including special‑case handling for KCS, eLUX, Uber, etc.).  
- **`MNAAS_Create_Customer_activation_files_backup_dir`** – backup directory per customer.  
- **`FEED_FILE_PATTERN`** – pattern strings for inbound feed files (SNG, HOL).  
- **`MNAAS_Create_Customer_activation_files_filter_condition`** – awk filter expression per customer, referencing the SECS ID from the dynamic mapping `MNAAS_Customer_SECS`.  
- **SeqNo & Timestamp check file locations** – paths for activation‑file integrity monitoring.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| Load config | `MNAAS_CommonProperties.properties` | `source` into current shell | Environment variables, associative arrays |
| Load customer list | `MNAAS_Create_New_Customers` (pipe‑delimited) | `cut -d'|' -f1` loop | Dynamic entries added to all associative arrays |
| Build column definitions | Hard‑coded list + dynamic loop | Populate `MNASS_Create_Customer_activation_files_columns` | In‑memory map |
| Build column positions | Hard‑coded list + dynamic loop | Populate `MNAAS_Create_Customer_activation_files_column_position` | In‑memory map |
| Determine output/backup dirs | Hard‑coded list + conditional logic for special customers | Populate `MNAAS_Create_Customer_activation_files_out_dir` & `*_backup_dir` | In‑memory map |
| Feed file pattern mapping | Static literals | Populate `FEED_FILE_PATTERN` | In‑memory map |
| Filter condition generation | `MNAAS_Customer_SECS` (populated elsewhere) | Populate `MNAAS_Create_Customer_activation_files_filter_condition` | In‑memory map |
| Monitoring files | `MNAASConfPath` variable | Define paths for seq‑no & timestamp checks | Files used by downstream validation scripts |

No direct external services are invoked; the file only prepares configuration data for downstream Bash/awk/Java/Hive jobs.

# Integrations
- **`MNAAS_CommonProperties.properties`** – provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`, etc.).  
- **`MNAAS_Create_New_Customers_*.properties`** – supplies the dynamic customer list and SECS IDs (via `MNAAS_Customer_SECS` associative array defined in the dynamic property script).  
- **`MNAAS_create_customer_cdr_activation_files.sh`** – consumes all associative arrays defined here to process raw activation feeds, apply filters, and write per‑customer files.  
- **SeqNo/Timestamp check scripts** – read the paths defined at the end of this file to verify file completeness and ordering.  
- **Hive/Impala jobs** – downstream scripts may reference the generated activation files for loading into data warehouses.

# Operational Risks
- **Missing or malformed customer entry** – `cut` on a pipe‑delimited file may produce empty keys; mitigated by validating `$MNAAS_Create_New_Customers` before sourcing.  
- **Hard‑coded column positions** – any change in upstream feed schema requires manual update; mitigate with schema‑version control and automated tests.  
- **Special‑case directory logic** – new customers added without updating the conditional block will default to generic path; ensure onboarding checklist updates this file.  
- **Dependency on `MNAAS_Customer_SECS`** – if the dynamic mapping script fails, filter conditions become invalid; add error‑checking after sourcing dynamic properties.  
- **Log file name includes `$(date +_%F)` at variable definition time** – may cause log rotation issues if the script is sourced multiple times; consider generating log name at execution time.

# Usage
```bash
# Source the configuration in a Bash session
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source /path/to/MNAAS_create_customer_cdr_activation_files.properties

# Verify associative arrays
declare -p MNAAS_Create_Customer_activation_files_out_dir
declare -p MNAAS_Create_Customer_activation_files_columns

# Run the main script (example)
bash /app/hadoop_users/MNAAS/MNAAS_Scripts/MNAAS_create_customer_cdr_activation_files.sh
```
For debugging, add `set -x` after sourcing to trace array population.

# Configuration
- **Environment variables** (inherited from `MNAAS_CommonProperties.properties`):  
  - `MNAASConfPath` – base config directory.  
  - `MNAASLocalLogPath` – local log root.  
- **Referenced files**:  
  - `MNAAS_CommonProperties.properties` (static base).  
  - `MNAAS_Create_New_Customers_Non_traffic.properties` (dynamic customer list).  
  - Dynamic property script that defines `MNAAS_Customer_SECS` and other mappings (e.g., `MNAAS_CommonProperties_dynamic*.properties`).  
- **File extensions**: `MNAAS_dtail_extn['activations']` is expected to be defined in the dynamic property script.

# Improvements
1. **Externalize associative array definitions to JSON/YAML** and load them with `jq` or `yq` to simplify maintenance and enable schema validation.  
2. **Add validation step** after sourcing to assert that every customer key has non‑empty column list, position, output, backup, and filter condition; abort with clear error if any mapping is incomplete.