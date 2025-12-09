# Summary
`MNAAS_CommonProperties_dynamic.properties` dynamically builds associative arrays that map customers to SECS IDs, feed‑to‑customer relationships, backup directories, Windows job mappings, and customer classification flags. It processes the static property file `MNAAS_Create_New_Customers.properties` at runtime, populating in‑memory structures used by downstream ingestion, validation, and backup scripts in the Move‑Network‑As‑A‑Service (MNAAS) production pipeline.

# Key Components
- **`declare -A MNAAS_Customer_SECS`** – associative array: key = customer name, value = SECS ID.  
- **`MNAAS_Create_New_Customers`** – path to the source property file containing pipe‑delimited records (`customer|secs_id|src_feed`).  
- **`while IFS='|' read -r customer secs_id src_feed; do … done`** – parser loop that:
  - Populates `MNAAS_Customer_SECS`.
  - Updates `FEED_CUSTOMER_MAPPING` (feed → space‑separated customer list) with duplicate detection.
  - Populates `MNAAS_customer_backup_dir` and `MNAAS_windows_job_mapping` for feeds `HOL_01`/`HOL_02`.
  - Sets `CUSTOMER_MAPPING` classification based on `new_customer` flag (`Tech_Create`, `INDEPENDENT_TEST`, default).  
- **Debug output** – `declare -p` statements printing the final associative arrays.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|-----------------------|
| 1 | `MNAAS_Create_New_Customers.properties` (file) | Line‑by‑line parsing, conditional array updates | Populated associative arrays in the current shell |
| 2 | Environment variable `new_customer` (optional) | Determines value assigned to `CUSTOMER_MAPPING` | Classification flag per customer |
| 3 | Standard output | Debug prints of all arrays | Visible logs for verification |

No external services, databases, or message queues are invoked directly; the script only reads a local file and writes to stdout.

# Integrations
- Sourced by other MNAAS cron jobs that require up‑to‑date customer‑to‑SECS mappings (e.g., Sqoop ingestion scripts, Hive table refresh jobs).  
- The associative arrays become environment variables for child processes when the script is sourced (`. MNAAS_CommonProperties_dynamic.properties`).  
- Dependent on `MNAAS_CommonProperties.properties` for shared definitions (e.g., base paths, other associative arrays like `FEED_CUSTOMER_MAPPING`).  

# Operational Risks
- **Malformed input line** – missing fields cause silent skips; may lead to incomplete mappings. *Mitigation*: validate record count and log warnings.  
- **Duplicate customer entries** – only first occurrence added to `FEED_CUSTOMER_MAPPING`; duplicates are logged but not fatal. *Mitigation*: enforce uniqueness upstream.  
- **Unset `new_customer` variable** – defaults to `'0'`, which may be unintended for certain customers. *Mitigation*: enforce explicit setting or default handling.  
- **Hard‑coded feed names (`HOL_01`, `HOL_02`)** – new feeds require script modification. *Mitigation*: externalize feed list to a config file.

# Usage
```bash
# Ensure common properties are sourced first
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Source the dynamic file to populate arrays in the current shell
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties_dynamic.properties

# Verify contents
declare -p MNAAS_Customer_SECS FEED_CUSTOMER_MAPPING \
           MNAAS_customer_backup_dir MNAAS_windows_job_mapping CUSTOMER_MAPPING
```
For debugging, run the script in a subshell to capture output:
```bash
bash -c ". /path/to/MNAAS_CommonProperties_dynamic.properties"
```

# Configuration
- **`MNAAS_Create_New_Customers`** – absolute path to the source property file (default: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Create_New_Customers.properties`).  
- **Environment variable `new_customer`** – optional flag influencing `CUSTOMER_MAPPING` values.  
- **Dependent config** – `MNAAS_CommonProperties.properties` (must be sourced beforehand).

# Improvements
1. **Input validation** – add checks for field count and permissible values; exit with non‑zero status on fatal errors.  
2. **Externalize feed list** – move `HOL_01`/`HOL_02` definitions to a configurable array to simplify onboarding of new feeds.