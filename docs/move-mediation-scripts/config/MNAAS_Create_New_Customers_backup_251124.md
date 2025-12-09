# Summary
Defines the static mapping of each new customer to its source SECS feed‑file glob pattern and the target Home‑Of‑Load (HOL) bucket. The file is read at runtime by the MNAAS CDR traffic‑file generation scripts to route incoming feed files to the correct processing directory during a production network move.

# Key Components
- **Customer‑to‑Feed Mapping** – `<CustomerName>|<FeedPattern>|<HOL_ID>` lines that drive file selection and destination routing.  
- **Backup Identifier** – Filename suffix `_backup_251124` indicates a snapshot used for rollback or audit.  

# Data Flow
- **Input**: Read line‑by‑line by Bash scripts (`MNAAS_create_customer_cdr_traffic_files*.sh`).  
- **Processing**: Scripts parse each line, expand `<FeedPattern>` against the inbound SECS directory, and associate matching files with `<HOL_ID>`.  
- **Output**: Generated CDR traffic files placed in the HOL‑specific staging directory; optional log entries and status files.  
- **Side Effects**: Creation of backup copies of processed source files; updates to per‑customer status logs.  
- **External Services**: File system (source SECS directory, HOL staging directories), logging subsystem, optional validation JARs invoked by the parent scripts.

# Integrations
- Consumed by **MNAAS_create_customer_cdr_traffic_files.sh** and **MNAAS_create_customer_cdr_traffic_files_crushsftp.sh** via `source` or `while IFS='|' read …` loops.  
- Provides parameters to AWK column selectors defined in `MNAAS_create_customer_cdr_traffic_files_dynamic.properties`.  
- Influences the destination path used by the SFTP upload component (CrushSFTP) when moving staged HOL files to the remote server.

# Operational Risks
- **Missing or malformed entry** → files not routed, causing data loss. *Mitigation*: schema validation script before production run.  
- **Duplicate customer rows** → ambiguous routing. *Mitigation*: enforce uniqueness check during CI validation.  
- **Glob pattern mismatch** (e.g., missing `*`) → zero files matched, job stalls. *Mitigation*: unit test pattern against a sample directory.  
- **Backup file out‑of‑date** → rollback may restore stale mappings. *Mitigation*: version control and timestamped backups.

# Usage
```bash
# Example: list customers and their HOL targets
while IFS='|' read -r cust pattern hol; do
  printf "%-25s %-30s %s\n" "$cust" "$pattern" "$hol"
done < move-mediation-scripts/config/MNAAS_Create_New_Customers_backup_251124.properties

# Debug: verify pattern expansion for a customer
cust="Vision"
pattern=$(grep "^$cust|" move-mediation-scripts/config/MNAAS_Create_New_Customers_backup_251124.properties | cut -d'|' -f2)
echo "Matching files for $cust:"
ls /data/secs/inbound/$pattern
```

# Configuration
- **File Path**: `move-mediation-scripts/config/MNAAS_Create_New_Customers_backup_251124.properties` (absolute or relative to script working directory).  
- **Environment Variables** referenced by consuming scripts:  
  - `MNAAS_INBOUND_SECS_DIR` – base directory for SECS feed files.  
  - `MNAAS_HOL_ROOT` – root directory where HOL sub‑folders (e.g., `HOL_01`) reside.  
- **Related Config Files**: `MNAAS_create_customer_cdr_traffic_files_dynamic.properties`, `MNAAS_create_customer_cdr_traffic_files_crushsftp.properties`.

# Improvements
1. **Migrate to structured format (JSON/YAML)** to enable schema validation and easier programmatic consumption.  
2. **Add checksum or version tag** within the file and implement a pre‑run integrity check to prevent accidental use of stale backup mappings.