# Summary
`MNAAS_Missing_Customer_Files.properties` supplies runtime constants for the **Missing Customer Files** batch job in the Move‑Mediation production environment. It imports common property definitions, then defines the script identifier, log‑file location, process‑status file, query‑output file, and static arrays of customer identifiers used by `MNAAS_Missing_Customer_Files.sh` to detect absent inbound files for each customer and processing type.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – imports shared environment variables (e.g., `MNAASLocalLogPath`, `MNAASConfPath`).  
- **`MNAAS_Missing_Customer_Files_Scriptname`** – name of the executable shell script.  
- **`MNAAS_Missing_Customer_Files_logpath`** – full path for the daily log file (date‑suffixed).  
- **`MNAAS_Missing_Customer_Files_ProcessStatusFilename`** – path to the process‑status flag file used by the orchestration layer.  
- **`query_output`** – destination file for the generated list of missing files.  
- **`customer_names` (array)** – primary customer identifiers scanned by the job.  
- **`infonova_customer_names` (array)** – secondary customer identifiers (Infonova partner set).  
- **`process_list` (array)** – processing categories (`traffic`, `actives`, `activations`, `tolling`) applied to each customer.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|------|-------|------------|----------------------|
| **Initialization** | `MNAAS_CommonProperties.properties` (environment variables) | Source file, expand variables (`$MNAASLocalLogPath`, `$MNAASConfPath`) | Populated Bash variables |
| **Execution (via `MNAAS_Missing_Customer_Files.sh`)** | Filesystem directories for each `<customer>/<process>` pair | Loop over `customer_names` + `infonova_customer_names` and `process_list`; check presence of expected inbound files; record missing entries | Append entries to `$query_output`; write operational messages to `$MNAAS_Missing_Customer_Files_logpath`; update `$MNAAS_Missing_Customer_Files_ProcessStatusFilename` (e.g., SUCCESS/FAIL) |
| **Post‑run** | – | – | Log rotation handled externally; status file consumed by monitoring/alerting system |

# Integrations
- **Common Properties** – `MNAAS_CommonProperties.properties` provides base paths, Hadoop/Hive configuration, and generic job settings.  
- **Shell Script** – `MNAAS_Missing_Customer_Files.sh` sources this properties file (`source …/MNAAS_Missing_Customer_Files.properties`) and implements the file‑existence logic.  
- **Monitoring/Orchestration** – Process‑status file is polled by the Move‑Mediation scheduler (e.g., Oozie, Airflow) to determine job completion.  
- **Logging Infrastructure** – Log file path integrates with the central log aggregation system (e.g., Splunk, ELK).  

# Operational Risks
- **Hard‑coded customer arrays** – New customers require manual property updates; risk of omission. *Mitigation*: externalize to a database or centrally managed config service.  
- **Path resolution failures** – If `MNAASLocalLogPath` or `MNAASConfPath` are mis‑set, log and status files are not created, breaking monitoring. *Mitigation*: add validation at script start, fail fast with clear error.  
- **Date suffix format** – `$(date +_%F)` may produce locale‑dependent output; inconsistent naming across nodes. *Mitigation*: enforce `LC_ALL=C` or use ISO‑8601 format explicitly.  
- **File‑system latency** – Late arriving files may be incorrectly flagged as missing. *Mitigation*: incorporate configurable grace period before evaluation.  

# Usage
```bash
# From the job’s working directory
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Missing_Customer_Files.properties
bash $MNAAS_Missing_Customer_Files_Scriptname   # or ./MNAAS_Missing_Customer_Files.sh
# Debug: enable Bash tracing
bash -x $MNAAS_Missing_Customer_Files_Scriptname
```
The script will generate `$query_output` and write a log entry to `$MNAAS_Missing_Customer_Files_logpath`.

# Configuration
- **Environment variables (provided by `MNAAS_CommonProperties.properties`)**  
  - `MNAASLocalLogPath` – base directory for job logs.  
  - `MNAASConfPath` – base directory for configuration and status files.  
- **Referenced files**  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` (must be present and readable).  
- **Local overrides** – can be set before sourcing, e.g., `export MNAASLocalLogPath=/var/log/mnaas`.

# Improvements
1. **Externalize customer lists** – Store `customer_names`, `infonova_customer_names`, and `process_list` in a Hive table or configuration service (e.g., ZooKeeper, Consul) and fetch at runtime to eliminate code changes for new customers.  
2. **Add pre‑run validation** – Implement a function in `MNAAS_Missing_Customer_Files.sh` that verifies all required directories and variables exist, exits with a distinct status code, and writes a diagnostic entry to the process‑status file.