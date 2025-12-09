# Summary
Defines Bash associative arrays and variables for the **create‑customer‑cdr‑actives** pipeline. It loads common MNAAS properties, reads a static customer list, and dynamically builds per‑customer column definitions, column positions, output/backup directories, feed‑file patterns, and filter conditions. These structures are consumed by `MNAAS_create_customer_cdr_actives_files.sh` to generate active‑CDR files for each customer in the Move‑Network‑As‑A‑Service production workflow.

# Key Components
- **MNAAS_Create_New_Customers** – path to the static customer list file (`MNAAS_Create_New_Customers_Non_traffic.properties`).  
- **MNASS_Create_Customer_actives_files_columns** – associative array mapping customer name → semicolon‑delimited column list.  
- **MNAAS_Create_Customer_actives_files_column_position** – associative array mapping customer name → comma‑separated list of input‑field positions (`$4,$5,…`).  
- **MNAAS_Create_Customer_actives_files_out_dir** – associative array mapping customer name → output directory for generated actives files.  
- **MNAAS_Create_Customer_actives_files_backup_dir** – associative array mapping customer name → backup directory for processed files.  
- **FEED_FILE_PATTERN** – associative array mapping feed identifiers (`SNG_01`, `HOL_01`, `HOL_02`) → filename glob pattern using the `actives` extension.  
- **MNAAS_Create_Customer_actives_files_filter_condition** – associative array mapping customer name → AWK filter expression used to select rows for that customer.  
- **MNAAS_Customer_Seq_check_JarPath**, **MNAAS_MyRep_seqno_check_classname**, **MNASS_MyRep_SeqNo_Check_*_files** – configuration for sequence‑number validation JAR.  
- **MNAAS_MyRep_timestamp_check_classname**, **MNASS_MyRep_Timestamp_Check_*_files** – configuration for timestamp validation JAR.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| Load common props | `MNAAS_CommonProperties.properties` | `source` into current shell | Environment variables (paths, log locations) |
| Load customer list | `MNAAS_Create_New_Customers_Non_traffic.properties` (pipe‑delimited) | `cut -d'|' -f1` loop | Dynamic population of all associative arrays for any new customer |
| Generate actives files | Raw CDR feed files matching `FEED_FILE_PATTERN` | `awk` using column positions & filter condition per customer | Files written to `MNAAS_Create_Customer_actives_files_out_dir[customer]` |
| Backup processed feeds | Same feed files | `mv`/`cp` to `MNAAS_Create_Customer_actives_files_backup_dir[customer]` | Archived copies in backup directories |
| Validation | Generated actives files | Java JAR (`mnaas_Customer_Seq_Check.jar`) invoked with class names and config paths | Logs / status files under `*_history_of_files`, `*_current_missing_files` |

# Integrations
- **MNAAS_create_customer_cdr_actives_files.sh** – primary consumer; sources this property file to obtain all associative arrays.  
- **MNAAS_CommonProperties.properties** – provides base configuration (log paths, Hadoop user, etc.).  
- **MNAAS_Create_New_Customers_Non_traffic.properties** – external static list of customers; can be extended without code change.  
- **Java validation JARs** – invoked downstream for sequence‑number and timestamp checks.  
- **Hadoop / HDFS** – implied by path conventions (`/app/hadoop_users/...`) but not directly accessed in this script.

# Operational Risks
- **Missing/Corrupt customer list** – script will silently skip undefined customers; mitigate by validating the properties file existence and format before execution.  
- **Incorrect column position mapping** – hard‑coded `$` indices assume a fixed input schema; schema change will break output. Mitigation: externalize schema definition or add unit tests.  
- **Filter condition syntax errors** – embedded quotes may break when customer names contain special characters. Mitigation: sanitize customer identifiers.  
- **Backup directory collisions** – same path for multiple customers could cause overwrites. Ensure unique backup paths per customer (already done via `${new_customer}_backup`).  
- **Jar path or class name mismatch** – validation steps will fail silently if JAR not present. Add existence checks and exit codes.

# Usage
```bash
# Source the property file in the main script
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
. /path/to/MNAAS_create_customer_cdr_actives_files_dynamic.properties

# Run the generation script (example)
bash MNAAS_create_customer_cdr_actives_files.sh \
    -feedDir /data/feeds \
    -date 2025-12-07
```
To debug associative arrays:
```bash
declare -p MNASS_Create_Customer_actives_files_columns
declare -p MNAAS_Create_Customer_actives_files_column_position
```

# Configuration
- **Environment variables** (populated by `MNAAS_CommonProperties.properties`):  
  - `MNAASConfPath`, `MNAASLocalLogPath`, `MNAAS_dtail_extn['actives']`, `MNAAS_Customer_SECS` (customer SEC codes).  
- **External config files**:  
  - `MNAAS_Create_New_Customers_Non_traffic.properties` (customer list).  
  - Validation JAR and class names (`MNAAS_Customer_Seq_check_JarPath`, `MNAAS_MyRep_seqno_check_classname`, etc.).  

# Improvements
1. **Externalize schema definitions** – replace hard‑coded `$4,$5,…` with a configurable column‑map file to accommodate future feed format changes.  
2. **Add validation step** – implement a pre‑run sanity check that verifies the existence and parse‑ability of all referenced files (customer list, JARs, extensions) and aborts with clear error messages.