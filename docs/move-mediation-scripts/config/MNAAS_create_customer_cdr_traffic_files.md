# Summary
The `MNAAS_create_customer_cdr_traffic_files.properties` script defines the configuration for the **MNAAS Create Customer CDR Traffic Files** job. It loads common properties, declares associative arrays that map each customer to its column list, column positions, output/backup directories, feed‑file patterns, and AWK filter conditions. The script is used at runtime to drive the generation, filtering, and placement of CDR traffic files for all customers during a production move.

# Key Components
- **Source inclusion** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
- **Script metadata** – `MNAAS_Create_Customer_cdr_traffic_scriptname`, status‑file, log‑file paths.  
- **Customer column definitions** – `declare -A MNASS_Create_Customer_traffic_files_columns` (static per‑customer column header list).  
- **Infonova column definitions** – `declare -A MNASS_Create_Customer_traffic_files_columns_Infonova`.  
- **Dynamic column‑position mapping** – two `while IFS='|' read …` loops that populate `MNASS_Create_Customer_traffic_files_column_position` and `MNAAS_Create_Customer_traffic_files_column_position_Infonova` based on the master customer list file (`$MNAAS_Create_New_Customers`).  
- **Output directory map** – `declare -A MNAAS_Create_Customer_traffic_files_out_dir` (pre‑populated) plus a fallback loop that adds missing customers under `Crushsftp/<customer>/traffic`.  
- **Backup directory map** – `declare -A MNAAS_Create_Customer_traffic_files_backup_dir` with similar fallback logic.  
- **Feed‑file pattern map** – `declare -A FEED_FILE_PATTERN`.  
- **AWK filter condition map** – `declare -A MNAAS_Create_Customer_traffic_files_filter_condition` (static + dynamic augmentation for new customers).  
- **Lookup‑specific filter** – `declare -A MNAAS_Create_Customer_traffic_files_filter_condition_lookup` (currently only JLR).  

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| Load common props | `MNAAS_CommonProperties.properties` | Bash variable export | Global env vars (e.g., `$MNAASConfPath`, `$MNAASLocalLogPath`) |
| Customer list | `$MNAAS_Create_New_Customers` (pipe‑separated file) | Two loops read each line, populate associative arrays for column positions, output dirs, backup dirs, filter conditions | In‑memory maps used by downstream CDR generation scripts |
| Column header map | Hard‑coded literals | Direct assignment to `MNASS_Create_Customer_traffic_files_columns` | Header strings for each customer |
| Column position map | Customer list + conditional logic per customer | Build `$1,$2,…` position strings for AWK `cut` | Position strings stored in `MNAAS_Create_Customer_traffic_files_column_position` |
| Output/backup dirs | Pre‑defined literals + fallback logic | Ensure map contains an entry for every customer | Directory path strings (actual directories are created later by the main job) |
| Feed pattern map | Hard‑coded literals | Simple key/value assignment | Pattern strings used to locate source files |
| Filter condition map | Hard‑coded literals + dynamic addition for new customers | Build AWK filter expressions (field formatting, SECS match, optional lookup) | Expressions consumed by the CDR processing pipeline |

No direct DB or queue interaction occurs in this file; it only prepares configuration data for downstream Hadoop/MapReduce or shell‑based processing.

# Integrations
- **MNAAS_Create_Customer_cdr_traffic_files.sh** – consumes the associative arrays defined here to drive file extraction, transformation, and loading.  
- **MNAAS_CommonProperties.properties** – provides base paths, file extensions (`MNAAS_dtail_extn['traffic']`), and SECS values (`MNAAS_Customer_SECS`).  
- **Hadoop environment** – paths such as `$MNAASConfPath` and `$MNAASLocalLogPath` are Hadoop‑specific directories.  
- **External feed files** – matched via `FEED_FILE_PATTERN` (e.g., `SNG*01`) located in source ingestion directories.  
- **AWK scripts** – filter condition strings are passed to AWK commands that parse raw CDR files.

# Operational Risks
- **Missing/duplicate customer entries** – If `$MNAAS_Create_New_Customers` contains duplicates, associative arrays may be overwritten silently. *Mitigation*: validate uniqueness before processing.  
- **Stale column definitions** – Hard‑coded column lists may diverge from source schema, causing mis‑aligned data. *Mitigation*: automate generation from a schema repository or add a validation step.  
- **Incorrect SECS mapping** – Filter conditions rely on `${MNAAS_Customer_SECS[...]}`; a typo leads to empty filters and data leakage. *Mitigation*: sanity‑check SECS values during deployment.  
- **Path collisions** – Fallback output/backup directory logic may create overlapping paths if a customer name matches an existing key. *Mitigation*: enforce naming conventions or log conflicts.  
- **Shell quoting errors** – Complex AWK expressions embedded in associative arrays can break if special characters appear in SECS values. *Mitigation*: escape or sanitize SECS strings.

# Usage
```bash
# Load the configuration (typically done by the main driver script)
source /path/to/move-mediation-scripts/config/MNAAS_create_customer_cdr_traffic_files.properties

# Example: print the column list for a given customer
echo "${MNASS_Create_Customer_traffic_files_columns[MyRep]}"

# Example: run the CDR creation script for a single customer (debug)
MNAAS_Create_New_Customers=/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Create_New_Customers.properties \
  ./MNAAS_create_customer_cdr_traffic_files.sh MyRep
```
To debug mapping issues, `declare -p` any associative array after sourcing.

# Configuration
- **Environment variables** (set in `MNAAS_CommonProperties.properties`):  
  - `MNAASConfPath` – base config directory.  
  - `MNAASLocalLogPath` – local log directory.  
  - `MNAAS_dtail_extn['traffic']` – file extension for traffic files.  
  -