# Summary
Defines all runtime variables, associative arrays, and file‑pattern mappings required by the Bash job **MNAAS_create_customer_cdr_traffic_files.sh**. It supplies per‑customer column lists, column‑position selectors, output/backup directories, feed‑file glob patterns, AWK filter expressions, and paths to validation JARs used in the Move‑Network‑As‑A‑Service (MNAAS) tolling CDR traffic‑file generation pipeline.

# Key Components
- **Scalar variables** – script name, status‑file path, log path, new‑customer list file, validation‑jar path, class names, and history‑file locations.  
- **Associative array `MNASS_Create_Customer_traffic_files_columns`** – maps each customer key to a semicolon‑delimited list of column names to be emitted.  
- **Associative array `MNAAS_Create_Customer_traffic_files_column_position`** – maps each customer to a comma‑separated list of `$n` field selectors used by `awk` to reorder input columns.  
- **Associative array `MNAAS_Create_Customer_traffic_files_out_dir`** – destination directory for each customer’s generated traffic files.  
- **Associative array `MNAAS_Create_Customer_traffic_files_backup_dir`** – backup directory for each customer’s generated files.  
- **Associative array `FEED_FILE_PATTERN`** – glob patterns for inbound feed files (SNG, HOL).  
- **Associative array `MNAAS_Create_Customer_traffic_files_filter_condition`** – per‑customer AWK filter expression, including numeric formatting and SECS code matching.  
- **Loop blocks** – extend the above associative arrays with entries from the external file `$MNAAS_Create_New_Customers`.  

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|------|-------|------------|----------------------|
| Load common props | `MNAAS_CommonProperties.properties` | `source` into current shell | Environment variables |
| Load new‑customer list | `$MNAAS_Create_New_Customers` (pipe‑delimited) | `cut -d'|' -f1` → loop | Dynamic entries added to associative arrays |
| Generate traffic files (performed by the main script) | Raw CDR feed files matching `FEED_FILE_PATTERN` | `awk` with `${MNAAS_Create_Customer_traffic_files_column_position[customer]}` and `${MNAAS_Create_Customer_traffic_files_filter_condition[customer]}` | Files written to `${MNAAS_Create_Customer_traffic_files_out_dir[customer]}`; copies to backup dir |
| Validation | Generated files + validation JARs | Java classes `Seqno_validation` & `Timestamp_validation` invoked via `$MNAAS_Customer_Seq_check_JarPath` | Logs, history files under `$MNASS_MyRep_SeqNo_Check_traffic_*` and `$MNASS_MyRep_Timestamp_Check_traffic_*` |

# Integrations
- **MNAAS_CommonProperties.properties** – provides base paths (`$MNAASConfPath`, `$MNAASLocalLogPath`, `$MNAAS_dtail_extn`, `$MNAAS_Customer_SECS`).  
- **MNAAS_Create_New_Customers.properties** – external list of customers to be added at runtime.  
- **MNAAS_create_customer_cdr_traffic_files.sh** – consumes all associative arrays defined here.  
- **Java validation JAR** (`mnaas_Customer_Seq_Check.jar`) – called from the Bash job for sequence‑number and timestamp checks.  
- **File system** – reads raw feed files, writes to per‑customer output and backup directories, writes status and log files.  

# Operational Risks
- **Missing/ malformed new‑customer file** → associative arrays incomplete → runtime `awk` errors. *Mitigation*: validate `$MNAAS_Create_New_Customers` existence and format before sourcing.  
- **Hard‑coded column positions** become stale when upstream CDR schema changes. *Mitigation*: version control schema definitions; add automated schema‑validation step.  
- **Path concatenation without quoting** may break on spaces. *Mitigation*: enforce quoting of all variable expansions.  
- **AWK filter strings embed unescaped quotes**; SECS values containing special characters could break the script. *Mitigation*: sanitize `${MNAAS_Customer_SECS[...]}` before insertion.  

# Usage
```bash
# Source the properties (normally done by the main script)
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source move-mediation-scripts/config/MNAAS_create_customer_cdr_traffic_files_dynamic.properties

# Run the main job (example)
bash MNAAS_create_customer_cdr_traffic_files.sh -c MyRep -d 2025-01-15
```
*Debug*: add `set -x` after sourcing to trace associative‑array population.

# Configuration
- **Environment variables** (populated by common properties):  
  - `MNAASConfPath`, `MNAASLocalLogPath`, `MNAAS_dtail_extn`, `MNAAS_Customer_SECS`  
- **External config files**:  
  - `MNAAS_CommonProperties.properties`  
  - `MNAAS_Create_New_Customers.properties` (pipe‑delimited list of new customers)  
- **Scalar definitions in this file**: script name, status‑file, log path, validation JAR path, class names, history directories.  

# Improvements
1. **Externalize column‑position maps** to a CSV/JSON file and generate associative arrays programmatically – reduces duplication and eases schema updates.  
2. **Add validation function** that checks existence of all referenced directories, files, and that each customer key has matching entries across all associative arrays before processing begins.  