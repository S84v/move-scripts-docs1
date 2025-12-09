# Summary
Defines runtime variables for the **MNAAS_Customer_SNG_files_Stat_process** Bash pipeline. It imports shared properties, sets script metadata, log and status file locations, HDFS staging paths, and Hive table identifiers used to aggregate SNG KPI file‑count statistics during a network‑move operation.

# Key Components
- **Import of common properties** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Debug toggle** – `setparameter='set -x'` (comment to enable Bash tracing)
- **Script metadata** – `MNAAS_Customer_SNG_files_Stat_process_Scriptname`
- **Process status file** – `MNAAS_Customer_SNG_files_Stat_process_ProcessStatusFileName`
- **Log file definition** – `MNAAS_Customer_SNG_files_Stat_process_logpath` (includes date suffix)
- **Local SNG KPI source directory** – `MNAAS_Customer_SNG_Stat_file_path`
- **HDFS target directory** – `MNAAS_Customer_SNG_filename_hdfs_customer`
- **Hive intermediate table names** – `customers_files_sng_inter_record_count_tblname`, `customers_files_sng_records_count_tblname`
- **Hive refresh command** – `customer_files_record_count_tblname_refresh`

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|-------------|----------------------|
| 1 | Common property file (`MNAAS_CommonProperties.properties`) | Variable sourcing | Populates environment (e.g., `MNAASConfPath`, `MNAASLocalLogPath`, `dbname`) |
| 2 | Local SNG KPI files (`/app/hadoop_users/MNAAS/SNG_KPI_Measurement`) | Bash script reads files, counts records | Writes per‑customer counts to Hive tables |
| 3 | Hive tables (`$dbname.$customers_files_sng_inter_record_count_tblname`, `$dbname.$customers_files_sng_records_count_tblname`) | `REFRESH` command executed | Ensures Hive metadata is up‑to‑date |
| 4 | HDFS path (`/user/MNAAS/FileCount/sng/customer_data`) | `hdfs dfs -put` (performed by the main script) | Persists aggregated counts for downstream consumption |
| 5 | Log file (`$MNAASLocalLogPath/...log_YYYY-MM-DD`) | Bash `echo`/`logger` statements | Auditable execution trace |
| 6 | Process status file (`$MNAASConfPath/...`) | Write status codes (START, SUCCESS, FAIL) | Enables monitoring/orchestration |

# Integrations
- **MNAAS_CommonProperties.properties** – provides base configuration (paths, DB name, Hadoop/Hive credentials).
- **MNAAS_Customer_SNG_files_Stat_process.sh** – consumes the variables defined here.
- **Hive Metastore** – referenced via `$dbname` for table creation/refresh.
- **HDFS** – target directory for aggregated KPI files.
- **External orchestration** (e.g., Oozie/Airflow) – may poll the status file to trigger downstream jobs.

# Operational Risks
- **Missing or outdated common properties** → script fails at variable resolution. *Mitigation*: Validate existence of the sourced file at start; abort with clear error.
- **Hard‑coded absolute paths** → breakage on environment changes. *Mitigation*: Externalize paths to a central config or use environment variables.
- **Date suffix in log path** evaluated at source time; log rotation may create duplicate files if script is re‑sourced. *Mitigation*: Append timestamp (`%s`) or enforce single‑run per day.
- **Debug flag left enabled in production** → verbose logs, potential credential leakage. *Mitigation*: Enforce `setparameter` comment in production CI pipeline.
- **Hive REFRESH failure** (e.g., table does not exist) → downstream queries return stale data. *Mitigation*: Guard refresh with existence check; fail fast with alert.

# Usage
```bash
# Source configuration (debug disabled)
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source move-mediation-scripts/config/MNAAS_Customer_SNG_files_Stat_process.properties

# Run the processing script
bash $MNAAS_Customer_SNG_files_Stat_process_Scriptname
```
To enable Bash tracing for debugging, comment out the `setparameter` assignment:
```bash
# setparameter='set -x'   # uncomment to enable -x
```

# Configuration
- **Environment variables required (populated by common properties)**
  - `MNAASConfPath` – directory for process status files.
  - `MNAASLocalLogPath` – base log directory.
  - `dbname` – Hive database name.
- **Referenced config files**
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`

# Improvements
1. **Parameterize file system roots** – replace hard‑coded `/app/hadoop_users/MNAAS` and `/user/MNAAS` with variables sourced from common properties to support multi‑environment deployments.
2. **Add validation block** – early‑exit checks for required directories, HDFS accessibility, and Hive table existence; emit structured error codes to the status file.