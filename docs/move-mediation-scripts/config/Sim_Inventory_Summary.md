# Summary
`Sim_Inventory_Summary.properties` is a Bash‑style configuration wrapper for the Move mediation pipeline’s SIM inventory summary job. It sources the global `MNAAS_CommonProperties.properties` file and defines all environment‑specific variables required by the `Sim_Inventory_Summary.sh` driver script, including status‑file location, log file path, Hive refresh command, target table name, and the SQL statements that populate the `move_sim_inventory_summary_tbl` partition for the current and previous month.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – imports shared paths and common variables.  
- **`setparameter='set -x'`** – optional Bash debugging flag (commented to disable).  
- **`MNAAS_Sim_Inventory_Summary_ProcessStatusFileName`** – HDFS path for the job status file.  
- **`MNAAS_Sim_Inventory_Summary_logpath`** – local log file name with date suffix.  
- **`MNAAS_Sim_Inventory_Summary_Script`** – driver script name (`Sim_Inventory_Summary.sh`).  
- **`MNAAS_Sim_Inventory_Summary_Refresh`** – Hive `refresh` command for the target table.  
- **`MNAAS_Sim_Inventory_Summary_tblname`** – logical name of the Hive table.  
- **`MNAAS_Sim_Inventory_Summary_Query`** – Hive `INSERT … SELECT` for the current month partition.  
- **`MNAAS_Sim_Inventory_Summary_Prev_Query`** – Hive `INSERT … SELECT` for the previous month partition.

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| **Configuration Load** | `MNAAS_CommonProperties.properties` | Variable substitution | In‑memory Bash variables | None |
| **Driver Execution** (`Sim_Inventory_Summary.sh`) | Variables above, HDFS/Hive environment | 1. Create/overwrite status file 2. Execute `MNAAS_Sim_Inventory_Summary_Query` (current month) 3. Execute `MNAAS_Sim_Inventory_Summary_Prev_Query` (previous month) 4. Run Hive `refresh` | Updated `move_sim_inventory_summary_tbl` partitions, status file, log file | Hive table writes, HDFS status file creation, log file append |
| **Logging** | Bash stdout/stderr | Redirected to `$MNAAS_Sim_Inventory_Summary_logpath` | Log file with timestamp | Log growth; requires rotation |

# Integrations
- **Global Config**: `MNAAS_CommonProperties.properties` supplies `MNAASConfPath` and `MNAASLocalLogPath`.  
- **Driver Script**: `Sim_Inventory_Summary.sh` sources this file and uses the defined variables to invoke Hive CLI / Beeline.  
- **Hive**: Executes the two INSERT statements and the `refresh` command against the `mnaas` database.  
- **HDFS**: Writes the process status file to `$MNAASConfPath`.  
- **Scheduler**: Typically invoked by an Oozie or cron job that runs the driver script on a monthly schedule.

# Operational Risks
- **Missing Common Properties** – job fails early; mitigate by validating file existence before sourcing.  
- **Date Formatting Mismatch** – `date +_%F` may produce unexpected suffix if locale changes; enforce `LC_ALL=C`.  
- **Log File Bloat** – unbounded growth; implement log rotation or size‑based truncation.  
- **Hive Partition Drift** – if Hive `date_format` pattern changes, partitions may be mis‑aligned; lock version of Hive syntax.  
- **Debug Flag Exposure** – leaving `set -x` enabled in production can leak credentials; ensure it is commented out in prod.

# Usage
```bash
# Source the configuration (normally done inside the driver)
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/Sim_Inventory_Summary.properties

# Run the driver script (example)
bash /app/hadoop_users/MNAAS/move-mediation-scripts/Sim_Inventory_Summary.sh

# To enable Bash tracing for debugging, comment out the setparameter line:
# setparameter='set -x'
# Then re‑source and re‑run the driver.
```

# Configuration
- **External Files**  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` – defines `MNAASConfPath`, `MNAASLocalLogPath`, and other shared settings.  
- **Environment Variables (inherited from common file)**  
  - `MNAASConfPath` – base HDFS config directory.  
  - `MNAASLocalLogPath` – base local log directory.  
- **Local Variables Defined Here** (see *Key Components*).  

# Improvements
1. **Parameterize Date Logic** – expose current/previous month as configurable variables to simplify testing and allow back‑fill runs.  
2. **Implement Log Rotation** – add a pre‑run step that archives or truncates `$MNAAS_Sim_Inventory_Summary_logpath` when size exceeds a threshold.  