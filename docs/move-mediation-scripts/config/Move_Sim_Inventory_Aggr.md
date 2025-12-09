# Summary
The `Move_Sim_Inventory_Aggr.properties` file supplies Bash‑sourced runtime constants for the **Move_Sim_Inventory_Aggr** job, which aggregates simulated inventory status data. It imports the shared `MNAAS_CommonProperties.properties` file, then defines the process‑status file location, the job‑specific log file, and the driver script name. These constants are consumed by the `Move_Sim_Inventory_Aggr.sh` driver to coordinate status tracking, logging, and execution of the aggregation logic in production.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – Sources global MNAAS environment variables (e.g., `MNAASConfPath`, `MNAASLocalLogPath`).
- **`MNAAS_SimInventory_Status_Aggr_ProcessStatusFileName`** – Fully‑qualified path to the process‑status file used for idempotency and monitoring.
- **`sim_inventory_status_Aggr_Log`** – Path to the job‑specific log file for audit and troubleshooting.
- **`MNAASSimInventoryStatusAggrScriptName`** – Name of the driver shell script (`Move_Sim_Inventory_Aggr.sh`) that implements the aggregation workflow.

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| **Initialization** | `MNAAS_CommonProperties.properties` | Sources global variables (`MNAASConfPath`, `MNAASLocalLogPath`). | Environment variables populated. | None |
| **Status Tracking** | N/A (status file path defined) | Driver script reads/writes `MNAAS_SimInventory_Status_Aggr_ProcessStatusFileName` to record start, success, or failure timestamps. | Updated status file. | Persistent state for downstream jobs. |
| **Logging** | N/A (log path defined) | Driver script redirects stdout/stderr to `sim_inventory_status_Aggr_Log`. | Log file with execution details. | Log rotation/retention policies apply. |
| **Aggregation** | Source data files (HDFS paths defined elsewhere) | `Move_Sim_Inventory_Aggr.sh` runs Hive/MapReduce jobs to aggregate simulated inventory. | Aggregated Hive tables / reports. | HDFS reads, Hive writes, possible temporary tables. |

External services referenced indirectly:
- HDFS (source data)
- Hive Metastore (target tables)
- Linux filesystem (status & log files)

# Integrations
- **Common Properties**: Inherits all global configuration (e.g., Hadoop classpath, default HDFS directories) from `MNAAS_CommonProperties.properties`.
- **Driver Script**: `MNAASSimInventoryStatusAggrScriptName` is resolved by the orchestration layer (e.g., cron, Oozie) to invoke `Move_Sim_Inventory_Aggr.sh`.
- **Process‑Status Framework**: The status file integrates with the broader MNAAS process‑status monitoring system used by other Move jobs.
- **Logging Infrastructure**: Log path aligns with the central MNAAS log aggregation (e.g., Logstash/ELK) for unified observability.

# Operational Risks
- **Missing Common Properties**: If the sourced common file is absent or corrupted, all dependent variables become undefined → job failure. *Mitigation*: Validate file existence before sourcing; enforce version control.
- **Path Misconfiguration**: Incorrect `MNAASConfPath` or `MNAASLocalLogPath` leads to unwritable status/log files. *Mitigation*: Pre‑run permission checks; monitor filesystem quotas.
- **Stale Status File**: Failure to clean or rotate the status file may cause false‑positive “already processed” checks. *Mitigation*: Implement TTL cleanup in the driver script.
- **Script Name Drift**: Changing the driver script name without updating this property breaks orchestration. *Mitigation*: Centralize script naming in a single source of truth and enforce CI checks.

# Usage
```bash
# Load the property file (typically done by the orchestration wrapper)
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/Move_Sim_Inventory_Aggr.properties

# Execute the driver script using the defined variable
bash "$MNAASSimInventoryStatusAggrScriptName"
```
For debugging, inspect the generated log:
```bash
tail -f "$sim_inventory_status_Aggr_Log"
```
And verify status file updates:
```bash
cat "$MNAAS_SimInventory_Status_Aggr_ProcessStatusFileName"
```

# Configuration
- **Environment Variables (inherited from `MNAAS_CommonProperties.properties`)**
  - `MNAASConfPath` – Base directory for configuration files.
  - `MNAASLocalLogPath` – Base directory for local log files.
- **Local Constants (defined in this file)**
  - `MNAAS_SimInventory_Status_Aggr_ProcessStatusFileName`
  - `sim_inventory_status_Aggr_Log`
  - `MNAASSimInventoryStatusAggrScriptName`

# Improvements
1. **Add Validation Logic** – Include a Bash function that verifies the existence and write permissions of the status and log paths at load time; exit with a clear error if checks fail.
2. **Parameterize Script Name** – Replace the hard‑coded `Move_Sim_Inventory_Aggr.sh` with a placeholder that can be overridden by an environment variable, enabling easier script versioning without modifying the properties file.