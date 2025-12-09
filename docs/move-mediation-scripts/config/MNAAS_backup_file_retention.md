# Summary
Defines environment‑specific constants for the **MNAAS backup‑file‑retention** job.  When sourced by `MNAAS_backup_file_retention.sh` it supplies script metadata, log and status file locations, edge‑node backup parameters, per‑customer retention windows, file‑type patterns, and per‑customer edge‑2 backup directories.  The script also dynamically extends the customer maps with entries from `MNAAS_Create_New_Customers.properties`.

# Key Components
- **Sourced common properties** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
- **Script metadata** – `MNAAS_backup_file_retention_scriptName`, `MNAAS_file_retention_period_logpath`, `MNAAS_file_retention_period_ProcessStatusFileName`  
- **Edge‑node configuration** – `MNAAS_edge2_backup_path`, `MNAAS_edge2_user_name`, `MNAAS_edge_server_name`  
- **Customer list reference** – `MNAAS_Create_New_Customers` (properties file)  
- **Associative array `MNAAS_file_retention_period`** – default 30‑day retention per known customer  
- **Associative array `MNAAS_dtail_extn`** – glob patterns for file‑type classification (traffic, tolling, etc.)  
- **Associative array `MNAAS_edge2_customer_backup_dir`** – target sub‑directory per customer on edge‑2 storage  
- **Dynamic augmentation loops** – append new customers from `MNAAS_Create_New_Customers` to the two associative arrays.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| Source common props | `MNAAS_CommonProperties.properties` | Populate base variables (`MNAASLocalLogPath`, `MNAASConfPath`, …) | Global env vars |
| Read new‑customer list | `MNAAS_Create_New_Customers.properties` (pipe‑delimited) | `cut -d'|' -f1` → loop | Adds entries to `MNAAS_file_retention_period` & `MNAAS_edge2_customer_backup_dir` |
| Export variables | All defined vars & arrays | Exported to calling shell | Consumed by `MNAAS_backup_file_retention.sh` for: <br>• Log file creation <br>• Status‑file updates <br>• HDFS/Hadoop file‑retention actions <br>• Edge‑2 backup directory resolution |
| No direct DB/queue interaction | – | – | All I/O performed by downstream script (HDFS commands, Hive/Impala queries, etc.) |

# Integrations
- **`MNAAS_backup_file_retention.sh`** – primary driver; sources this file to obtain configuration.  
- **`MNAAS_backup_files_process.sh`** – shares status‑file naming convention (`MNAAS_backup_files_process_ProcessStatusFile`).  
- **`MNAAS_Create_New_Customers.properties`** – external list of customers to be added at runtime.  
- **`MNAAS_CommonProperties.properties`** – provides base paths (`MNAASLocalLogPath`, `MNAASConfPath`) used throughout the Move‑Mediation pipeline.  

# Operational Risks
- **Missing or malformed `MNAAS_Create_New_Customers.properties`** → loops fail, new customers not added; *mitigation*: validate file existence and format before sourcing.  
- **Hard‑coded 30‑day retention** may be unsuitable for some customers; *mitigation*: allow per‑customer override via external config.  
- **Edge server name mismatch** (`MNAAS_edge_server_name`) could cause HDFS copy failures; *mitigation*: centralize host name in common properties and enforce consistency checks.  
- **Log path concatenation uses `$(date +_%F)` without separator** → potential filename collision if `MNAASLocalLogPath` already ends with a slash; *mitigation*: enforce trailing slash handling or use `${MNAASLocalLogPath}/MNAAS_backup_file_retention.log$(date +_%F)`.  
- **Associative array expansion in Bash 3.x** (unsupported) → script will abort; *mitigation*: enforce Bash ≥4 on all nodes.

# Usage
```bash
# Load configuration in a Bash session
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_backup_file_retention.properties

# Verify loaded variables
echo "Log path: $MNAAS_file_retention_period_logpath"
declare -p MNAAS_file_retention_period
declare -p MNAAS_edge2_customer_backup_dir

# Run the driver (debug mode)
bash -x MNAAS_backup_file_retention.sh
```

# Configuration
- **External files**  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Create_New_Customers.properties`  
- **Environment variables expected from common properties**  
  - `MNAASLocalLogPath` – base directory for local logs  
  - `MNAASConfPath` – directory for status‑file artifacts  
- **Hard‑coded values** (can be overridden in common properties)  
  - `MNAAS_edge2_backup_path=/backup1/MNAAS/Customer_CDR`  
  - `MNAAS_edge2_user_name=hdfs`  
  - `MNAAS_edge_server_name=camttppap002.intl.vsnl.co.in`

# Improvements
1. **Externalize associative arrays** to a structured format (JSON/YAML) and load via `jq` or `yq` to simplify maintenance and enable per‑customer retention overrides.  
2.