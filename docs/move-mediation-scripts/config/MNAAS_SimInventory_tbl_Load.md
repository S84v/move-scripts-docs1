# Summary
`MNAAS_SimInventory_tbl_Load.properties` is a Bash‑sourced configuration fragment that supplies runtime constants for the “SIM‑Inventory aggregation load” step of the MNAAS daily‑processing pipeline. It imports the shared `MNAAS_CommonProperties.properties` file and defines the process‑status file path, log file name (including date suffix), script name, and the HDFS intermediate directory used by the `MNAAS_SimInventory_tbl_Load.sh` job.

# Key Components
- **Source statement** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  Loads global environment variables and Hadoop/Hive connection settings.
- **MNAAS_Daily_SimInventory_Load_Aggr_ProcessStatusFileName** – Path to the process‑status file for the aggregation step.
- **MNAAS_DailySimInventoryLoadAggrLogPath** – Log file path with a date suffix (`_YYYY‑MM‑DD`).
- **MNAASDailySimInventoryAggregationScriptName** – Name of the executable Bash script (`MNAAS_SimInventory_tbl_Load.sh`).
- **MNAASInterFilePath_Daily_SimInventory** – HDFS intermediate directory for daily SIM‑Inventory files.

# Data Flow
| Element | Direction | Description |
|---------|-----------|-------------|
| `MNAAS_CommonProperties.properties` | Input | Provides base variables (`MNAASConfPath`, `MNAASLocalLogPath`, `MNAASInterFilePath_Daily`). |
| Process‑status file (`*.status`) | Output | Updated by `MNAAS_SimInventory_tbl_Load.sh` to indicate success/failure. |
| Log file (`*.log_YYYY‑MM‑DD`) | Output | Written by the aggregation script for audit and troubleshooting. |
| HDFS intermediate path (`/SimInventory`) | Output | Destination for temporary files generated during the load. |
| `MNAAS_SimInventory_tbl_Load.sh` | Trigger | Consumes the variables defined here to perform the aggregation load. |

Side effects: creation/modification of status and log files; write operations to HDFS intermediate directory.

# Integrations
- **Shared configuration** – `MNAAS_CommonProperties.properties` (global environment, Hadoop/Hive credentials).  
- **Processing script** – `MNAAS_SimInventory_tbl_Load.sh` (reads all variables defined in this file).  
- **HDFS** – Uses the `MNAASInterFilePath_Daily` base path to construct the SIM‑Inventory intermediate directory.  
- **Monitoring/Orchestration** – Process‑status file is polled by downstream jobs or the orchestration framework to determine pipeline progression.

# Operational Risks
- **Missing or corrupted common properties file** → pipeline aborts; mitigate with existence check before sourcing.  
- **Incorrect base paths (`MNAASConfPath`, `MNAASLocalLogPath`)** → logs/status files written to wrong location; mitigate by validating paths at startup.  
- **Date suffix generation (`date +_%F`)** may produce unexpected format on non‑GNU `date`; mitigate by using a portable date command or explicit format.  
- **HDFS permission issues on `MNAASInterFilePath_Daily_SimInventory`** → write failures; mitigate with pre‑flight HDFS ACL verification.

# Usage
```bash
# Source the configuration
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_SimInventory_tbl_Load.properties

# Verify variables
echo "$MNAAS_Daily_SimInventory_Load_Aggr_ProcessStatusFileName"
echo "$MNAAS_DailySimInventoryLoadAggrLogPath"

# Run the aggregation script (debug mode)
bash -x $MNAASDailySimInventoryAggregationScriptName
```

# Configuration
- **External config file**: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
- **Environment variables expected from common properties**:  
  - `MNAASConfPath` – directory for process‑status files.  
  - `MNAASLocalLogPath` – base directory for local logs.  
  - `MNAASInterFilePath_Daily` – base HDFS intermediate directory.  

# Improvements
1. **Add validation logic** – after sourcing, verify that all required variables are non‑empty and that target directories exist; exit with a clear error code if validation fails.  
2. **Parameterize the date format** – expose a variable (e.g., `MNAASLogDateFormat`) to allow custom log naming without modifying the script, improving portability across environments.