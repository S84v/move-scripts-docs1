# Summary
Defines Bash‑sourced runtime constants for the **MNAAS_TrafficDetails_hourly_tbl_aggr** aggregation job. The constants configure status‑file locations, log file naming, Hive temporary and final table identifiers, and the process name used by the hourly traffic‑details aggregation script in the telecom mediation pipeline.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – imports shared environment defaults (base paths, DB names, Hadoop configuration).
- **`MNAAS_Daily_Traffictable_Load_Hourly_Aggr_ProcessStatusFileName`** – HDFS path for the job’s status file.
- **`MNAAS_DailyTrafficDetailsLoad_Hourly_AggrLogPath`** – Local log file path with date suffix.
- **`Dname_MNAAS_drop_partitions_hourly_TrafficDetails_aggr_tbl`** – Hive script name for dropping stale hourly partitions.
- **`Dname_MNAAS_Load_hourly_TrafficDetails_aggr_tbl_temp`** – Hive temporary table name used during load.
- **`Dname_MNAAS_Load_hourly_TrafficDetails_aggr_tbl`** – Final Hive table name for hourly traffic‑details aggregation.
- **`MNAAS_Traffic_Partitions_Hourly_Uniq_FileName`** – HDFS file that stores the list of unique hourly partitions processed.
- **`processname_traffic_details_hourly_aggr`** – Identifier string used for logging and monitoring.

# Data Flow
| Stage | Input | Transformation | Output | Side Effects |
|-------|-------|----------------|--------|---------------|
| **Status Check** | HDFS status file (`MNAAS_Daily_Traffictable_Load_Hourly_Aggr_ProcessStatusFileName`) | Read to determine prior run state | N/A | May abort if previous run failed |
| **Log Init** | None | Create/append to local log (`MNAAS_DailyTrafficDetailsLoad_Hourly_AggrLogPath`) | Log entries | Disk I/O on local node |
| **Partition Drop** | Hive table `Dname_MNAAS_drop_partitions_hourly_TrafficDetails_aggr_tbl` | Execute Hive script to drop old hourly partitions | Updated Hive metadata | Hive metastore mutation |
| **Load Temp** | Source raw traffic‑detail files (path defined in common properties) | Insert into temporary Hive table `Dname_MNAAS_Load_hourly_TrafficDetails_aggr_tbl_temp` | Temp table populated | HDFS write |
| **Finalize Load** | Temp table data | Insert/overwrite into final table `Dname_MNAAS_Load_hourly_TrafficDetails_aggr_tbl` | Final hourly aggregated table | Hive metastore update, HDFS write |
| **Partition Tracking** | Processed partition list | Write unique partition identifiers to HDFS file `MNAAS_Traffic_Partitions_Hourly_Uniq_FileName` | Partition list file | HDFS write |
| **Status Update** | None | Write success/failure flag to status file | Updated status file | HDFS write |

# Integrations
- **Common Properties** (`MNAAS_CommonProperties.properties`): supplies `$MNAASConfPath`, `$MNAASLocalLogPath`, Hive DB name, Hadoop user, and other global constants.
- **Hive**: scripts referenced by `Dname_*` variables are executed via `hive -f <script>`; tables reside in the configured Hive database.
- **HDFS**: status file, partition‑tracking file, and intermediate data are stored on HDFS under `$MNAASConfPath`.
- **Driver Script**: `MNAAS_TrafficDetails_hourly_tbl_aggr.sh` (not shown) sources this properties file to obtain constants.
- **Monitoring/Orchestration**: `processname_traffic_details_hourly_aggr` is used by job schedulers (e.g., Oozie, Airflow) for alerting and SLA tracking.

# Operational Risks
- **Stale Status File**: If previous run failed and status file not cleared, subsequent runs may be skipped. *Mitigation*: Include cleanup step or status‑file timeout logic.
- **Partition Drop Mis‑execution**: Dropping wrong partitions can cause data loss. *Mitigation*: Validate partition list before execution; retain backups.
- **Log File Growth**: Daily log files appended with date suffix may accumulate. *Mitigation*: Implement log rotation/compression.
- **Hive Table Name Collisions**: Temporary table name must be unique per run. *Mitigation*: Append run timestamp or use session‑scoped temporary tables.
- **HDFS Path Permissions**: Incorrect permissions on `$MNAASConfPath` can cause write failures. *Mitigation*: Verify Hadoop ACLs before job start.

# Usage
```bash
# Source common properties first
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Source this job‑specific properties file
. /app/hadoop_users/MNAAS/move-mediation-scripts/config/MNAAS_TrafficDetails_hourly_tbl_aggr.properties

# Execute the driver script (example)
bash /app/hadoop_users/MNAAS/move-mediation-scripts/bin/MNAAS_TrafficDetails_hourly_tbl_aggr.sh
```
For debugging, export `HIVE_DEBUG=true` and inspect the generated log at `$MNAASDailyTrafficDetailsLoad_Hourly_AggrLogPath`.

# Configuration
- **Environment Variables (in common properties)**  
  - `MNAASConfPath` – Base HDFS configuration directory.  
  - `MNAASLocalLogPath` – Base local filesystem log directory.  
  - `HIVE_DB` – Target Hive database name.  
- **Files Referenced**  
  - `MNAAS_CommonProperties.properties` – shared defaults.  
  - Hive scripts: `MNAAS_drop_partitions_hourly_TrafficDetails_aggr_tbl.hql`, `MNAAS_Load_hourly_TrafficDetails_aggr_tbl_temp.hql`, `MNAAS_Load_hourly_TrafficDetails_aggr_tbl.hql`.  
  - HDFS status file: `$MNAASConfPath/MNAAS_Daily_Traffictable_Load_Hourly_Aggr_ProcessStatusFile`.  
  - Partition tracking file: `$MNAASConfPath/MNAAS_Traffic_Partitions_Hourly_Uniq`.

# Improvements
1. **Parameterize Log Rotation** – Add a property for max log size and integrate with `logrotate` to prevent uncontrolled log growth.