# Summary
`MNAAS_create_customer_cdr_activation_files_dynamic.properties` is a Bash‑style property script that loads common MNAAS properties, parses the static customer list (`MNAAS_Create_New_Customers_Non_traffic.properties`), and dynamically builds associative arrays used by downstream activation‑file generation scripts. The arrays define per‑customer column definitions, column positions, output/backup directories, feed‑file patterns, and filter conditions for the `MNAAS_create_customer_cdr_activation_files.sh` pipeline.

# Key Components
- **Source inclusion** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` loads base constants.  
- **Variable definitions** – script name, status file, log path, and path to the static customer list.  
- **Associative arrays**  
  - `MNASS_Create_Customer_activation_files_columns` – column header list per customer.  
  - `MNAAS_Create_Customer_activation_files_column_position` – awk‑style field list for each customer.  
  - `MNAAS_Create_Customer_activation_files_out_dir` – target activation‑file directory per customer.  
  - `MNAAS_Create_Customer_activation_files_backup_dir` – backup directory per customer.  
  - `FEED_FILE_PATTERN` – filename glob patterns for activation feed files.  
  - `MNAAS_Create_Customer_activation_files_filter_condition` – awk filter expression referencing `MNAAS_Customer_SECS`.  
- **Dynamic population loops** – `for new_customer in \`cut -d'|' -f1 $MNAAS_Create_New_Customers\`` extends each associative array with entries for newly added customers.  
- **Validation JAR configuration** – paths and class names for sequence‑number and timestamp validation jobs.

# Data Flow
| Stage | Input | Process | Output / Side Effect |
|-------|-------|---------|----------------------|
| Load common props | `MNAAS_CommonProperties.properties` | source | environment variables, base arrays (`MNAAS_Customer_SECS`, etc.) |
| Load customer list | `MNAAS_Create_New_Customers_Non_traffic.properties` (pipe‑delimited) | `cut` extracts first field | dynamic keys for associative arrays |
| Build arrays | static definitions + dynamic loop | Bash associative array assignment | In‑memory maps consumed by `MNAAS_create_customer_cdr_activation_files.sh` |
| Validation config | hard‑coded JAR/class paths | variable assignment | Paths passed to downstream Java validation steps |

No direct DB or queue interaction; all data resides on HDFS/local FS.

# Integrations
- **`MNAAS_create_customer_cdr_activation_files.sh`** – sources this property file to obtain per‑customer configuration for activation file generation.  
- **`MNAAS_CommonProperties.properties`** – provides foundational mappings (`MNAAS_Customer_SECS`, `MNAAS_dtail_extn`, etc.).  
- **Java validation JARs** – `mnaas_Customer_Seq_Check.jar` and timestamp validation JAR are invoked later in the pipeline using the class names defined here.  
- **Hadoop/HDFS** – output and backup directories resolve to HDFS paths referenced by downstream ingestion jobs.

# Operational Risks
- **Stale customer list** – if `MNAAS_Create_New_Customers_Non_traffic.properties` is not refreshed, new customers will lack proper mappings. *Mitigation*: schedule periodic validation of the source file.  
- **Array key collisions** – duplicate customer identifiers cause overwrites. *Mitigation*: enforce uniqueness in the source file via CI lint.  
- **Hard‑coded field positions** – changes in upstream feed schema break column extraction. *Mitigation*: externalize field mapping to a version‑controlled schema file.  
- **Missing common properties** – failure to source `MNAAS_CommonProperties.properties` aborts the script. *Mitigation*: add existence check and fallback defaults.

# Usage
```bash
# Source the dynamic property file in a Bash session
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_create_customer_cdr_activation_files_dynamic.properties

# Verify a generated map, e.g., output directory for a customer
echo "${MNAAS_Create_Customer_activation_files_out_dir[MyRep]}"
# Run the activation generation script (debug mode)
bash -x /app/hadoop_users/MNAAS/MNAAS_Scripts/MNAAS_create_customer_cdr_activation_files.sh
```

# Configuration
- **Environment variables** (set in `MNAAS_CommonProperties.properties`):  
  - `MNAASConfPath`, `MNAASLocalLogPath`, `MNAAS_dtail_extn` (file extensions).  
- **Referenced files**:  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Create_New_Customers_Non_traffic.properties`  
  - Validation JARs under `/app/hadoop_users/MNAAS/MNAAS_Jar/Raw_Aggr_tables_processing/`.  

# Improvements
1. **Externalize schema definitions** – move column lists and field positions to a JSON/YAML file parsed at runtime to simplify maintenance when feed formats evolve.  
2. **Add validation step** – implement a pre‑run sanity check that verifies the customer list file exists, is non‑empty, and contains unique keys; abort with clear error if validation fails.