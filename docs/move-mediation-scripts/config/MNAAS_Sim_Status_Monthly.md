# Summary
Bash‑sourced configuration for the **MNAAS Sim‑Status Monthly** processing step. It imports global properties, defines script metadata (status file, log path, script name), Hive/Impala connection parameters, target table names, and partition‑file locations used by `MNAAS_Sim_Status_Monthly.sh` in the daily‑processing pipeline.

# Key Components
- **Source statement** – loads shared variables from `MNAAS_CommonProperties.properties`.
- **Debug toggle** – `setparameter='set -x'` (comment to disable Bash tracing).
- **Process‑status file** – `MNAAS_Sim_Status_Monthly_ProcessStatusFileName`.
- **Script identifier** – `MNAAS_Sim_Status_Monthly_ScriptName`.
- **Log file definition** – `MNAAS_Sim_Status_MonthlyLogPath` (includes current date).
- **Hive table identifiers** – `Dname_MNAAS_Sim_Status_Monthly_tbl`, `Insert_Monthly_Status_table`, `Dname_move_sim_inventory_status_monthly_tbl`, `Insert_sim_inventory_Monthly_Status_table`.
- **Partition‑file paths** – `sim_status_monthly_Partitions_FileName`, `sim_status_inventory_monthly_Partitions_FileName`.
- **Process names** – `processname_sim_status_monthly`, `processname_sim_inventory_status_monthly`.
- **Impala connection** – `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`.
- **Hive connection** – `HIVE_HOST`, `HIVE_JDBC_PORT`.
- **Database & table names** – `dbname`, `actives_daily_tblname`, `sim_status_monthly_tblname`, `Move_siminventory_status_tblname`, `sim_inventory_status_monthly_tblname`.

# Data Flow
| Element | Direction | Description |
|---------|-----------|-------------|
| **Input** | Reads | Global vars from `MNAAS_CommonProperties.properties`; environment vars `MNAASConfPath`, `MNAASLocalLogPath`. |
| **Processing** | Consumed by | `MNAAS_Sim_Status_Monthly.sh` for status aggregation, Hive/Impala INSERT statements, and partition file generation. |
| **Output** | Writes | Process‑status file, dated log file, Hive tables (`SimStatusMonthly`, `SimInventoryStatusMonthly`). |
| **Side Effects** | External services | Hive metastore (via `HIVE_HOST:10000`), Impala daemon (via `IMPALAD_HOST:21051`), HDFS paths for partition files. |

# Integrations
- **`MNAAS_Sim_Status_Monthly.sh`** – sources this file to obtain all runtime constants.
- **Other config fragments** (`MNAAS_SimInventory_tbl_Load.properties`, etc.) – share the same common properties file, enabling consistent host/credential definitions.
- **Hive/Impala** – used for INSERT operations defined elsewhere; table names referenced here must exist in the `mnaas` database.
- **HDFS** – partition files referenced by `*_Partitions_FileName` are read/written by the shell script.

# Operational Risks
- **Hard‑coded hostnames/IPs** – changes in cluster topology require manual edit.
- **Debug flag exposure** – leaving `set -x` enabled in production may flood logs with sensitive data.
- **Static JDBC ports** – mismatches with actual service ports cause connection failures.
- **Missing credential handling** – passwords are not defined; reliance on external mechanisms may break silently.
- **Date‑dependent log filename** – if the script runs across midnight, log rotation may split a single run’s output.

# Usage
```bash
# Enable Bash tracing (optional)
# comment out the line below to disable
setparameter='set -x'

# Source the configuration
. /path/to/MNAAS_Sim_Status_Monthly.properties

# Execute the processing script
bash $MNAAS_Sim_Status_Monthly_ScriptName
```
To debug, ensure `setparameter` is active; the script will emit each command as it executes.

# Configuration
- **Referenced file**: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Environment variables required** (provided by common properties):
  - `MNAASConfPath` – directory for configuration files.
  - `MNAASLocalLogPath` – base directory for log files.
- **Adjustable parameters**:
  - `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`
  - `HIVE_HOST`, `HIVE_JDBC_PORT`
  - Table names and partition‑file names as needed.

# Improvements
1. **Externalize host/port definitions** to a secure secrets manager or environment‑specific property file to avoid hard‑coding.
2. **Add validation logic** (e.g., test Hive/Impala connectivity) at load time and fail fast if required services are unreachable.