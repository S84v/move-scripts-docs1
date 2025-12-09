# Summary
Defines a static mapping of new customers to their source SECS feed‑file glob patterns and the target Home‑Of‑Load (HOL) bucket (HOL_01). The file is read at runtime by the MNAAS CDR traffic‑file generation pipeline to route incoming feed files to the correct processing directory during a production network move.

# Key Components
- **Customer‑to‑Feed Mapping entries** – `<CustomerName>|<FeedPattern>|<HOL_ID>` lines used by Bash/awk scripts to select source files and assign them to a HOL bucket.  
- **Pattern tokens** – `*` wildcard for globbing; plain identifiers (e.g., `70862`) treated as exact file name prefixes.  

# Data Flow
| Stage | Description |
|-------|-------------|
| **Input** | `MNAAS_Create_New_Customers_Non_traffic.properties` read by `MNAAS_create_customer_cdr_traffic_files.sh` (or related scripts). |
| **Processing** | Script parses each line, builds associative arrays: `customer→feedPattern`, `customer→holId`. Uses the feed pattern to glob files in the inbound SECS directory. |
| **Output** | For each matched file: moves/copies file to `<HOL_ROOT>/<HOL_ID>/` and logs the action to the job status file. |
| **Side Effects** | File system changes (move/copy), status‑file updates, optional audit log entries. |
| **External Services** | None; pure file‑system operations. No DB or message‑queue interaction. |

# Integrations
- **`MNAAS_create_customer_cdr_traffic_files.sh`** – primary consumer; loads this properties file via `source` or custom parser.  
- **`MNAAS_create_customer_cdr_traffic_files_dynamic.properties`** – provides complementary runtime variables (log paths, validation JARs) referenced by the same Bash job.  
- **Validation JARs** – invoked after file routing to verify CDR format; paths defined in the dynamic properties file.  

# Operational Risks
- **Incorrect glob pattern** – may miss files or select unintended ones; mitigate by unit‑testing patterns against a staging directory.  
- **Duplicate customer entries** – could cause race conditions when moving files; enforce uniqueness via a pre‑deployment lint script.  
- **Hard‑coded HOL bucket** (`HOL_01`) limits scalability; future moves to multiple HOLs require property redesign.  

# Usage
```bash
# Example: run the traffic‑file generation job in debug mode
export DEBUG=true
bash move-mediation-scripts/MNAAS_create_customer_cdr_traffic_files.sh \
    -c config/MNAAS_Create_New_Customers_Non_traffic.properties \
    -d /data/secs/inbound \
    -o /data/hol
```
*Inspect log `/var/log/mnaas/create_customer_cdr.log` for mapping results.*

# Configuration
- **Environment Variables**  
  - `MNAAS_PROP_FILE` – path to this properties file (default `config/MNAAS_Create_New_Customers_Non_traffic.properties`).  
  - `SECS_INBOUND_DIR` – directory scanned for source files.  
  - `HOL_ROOT_DIR` – base directory for HOL buckets.  
- **Referenced Config Files**  
  - `config/MNAAS_create_customer_cdr_traffic_files_dynamic.properties` – runtime scalars, validation JAR locations.  

# Improvements
1. **Parameterise HOL identifier** – replace hard‑coded `HOL_01` with a column in the properties file to support multi‑HOL deployments.  
2. **Add validation schema** – embed a checksum or version tag per line and implement a pre‑run validator to detect malformed entries early.