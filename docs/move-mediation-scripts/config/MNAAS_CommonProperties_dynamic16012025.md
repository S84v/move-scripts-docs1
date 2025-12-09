# Summary
`MNAAS_CommonProperties_dynamic16012025.properties` is a Bash‑style property script that builds in‑memory associative arrays at runtime by parsing `MNAAS_Create_New_Customers.properties`. The arrays (`MNAAS_Customer_SECS`, `FEED_CUSTOMER_MAPPING`, `MNAAS_customer_backup_dir`, `MNAAS_windows_job_mapping`, `CUSTOMER_MAPPING`) provide customer‑to‑SECSID, feed‑to‑customer, backup‑directory, Windows‑job, and classification mappings for downstream ingestion, validation, and backup jobs in the Move‑Network‑As‑A‑Service (MNAAS) production pipeline.

# Key Components
- **`MNAAS_Create_New_Customers`** – path variable pointing to the static customer definition file.  
- **`MNAAS_Customer_SECS`** – associative array `[customer]=secs_id`.  
- **`FEED_CUSTOMER_MAPPING`** – associative array `[src_feed]=" customer1 customer2 …"`; ensures each feed lists its customers once.  
- **`MNAAS_customer_backup_dir`** – associative array `[customer]=backup_dir` (currently set to the customer name).  
- **`MNAAS_windows_job_mapping`** – associative array `[customer]="Mnaas:::::customer"` used by Windows‑job orchestration.  
- **`CUSTOMER_MAPPING`** – associative array `[customer]=flag` where flag = `2` for special test customers, otherwise `0`.  
- **Loop constructs** – `for new_customer in \`cat $MNAAS_Create_New_Customers\`` and `cut` pipelines that populate the arrays.  
- **Debug flags** – `setparameter='set -x'` and `setparameter1='set -vx'` (unused in the script but available for tracing.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| Load customer list | `MNAAS_Create_New_Customers.properties` (pipe‑delimited lines: `customer|secs_id|src_feed`) | `cut`/`awk` extracts fields; Bash string manipulation builds arrays. | Populated associative arrays in the current shell environment. |
| FEED‑CUSTOMER mapping | Same input file | Checks existing mapping via `grep -iw`; appends customer if not present. | Updated `FEED_CUSTOMER_MAPPING` array; console log for duplicates. |
| Backup directory mapping | Same input file | Direct assignment of customer name to backup dir. | `MNAAS_customer_backup_dir` array. |
| Windows job mapping | Same input file | Concatenates static prefix with customer name. | `MNAAS_windows_job_mapping` array. |
| Customer classification | Same input file | Conditional logic on customer name (`Tech_Create`, `INDEPENDENT_TEST`). | `CUSTOMER_MAPPING` array. |

No external services, databases, or message queues are invoked directly; the script only reads a local properties file and writes to shell memory.

# Integrations
- **`MNAAS_CommonProperties.properties`** – sources this dynamic script to expose the generated arrays to all cron‑driven jobs.  
- **Downstream ingestion/validation scripts** – reference arrays (e.g., `${MNAAS_Customer_SECS[$customer]}`) to resolve SECS IDs, locate backup directories, and select Hive tables.  
- **Windows job orchestrator** – consumes `${MNAAS_windows_job_mapping[$customer]}` to construct job identifiers.  
- **Feed processing pipelines** – use `${FEED_CUSTOMER_MAPPING[$feed]}` to enumerate customers for a given source feed.  

# Operational Risks
- **Parsing fragility** – reliance on `cut` and pipe‑delimited format; malformed lines cause empty fields or mis‑populated arrays. *Mitigation*: validate input file schema before loops.  
- **Duplicate handling** – `grep` on the entire array string may miss duplicates when whitespace varies. *Mitigation*: normalize spacing or use Bash associative array key set.  
- **Scalability** – `for ... \`cat file\`` spawns a subshell per iteration; large customer sets increase runtime. *Mitigation*: switch to `while IFS='|' read -r ...` loop.  
- **Unused debug flags** – `setparameter` variables are defined but never invoked, leading to hidden debugging state. *Mitigation*: remove or integrate with a `DEBUG` toggle.  

# Usage
```bash
# Source the script in a Bash session to populate arrays
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties_dynamic16012025.properties

# Verify a mapping
echo "${MNAAS_Customer_SECS[AcmeCorp]}"
echo "${FEED_CUSTOMER_MAPPING[CDR_FEED]}"
```
For debugging, replace `setparameter` with `eval $setparameter` before loops.

# Configuration
- **Environment variable** `MNAAS_Create_New_Customers` – absolute path to `MNAAS_Create_New_Customers.properties`.  
- **Referenced files**  
  - `MNAAS_Create_New_Customers.properties` – pipe‑delimited list: `customer|secs_id|src_feed`.  
- **No external service credentials** required.

# Improvements
1. **Refactor parsing to a single `while IFS='|' read -r customer secs_id src_feed` loop** – eliminates multiple file reads, reduces subshell overhead, and improves error handling.  
2. **Replace string‑based duplicate detection with associative array keys** – e.g., `declare -A FEED_CUSTOMER_SET; FEED_CUSTOMER_SET["${src_feed}_${customer}"]=1` to guarantee uniqueness and simplify logic.