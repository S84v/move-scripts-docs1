# Summary
`MNAAS_create_customer_cdr_tolling_files_dynamic.properties` defines all runtime variables, associative arrays, and file‑pattern mappings required by the Bash script `MNAAS_create_customer_cdr_tolling_files.sh`. It supplies per‑customer column lists, column‑position selectors, output/backup directories, feed‑file glob patterns, AWK filter expressions, and paths to validation JARs used in the Move‑Network‑As‑A‑Service (MNAAS) tolling CDR generation pipeline.

# Key Components
- **Scalar variables** – script name, process‑status file, log path, new‑customer list reference.  
- **`MNASS_Create_Customer_tolling_files_columns`** – associative array mapping each customer to the ordered list of output columns.  
- **`MNAAS_Create_Customer_tolling_files_column_position`** – associative array of `$`‑field selectors for the input feed.  
- **`MNAAS_Create_Customer_tolling_files_out_dir`** – per‑customer output directory mapping.  
- **`MNAAS_Create_Customer_tolling_files_backup_dir`** – per‑customer backup directory mapping.  
- **`FEED_FILE_PATTERN`** – mapping of feed identifiers (e.g., `SNG_01`) to filename glob patterns using the common tolling extension.  
- **`MNAAS_Create_Customer_tolling_files_filter_condition`** – AWK boolean expressions that filter rows per customer based on SEC codes.  
- **Validation JAR & class names** – paths and class identifiers for sequence‑number and timestamp validation of generated tolling files.

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|------|-------|-------------|--------|--------------|
| 1. Load common properties | `MNAAS_CommonProperties.properties` | `source` | environment variables | – |
| 2. Load new‑customer list | `MNAAS_Create_New_Customers_Non_traffic.properties` | `cut` | dynamic population of associative arrays | – |
| 3. Feed discovery | Files matching patterns in `FEED_FILE_PATTERN` | Bash loop → AWK using column positions & filter conditions | Customer‑specific toll‑CDR files in `out_dir` | Backup copies written to `backup_dir` |
| 4. Validation | Generated files | Java JAR (`mnaas_Customer_Seq_Check.jar`) invoked with class names | Validation logs / missing‑file reports | Files written to history/missing directories |

# Integrations
- **Common property source** – `MNAAS_CommonProperties.properties` provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`, `MNAAS_dtail_extn`, `MNAAS_Customer_SECS`).  
- **Customer list** – `MNAAS_Create_New_Customers_Non_traffic.properties` extends static customer mappings.  
- **Processing script** – `MNAAS_create_customer_cdr_tolling_files.sh` sources this file to obtain all runtime configuration.  
- **Validation services** – Java JAR executed from the script for sequence‑number and timestamp checks; results consumed by downstream monitoring/alerting components.  

# Operational Risks
- **Missing or malformed new‑customer list** → associative arrays incomplete → runtime errors. *Mitigation*: validate list file existence and format before sourcing.  
- **Hard‑coded field positions** (`$4,$5,…`) assume stable feed schema; schema change breaks output. *Mitigation*: version feed schema and update arrays in lockstep.  
- **Path concatenation without quoting** may fail on spaces. *Mitigation*: enforce path sanitization or use arrays.  
- **Java validation JAR version drift** → false negatives/positives. *Mitigation*: pin JAR version and include checksum verification in deployment pipeline.  

# Usage
```bash
# Source the configuration (performed inside the main script)
. /path/to/MNAAS_create_customer_cdr_tolling_files_dynamic.properties

# Run the processing script (example)
bash MNAAS_create_customer_cdr_tolling_files.sh -d /data/feeds -c MyRep
# Debug: enable Bash tracing
bash -x MNAAS_create_customer_cdr_tolling_files.sh
```

# Configuration
- **Environment variables** (provided by common properties):  
  - `MNAASConfPath`, `MNAASLocalLogPath`, `MNAAS_dtail_extn`, `MNAAS_Customer_SECS`  
- **Referenced files**:  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Create_New_Customers_Non_traffic.properties`  
  - Validation JAR: `/app/hadoop_users/MNAAS/MNAAS_Jar/Raw_Aggr_tables_processing/mnaas_Customer_Seq_Check.jar`  
  - Validation output directories under `/app/hadoop_users/MNAAS/MNAAS_Configuration_Files/...`  

# Improvements
1. **Externalize schema definitions** – move column‑position strings to a JSON/YAML file and generate associative arrays at runtime to simplify schema updates.  
2. **Add sanity checks** – implement a pre‑run validation function that verifies existence of all referenced directories, files, and that each customer key has a complete set of mappings; abort with clear error messages if any check fails.