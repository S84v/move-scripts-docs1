# Summary
Defines all Bash variables, associative arrays, and file‑pattern mappings required by `MNAAS_create_customer_cdr_tolling_files.sh`. It supplies per‑customer column lists, column positions, output/backup directories, feed‑file patterns, and AWK filter conditions for generating tolling CDR files in the Move‑Network‑As‑A‑Service production pipeline.

# Key Components
- **Scalar variables** – script name, status file, log path, new‑customer list reference.  
- **`MNASS_Create_Customer_tolling_files_columns`** – associative array mapping each customer to the ordered list of output columns.  
- **`MNAAS_Create_Customer_tolling_files_column_position`** – maps each customer to the `$`‑field list used by `awk` to extract columns from the source feed.  
- **`MNAAS_Create_Customer_tolling_files_out_dir`** – per‑customer output directory for generated tolling files.  
- **`MNAAS_Create_Customer_tolling_files_backup_dir`** – per‑customer backup directory for processed files.  
- **`FEED_FILE_PATTERN`** – pattern strings for locating source feed files (e.g., `SNG*01.tolling`).  
- **`MNAAS_Create_Customer_tolling_files_filter_condition`** – per‑customer AWK boolean expression used to filter rows belonging to the customer.  
- **SeqNo/Timestamp check file paths** – locations of history and missing‑file tracking files for monitoring.

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| 1. Load common properties | `MNAAS_CommonProperties.properties` | `source` | environment variables | – |
| 2. Load new‑customer list | `$MNAAS_Create_New_Customers` (pipe‑separated file) | `cut` loop | dynamic entries added to associative arrays | – |
| 3. Feed discovery | Files matching patterns in `FEED_FILE_PATTERN` | `awk` with column positions & filter condition | per‑customer tolling CDR files in `*_out_dir` | original files moved to `*_backup_dir` |
| 4. Monitoring | Generated files | SeqNo/Timestamp scripts (external) | history, missing, previous file logs | – |

# Integrations
- **`MNAAS_create_customer_cdr_tolling_files.sh`** – consumes all variables defined here.  
- **Common property file** – provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`, `MNAAS_dtail_extn`, `MNAAS_Customer_SECS`).  
- **SeqNo/Timestamp check utilities** – read the paths defined at the end of this file to verify file continuity.  
- **Hadoop / HDFS** – implied by the base path `/app/hadoop_users/...`; downstream processes likely ingest generated files into Hadoop pipelines.

# Operational Risks
- **Missing new‑customer entry** – if a new customer is added to the properties file but not to the new‑customer list, arrays will be incomplete → missing output. *Mitigation*: enforce validation step that compares the new‑customer list against the associative arrays.  
- **Hard‑coded column positions** – any upstream feed schema change breaks all customers. *Mitigation*: externalize column mapping to a version‑controlled schema file.  
- **Path concatenation without quoting** – spaces in customer names could break directory creation. *Mitigation*: quote all variable expansions when used in commands.  
- **Large associative arrays in Bash** – memory consumption grows with number of customers. *Mitigation*: monitor Bash memory usage; consider moving to a more scalable language (Python/Scala) for very large customer bases.

# Usage
```bash
# Load configuration
source /path/to/MNAAS_create_customer_cdr_tolling_files.properties

# Run the processing script (debug mode)
bash -x MNAAS_create_customer_cdr_tolling_files.sh
```
To test a single customer:
```bash
export TEST_CUSTOMER=MyRep
bash -x MNAAS_create_customer_cdr_tolling_files.sh
```
Check logs:
```bash
tail -f "$MNAAS_Create_Customer_cdr_tolling_files_LogPath"
```

# Configuration
- **Environment variables** (set by common properties):  
  - `MNAASConfPath` – base config directory.  
  - `MNAASLocalLogPath` – base log directory.  
  - `MNAAS_dtail_extn['tolling']` – file extension for tolling feeds.  
  - `MNAAS_Customer_SECS` – associative array of customer SECS identifiers.  
- **Referenced files**:  
  - `MNAAS_CommonProperties.properties` – core path definitions.  
  - `MNAAS_Create_New_Customers_Non_traffic.properties` – pipe‑separated list of new customers.  
  - SeqNo/Timestamp check files under `$MNAASConfPath/MNASS_Customer_Seqno_Check/...` and `$MNAASConfPath/MNASS_Customer_Timestamp_Check/...`.

# Improvements
1. **Externalize schema** – move column lists and positions to a JSON/YAML file parsed at runtime; eliminates hard‑coded `$` strings and simplifies schema evolution.  
2. **Modular validation** – add a pre‑run validation routine that verifies: (a) every customer in the new‑customer list has entries in all associative arrays, (b) output and backup directories exist and are writable, (c) feed file patterns resolve to at least one file. This will catch configuration drift before processing begins.