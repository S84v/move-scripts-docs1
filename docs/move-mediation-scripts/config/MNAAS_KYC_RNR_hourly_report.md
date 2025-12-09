# Summary
Defines runtime constants for the **KYC RNR hourly report** batch job. Supplies the process‑status file location, aggregated log file name, script identifier, target Hive table, Hive DDL script, and the Hive `REFRESH` command used to make the newly loaded data visible to subsequent queries in the Move‑Mediation production environment.

# Key Components
- **`MNAAS_KYC_RNR_hourly_report_ProcessStatusFileName`** – full path to the job’s process‑status file.  
- **`MNAAS_KYC_RNR_hourly_report_AggrLogPath`** – path to the hourly aggregation log, suffixed with the execution date.  
- **`MNAAS_KYC_RNR_hourly_report_Script`** – name of the shell driver script (`MNAAS_KYC_RNR_hourly_report.sh`).  
- **`MNAAS_sim_inventory_kyc_rnr_table`** – Hive table name (`move_kyc_sim_inventory_rnr_data`).  
- **`MNAAS_sim_inventory_kyc_rnr_script`** – Hive DDL/ETL script (`move_kyc_sim_inventory_rnr_data.hql`).  
- **`msisdn_sim_inventory_kyc_rnr_table_refresh`** – Hive `REFRESH` statement to invalidate metadata cache for the target table.  
- **`setparameter`** – optional Bash debugging flag (`set -x`).  

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| 1 | Raw KYC RNR source files (HDFS / external feed) | `MNAAS_KYC_RNR_hourly_report.sh` invokes Hive with `move_kyc_sim_inventory_rnr_data.hql` | Populated Hive table `move_kyc_sim_inventory_rnr_data` | Writes to `$MNAAS_KYC_RNR_hourly_report_AggrLogPath` |
| 2 | Completed Hive load | Executes `REFRESH move_kyc_sim_inventory_rnr_data` via `$msisdn_sim_inventory_kyc_rnr_table_refresh` | Updated Hive metadata cache | Updates `$MNAAS_KYC_RNR_hourly_report_ProcessStatusFileName` with success/failure status |
| 3 | Optional downstream jobs | Read from refreshed Hive table | Further analytics / reports | None |

External services: Hive metastore, HDFS, optional email notification (via common properties).

# Integrations
- **`MNAAS_CommonProperties.properties`** – provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`, `$dbname`).  
- **`MNAAS_KYC_RNR_hourly_report.sh`** – driver script that sources this properties file.  
- **Hive** – executes the `.hql` script and the `REFRESH` command.  
- **Process‑status framework** – shared across Move‑Mediation jobs for monitoring and alerting.  

# Operational Risks
- Missing or incorrect `MNAASConfPath` / `MNAASLocalLogPath` leads to file‑not‑found errors.  
- Log file growth without rotation may exhaust disk space.  
- Failure of the Hive `REFRESH` leaves stale metadata, causing downstream jobs to read stale data.  
- Concurrent executions could race on the same process‑status file.  

Mitigations: validate environment variables at script start; implement log rotation; enforce job locking; capture Hive exit codes and abort on non‑zero status.

# Usage
```bash
# Source common properties first
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Source this job‑specific properties file
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_KYC_RNR_hourly_report.properties

# Run the driver script (debug mode enabled by commenting out setparameter)
bash $MNAAS_KYC_RNR_hourly_report_Script
```
To enable Bash tracing, comment out the line `setparameter='set -x'` and re‑source the file.

# Configuration
- **Environment variables** required from `MNAAS_CommonProperties.properties`:  
  - `MNAASConfPath` – directory for process‑status files.  
  - `MNAASLocalLogPath` – base directory for log files.  
  - `dbname` – Hive database name used in the `REFRESH` statement.  
- **External config files**: `MNAAS_CommonProperties.properties`, `move_kyc_sim_inventory_rnr_data.hql`.  

# Improvements
1. **Externalize paths** – move `MNAASConfPath`, `MNAASLocalLogPath`, and `dbname` into a dedicated job‑level config to avoid cross‑job coupling.  
2. **Add validation logic** – script should verify existence and write permissions of all referenced files/directories before execution and exit with a clear error code if any check fails.