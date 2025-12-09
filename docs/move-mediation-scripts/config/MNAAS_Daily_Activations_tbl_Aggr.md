# Summary
`MNAAS_Daily_Activations_tbl_Aggr.properties` supplies runtime constants for the **MNAAS_Daily_Activations_tbl_Aggr** Bash pipeline. It imports shared MNAAS properties and defines file‑system locations, script names, and control‑file references used during daily activation table aggregation in a network‑move production workflow.

# Key Components
- **Import of common properties** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
- **Process status file** – `MNAAS_Daily_Activations_Aggr_ProcessStatusFileName` – path to the status flag file.  
- **Log file path** – `MNAASDailyActivationsAggrLogPath` – local log file with date suffix.  
- **Business‑trend script name** – `MNAASbusinessTrendAggrScriptName` – invoked to aggregate business‑trend metrics.  
- **Activations aggregation script name** – `MNAASDailyActivationsAggregationScriptName` – primary driver for loading activation data into Hive.  
- **Control file** – `MNAAS_Daily_Activations_AggrCntrlFileName` – properties file that controls job parameters (e.g., date range, partitions).  
- **Pipeline entry script** – `MNAASDailyActivationsAggrScriptName` – the executable Bash script that orchestrates the aggregation.

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| 1 | `MNAAS_Daily_Activations_AggrCntrlFile.properties` (control file) | Bash driver reads parameters, sets environment | Variables for Hive/Sqoop jobs | Updates `MNAAS_Daily_Activations_Aggr_ProcessStatusFile` |
| 2 | Source tables in HDFS/Hive (activation raw data) | `MNAAS_Activations_tbl_Load.sh` runs Sqoop/Hive INSERT/OVERWRITE | Aggregated Hive table `daily_activations` | Writes to HDFS staging path (defined in common properties) |
| 3 | Business‑trend source files | `MNAAS_business_trend_Aggr.sh` computes KPI aggregates | Trend Hive tables / HDFS files | Logs to `MNAASDailyActivationsAggrLogPath` |
| 4 | Completion | Status file updated to SUCCESS/FAIL | N/A | Log rotation, possible alert trigger |

# Integrations
- **Common property library** (`MNAAS_CommonProperties.properties`) – provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`, etc.).  
- **Shell scripts** `MNAAS_Activations_tbl_Load.sh` and `MNAAS_business_trend_Aggr.sh` – invoked by the main aggregation script.  
- **Hive/HDFS** – target storage for aggregated tables.  
- **Sqoop** – used within `MNAAS_Activations_tbl_Load.sh` to import from relational source.  
- **Process status monitor** – external watchdog reads `MNAAS_Daily_Activations_Aggr_ProcessStatusFile`.

# Operational Risks
- **Stale control file** – outdated parameters cause incorrect date ranges. *Mitigation*: validate control file timestamps before execution.  
- **Log file growth** – unbounded log size due to daily appends. *Mitigation*: implement log rotation or size‑based truncation.  
- **Missing common properties** – failure to source shared file aborts pipeline. *Mitigation*: add pre‑flight check for file existence.  
- **HDFS permission changes** – could prevent staging writes. *Mitigation*: enforce ACLs and monitor permission drift.  

# Usage
```bash
# Source properties
. /app/hadoop_users/MNAAS/move-mediation-scripts/config/MNAAS_Daily_Activations_tbl_Aggr.properties

# Execute the aggregation pipeline
bash $MNAASDailyActivationsAggrScriptName   # resolves to MNAAS_Daily_Activations_tbl_Aggr.sh

# Debug (enable verbose tracing)
set -x   # or modify setparameter in the script if present
```

# Configuration
- **Environment variables** defined in `MNAAS_CommonProperties.properties`:  
  - `MNAASConfPath` – directory for control and status files.  
  - `MNAASLocalLogPath` – base directory for log files.  
  - Additional Hadoop/Hive connection variables (e.g., `HIVE_HOME`, `SQOOP_HOME`).  
- **Referenced config files**:  
  - `MNAAS_Daily_Activations_AggrCntrlFile.properties` (control parameters).  
  - `MNAAS_Daily_Activations_Aggr_ProcessStatusFile` (runtime status flag).  

# Improvements
1. **Parameter validation module** – add a function to verify existence and format of all referenced files/paths before pipeline start.  
2. **Centralized logging wrapper** – replace raw `echo >> $log` with a logger that handles rotation, severity levels, and integrates with the existing monitoring/alerting framework.