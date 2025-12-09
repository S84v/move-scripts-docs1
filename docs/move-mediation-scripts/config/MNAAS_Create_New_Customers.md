# Summary
Defines the mapping of new customers to their source feed‑file glob patterns and destination HOL (Home‑Of‑Load) identifiers. The file is consumed at runtime by the MNAAS CDR traffic‑file generation scripts to determine which source files belong to each customer and which HOL bucket they should be processed into during a production move.

# Key Components
- **Customer‑to‑Feed Mapping** – Each line: `<CustomerName>|<FeedPattern>|<HOL_ID>`
- **Duplicate Entries** – Two rows per customer for HOL_01 and HOL_02, enabling parallel processing of two HOL environments.
- **Static Entries** – ENGINE, Adeo_Tech_LLC, GUPI_MOBILE, FLEXEXPORTATION, CLOUDEXPEDITION, XTREME_COMMUNICATIONS have single‑value feed patterns (no wildcard).

# Data Flow
- **Input**: This properties file is read by Bash/awk scripts (`MNAAS_create_customer_cdr_traffic_files*.sh`).
- **Processing**: Scripts parse each line, populate associative arrays:
  - `customer_to_pattern[customer]=feedPattern`
  - `customer_to_hol[customer]=holId`
- **Output**: Generated CDR traffic files placed in `$OUTPUT_DIR/$HOL_ID/<customer>/`.
- **Side Effects**: Creation of backup copies, logging to status files, optional validation JAR execution.
- **External Services**: None directly; downstream scripts may invoke SFTP, HDFS, or validation services.

# Integrations
- **`MNAAS_create_customer_cdr_traffic_files*.sh`** – loads this file via `source` or custom parser to build runtime configuration.
- **`MNAAS_create_customer_cdr_traffic_files_dynamic.properties`** – may reference the same customer list for dynamic column selection.
- **Validation JARs** – invoked after file generation, using customer list to select appropriate schema.

# Operational Risks
- **Incorrect FeedPattern** – Wildcard mis‑match leads to missing or extra files. *Mitigation*: Validate patterns against file system before move.
- **Duplicate Customer Entries** – Inconsistent HOL mapping can cause files to be written to wrong bucket. *Mitigation*: Enforce uniqueness check in parser.
- **Hard‑coded HOL IDs** – Adding new HOL environments requires file update. *Mitigation*: Externalize HOL mapping to a separate config.

# Usage
```bash
# Load mapping into associative arrays (example snippet)
declare -A CUSTOMER_PATTERN
declare -A CUSTOMER_HOL
while IFS='|' read -r cust pattern hol; do
  CUSTOMER_PATTERN["$cust"]="$pattern"
  CUSTOMER_HOL["$cust"]="$hol"
done < move-mediation-scripts/config/MNAAS_Create_New_Customers.properties

# Debug: print mapping for a customer
echo "Pattern for UberUSA: ${CUSTOMER_PATTERN[UberUSA]}"
echo "HOL for UberUSA: ${CUSTOMER_HOL[UberUSA]}"
```

# Configuration
- **Environment Variables**  
  - `MNAAS_Create_Customer_cdr_traffic_scriptname` – script name invoking this file.  
  - `MNAAS_INPUT_ROOT` – base directory where feed files reside.  
  - `MNAAS_OUTPUT_ROOT` – base directory for generated traffic files.  
- **Referenced Config Files**  
  - `MNAAS_create_customer_cdr_traffic_files.properties` (common settings).  
  - `MNAAS_create_customer_cdr_traffic_files_dynamic.properties` (dynamic column config).  

# Improvements
1. **Schema Validation** – Add a checksum column to each line and verify integrity during parsing.  
2. **Centralized Customer Registry** – Move customer definitions to a database table and generate this properties file at build time to eliminate manual duplication.