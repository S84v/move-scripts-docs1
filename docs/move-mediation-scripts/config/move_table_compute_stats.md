# Summary
Defines Bash‑sourced runtime constants for the **Move_Table_Compute_Stats** job. The job reads a list of Hive/Impala tables, computes statistics via Impala, and logs execution. Constants are imported from the shared `MNAAS_CommonProperties.properties` file.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – imports shared environment variables.  
- **`IMPALA_URL`** – constructed Impala JDBC endpoint (`$IMPALAD_HOST:$IMPALAD_JDBC_PORT`).  
- **`TIMEOUT`** – maximum seconds allowed for the Impala statistics computation.  
- **`TABLE_LIST`** – path to the file containing the list of tables to process (`$MNAASConfPath/move_table_compute_stats_tables`).  
- **`MOVE_TABLE_COMPUTE_STATS_LOG`** – log file path with daily suffix (`$MNAASLocalLogPath/move_table_compute_stats.log_YYYY-MM-DD`).  

# Data Flow
| Element | Direction | Description |
|---------|-----------|-------------|
| **Input** | Read | `MNAAS_CommonProperties.properties` (shared vars). |
| **Input** | Read | `TABLE_LIST` file – plain‑text list of target tables. |
| **External Service** | Connect | Impala server via `$IMPALA_URL`. |
| **Processing** | Execute | For each table, issue `COMPUTE STATS` (or equivalent) with a `$TIMEOUT` guard. |
| **Output** | Write | Log entries appended to `$MOVE_TABLE_COMPUTE_STATS_LOG`. |
| **Side Effect** | None | No data mutation beyond Impala statistics metadata. |

# Integrations
- **Driver Script**: `Move_Table_Compute_Stats.sh` sources this properties file to obtain constants.  
- **Common Library**: `MNAAS_CommonProperties.properties` supplies base paths, hostnames, and credentials used across the move suite.  
- **Impala**: Statistics computation is performed via Impala JDBC; the URL is built from common vars.  

# Operational Risks
- **Missing Common Properties** – job fails if the sourced file is absent or unreadable. *Mitigation*: Validate file existence before sourcing.  
- **Incorrect Impala Host/Port** – leads to connection timeout. *Mitigation*: Health‑check Impala endpoint prior to execution.  
- **Stale or Corrupt `TABLE_LIST`** – may cause unnecessary processing or errors. *Mitigation*: Verify file format and non‑empty content.  
- **Log File Growth** – unbounded log size can fill local disk. *Mitigation*: Implement log rotation or retention policy.  

# Usage
```bash
# Source the properties (normally done inside the driver)
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/move_table_compute_stats.properties

# Run the driver script (example)
bash Move_Table_Compute_Stats.sh   # driver uses the variables defined above
```
For debugging, echo variables after sourcing:
```bash
echo "Impala URL: $IMPALA_URL"
echo "Table list: $TABLE_LIST"
echo "Log file: $MOVE_TABLE_COMPUTE_STATS_LOG"
```

# Configuration
- **Environment Variables (from common properties)**  
  - `IMPALAD_HOST` – Impala server hostname.  
  - `IMPALAD_JDBC_PORT` – Impala JDBC port.  
  - `MNAASConfPath` – Base directory for configuration files.  
  - `MNAASLocalLogPath` – Base directory for local logs.  
- **Referenced Files**  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  - `$MNAASConfPath/move_table_compute_stats_tables` (table list)  

# Improvements
1. **Parameterize Timeout** – expose `TIMEOUT` via an external config or command‑line argument to allow per‑run tuning.  
2. **Add Log Rotation** – integrate `logrotate` or timestamped archival to prevent uncontrolled log growth.