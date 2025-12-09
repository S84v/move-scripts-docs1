# Summary
Defines runtime variables, associative arrays, and file‑pattern mappings for the Bash script **MNAAS_create_customer_cdr_traffic_files_crushsftp.sh**. Supplies per‑customer column lists, column‑position selectors, output/backup directories, feed‑file glob patterns, and AWK filter expressions used to generate, filter, and stage traffic CDR files during a Move‑Network‑As‑A‑Service production move.

# Key Components
- **Scalar variables**
  - `MNAAS_Create_Customer_cdr_traffic_scriptname` – script filename.
  - `MNAAS_Create_Customer_cdr_traffic_files_ProcessStatusFilename` – path to process‑status file.
  - `MNAAS_Create_Customer_cdr_traffic_files_LogPath` – log file path (timestamped).
  - `MNAAS_Create_New_Customers` – reference to new‑customer property list.
- **Associative arrays**
  - `MNASS_Create_Customer_traffic_files_columns` – maps customer → semicolon‑delimited column list.
  - `MNAAS_Create_Customer_traffic_files_column_position` – maps customer → comma‑separated column positions in source file.
  - `MNAAS_Create_Customer_traffic_files_out_dir` – maps customer → relative output directory.
  - `FEED_FILE_PATTERN` – maps feed‑type keys (e.g., `SNG_01`) → filename glob pattern using `MNAAS_dtail_extn`.
  - `MNAAS_Create_Customer_traffic_files_filter_condition` – maps customer → AWK filter expression (includes formatting of `$45` and row validation).
- **Sourced common properties**
  - `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|------|-------|------------|----------------------|
| **Source** | Common properties file, `MNAAS_Create_New_Customers.properties`, raw feed files matching patterns in `FEED_FILE_PATTERN` | Bash reads variables; associative arrays drive AWK commands | Populated environment for the main script |
| **Transformation** | Raw feed files (e.g., `SNG*01*.traffic`) | AWK uses column positions and filter condition per customer to extract/format fields | Customer‑specific traffic CDR files written to `MNAAS_Create_Customer_traffic_files_out_dir` |
| **Logging** | Script execution | Writes operational messages to `MNAAS_Create_Customer_cdr_traffic_files_LogPath` | Log file |
| **Status** | Script execution result | Writes status (success/failure) to `MNAAS_Create_Customer_cdr_traffic_files_ProcessStatusFilename` | Status file |
| **Backup/Archive** (implicit) | Generated CDR files | May be moved to backup locations by downstream processes | Not defined in this file |

# Integrations
- **MNAAS_CommonProperties.properties** – provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`, `MNAAS_dtail_extn`, `MNAAS_Customer_SECS`).
- **MNAAS_create_customer_cdr_traffic_files_crushsftp.sh** – consumes all variables/arrays defined here.
- **MNAAS_Create_New_Customers.properties** – external list of customers to be processed.
- Downstream validation JARs or Hadoop jobs (not referenced directly but typical in the pipeline) consume the generated traffic files.

# Operational Risks
- **Hard‑coded customer entries** – new customers require manual edit; risk of omission. *Mitigation*: automate population from `MNAAS_Create_New_Customers.properties`.
- **Pattern mismatches** – feed‑file glob relies on `MNAAS_dtail_extn` values; incorrect extension leads to missed files. *Mitigation*: validate extensions at startup.
- **AWK filter syntax errors** – embedded quotes and variable expansion can break if customer SECS value contains special characters. *Mitigation*: sanitize `MNAAS_Customer_SECS` values or use safer string handling.
- **Log path timestamp** – `$(date +_%F)` evaluated at source time; if script is sourced multiple times, log file may change unexpectedly. *Mitigation*: compute once in the main script.

# Usage
```bash
# Ensure common properties are accessible
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Source this configuration
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_create_customer_cdr_traffic_files_crushsftp.properties

# Execute the processing script (run as the appropriate Hadoop user)
bash /app/hadoop_users/MNAAS/MNAAS_create_customer_cdr_traffic_files_crushsftp.sh
```
To debug, enable Bash tracing:
```bash
set -x
bash .../MNAAS_create_customer_cdr_traffic_files_crushsftp.sh
```

# Configuration
- **Environment variables** (provided by common properties):
  - `MNAASConfPath`
  - `MNAASLocalLogPath`
  - `MNAAS_dtail_extn` (associative array, e.g., `['traffic']`)
  - `MNAAS_Customer_SECS` (associative array of customer security identifiers)
- **Referenced config files**
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Create_New_Customers.properties`

# Improvements
1. **Externalize customer mappings** – generate associative arrays from `MNAAS_Create_New_Customers.properties` to eliminate manual edits.
2. **Add validation step** – script should verify that each `FEED_FILE_PATTERN` resolves to at least one file before processing, and abort with a clear error if not.