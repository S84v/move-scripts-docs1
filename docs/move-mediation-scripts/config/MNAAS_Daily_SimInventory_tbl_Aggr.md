# Summary
Defines runtime constants for the **MNAAS_Daily_SimInventory_tbl_Aggr** batch job. The properties are sourced by the `MNAAS_Daily_SimInventory_tbl_Aggr.sh` script to locate the process‑status file, set the local log directory, and identify the script name used in the daily SIM inventory aggregation pipeline of the telecom Move production environment.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – imports shared configuration (base paths, Hadoop settings, environment variables).  
- **`MNAAS_Daily_SimInventory_Aggr_ProcessStatusFileName`** – full HDFS/local path to the process‑status file used for idempotency and monitoring.  
- **`MNAASDailySimInventoryAggrLogPath`** – local filesystem path where job logs are written.  
- **`MNAASDailySimInventoryAggrScriptName`** – name of the executable shell script that performs the aggregation.

# Data Flow
| Stage | Input | Transformation | Output | Side Effects |
|-------|-------|----------------|--------|--------------|
| 1. Load common props | `MNAAS_CommonProperties.properties` | Variable substitution (`$MNAASConfPath`, `$MNAASLocalLogPath`) | Populated environment variables | None |
| 2. Process status check | `MNAAS_Daily_SimInventory_Aggr_ProcessStatusFileName` | Script reads file to determine if job already ran | Boolean flag (run/skip) | May create/append status file |
| 3. Execution | `MNAASDailySimInventoryAggrScriptName` | Shell script runs Hive/HDFS commands to aggregate SIM inventory | Aggregated Hive table / HDFS files | Writes logs to `MNAASDailySimInventoryAggrLogPath` |
| 4. Completion | – | Script updates process‑status file | Updated status file | Log rotation, possible alerts |

# Integrations
- **Common Properties**: Inherits base paths (`MNAASConfPath`, `MNAASLocalLogPath`) and Hadoop configuration from `MNAAS_CommonProperties.properties`.  
- **Hive/HDFS**: The aggregation script interacts with Hive tables and HDFS directories defined indirectly via the common properties.  
- **Monitoring/Orchestration**: Process‑status file is consumed by the daily cron orchestrator (`MNAAS_Daily_*` jobs) to enforce idempotency.  
- **Logging Infrastructure**: Logs written to the path defined here are harvested by the central log aggregation system (e.g., Splunk/ELK).

# Operational Risks
- **Stale Process‑Status File**: If the status file is not cleaned after a successful run, subsequent runs may be incorrectly skipped. *Mitigation*: Implement TTL cleanup or explicit status reset in the cron.  
- **Path Misconfiguration**: Incorrect `$MNAASConfPath` or `$MNAASLocalLogPath` leads to file not found errors. *Mitigation*: Validate paths at script start; fail fast with clear error messages.  
- **Insufficient Disk Space for Logs**: Log directory growth can exhaust local storage. *Mitigation*: Enforce log rotation and retention policies.  
- **Permission Issues**: Script may lack write permission on status or log paths. *Mitigation*: Ensure the executing user (`hadoop_user`/`mnaas_user`) has appropriate ACLs.

# Usage
```bash
# Source the properties to export variables
source /app/hadoop_users/MNAAS/config/MNAAS_Daily_SimInventory_tbl_Aggr.properties

# Verify variables
echo $MNAAS_Daily_SimInventory_Aggr_ProcessStatusFileName
echo $MNAASDailySimInventoryAggrLogPath
echo $MNAASDailySimInventoryAggrScriptName

# Run the aggregation script (debug mode)
bash -x $MNAASDailySimInventoryAggrScriptName
```

# Configuration
- **Environment Variables** (populated by `MNAAS_CommonProperties.properties`):  
  - `MNAASConfPath` – base directory for configuration files.  
  - `MNAASLocalLogPath` – base directory for local logs.  
- **Referenced Config Files**:  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` – must be present and readable.  

# Improvements
1. **Add TTL Management**: Introduce a property `MNAAS_SimInventory_Aggr_StatusTTLHours` and modify the script to purge or reset status files older than the TTL.  
2. **Parameterize Log Retention**: Add `MNAAS_SimInventory_Aggr_LogRetentionDays` and integrate with logrotate to automate cleanup.