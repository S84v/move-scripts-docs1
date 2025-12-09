# Summary
Defines Bash‑sourced runtime constants for the **MNAAS_TrafficDetails_daily_tbl_aggr** job, which aggregates daily traffic‑detail records, manages Hive partition drops/loads, runs supporting Hive scripts, and logs process status.

# Key Components
- `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` – imports shared defaults (paths, DB name, etc.).
- `MNAAS_Daily_Traffictable_Load_daily_Aggr_ProcessStatusFileName` – HDFS status‑file path.
- `MNAAS_DailyTrafficDetailsLoad_daily_AggrLogPath` – local log file with date suffix.
- `Dname_MNAAS_drop_partitions_daily_TrafficDetails_aggr_tbl` – Hive drop‑partition script identifier.
- `Dname_MNAAS_Load_daily_TrafficDetails_aggr_tbl_temp` – Hive load‑temp‑table script identifier.
- `processname_traffic_details_daily_aggr` – logical process name (“daily”).
- `MNAASdim_date_time_3monthsScriptName` – script to generate 3‑month date dimension.
- `Move_msisdn_details_kyc_addon_tblname` / `MNAAS_msisdn_details_kyc_addon_script` / `msisdn_details_kyc_addon_table_refresh` – KYC addon Hive table, script, and refresh command.
- `Move_msisdn_details_kyc_bal_update_tblname` / `MNAAS_msisdn_details_kyc_bal_update_script` / `msisdn_details_kyc_bal_update_table_refresh` – KYC balance update Hive table, script, and refresh command.
- `MNAAS_sim_inventory_kyc_rnr_table` / `MNAAS_sim_inventory_kyc_rnr_script` / `msisdn_sim_inventory_kyc_rnr_table_refresh` – SIM inventory KYC RNR Hive table, script, and refresh command.
- `MNAAS_msisdn_jlr_rate_per_country` – auxiliary shell script for JLR rate per country.

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|---------------|
| Status Init | None | Reads common properties | Sets `*_ProcessStatusFileName` | Creates/updates HDFS status file |
| Log Init | None | Constructs log path with `date +_%F` | Writes to `$MNAASLocalLogPath/...` | Log file rotation |
| Partition Drop | Existing Hive table partitions | Executes `$Dname_MNAAS_drop_partitions_daily_TrafficDetails_aggr_tbl` script | Partitions removed | Hive metadata change |
| Temp Load | Staged CSV/Parquet in HDFS | Executes `$Dname_MNAAS_Load_daily_TrafficDetails_aggr_tbl_temp` script | Temp Hive table populated | HDFS read/write |
| Dim Table Generation | Date range | Runs `$MNAASdim_date_time_3monthsScriptName` | `dim_date_time` table updated | Hive DML |
| KYC Add‑on / Balance / SIM RNR | Source tables | Runs respective `.hql` scripts | Target tables refreshed via `refresh` commands | Hive table cache invalidation |
| JLR Rate | External rate source | Runs `$MNAAS_msisdn_jlr_rate_per_country` | Rate table updated | Potential external API calls |

# Integrations
- **MNAAS_CommonProperties.properties** – provides `$MNAASConfPath`, `$MNAASLocalLogPath`, `$dbname`, and Hadoop/Hive environment variables.
- **Hive** – all `*_script.hql` files are submitted via `hive -f`.
- **HDFS** – status file and staging directories reside on HDFS.
- **Downstream jobs** – refreshed tables are consumed by downstream billing and reporting pipelines.
- **External rate service** – invoked indirectly by `MNAAS_msisdn_jlr_rate_per_country.sh`.

# Operational Risks
- **Stale status file** – if job aborts, status may remain “running”. Mitigation: idempotent cleanup step or timeout monitor.
- **Partition drop race condition** – concurrent jobs may attempt to drop same partitions. Mitigation: enforce single‑instance lock (e.g., Zookeeper lock file).
- **Hive script failures** – syntax errors or missing tables cause job failure. Mitigation: unit‑test `.hql` scripts and validate existence before execution.
- **Log file growth** – daily logs accumulate. Mitigation: log rotation / retention policy.
- **Hard‑coded DB name in refresh commands** – may break if `$dbname` changes. Mitigation: derive dynamically from common properties.

# Usage
```bash
# Source common properties first
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Source this job‑specific config
. /path/to/MNAAS_TrafficDetails_daily_tbl_aggr.properties

# Execute driver script (example)
bash MNAAS_TrafficDetails_daily_tbl_aggr.sh \
    --status-file "$MNAAS_Daily_Traffictable_Load_daily_Aggr_ProcessStatusFileName" \
    --log "$MNAAS_DailyTrafficDetailsLoad_daily_AggrLogPath"
```
For debugging, export `HIVE_LOG_LEVEL=DEBUG` and inspect the generated log file.

# Configuration
- **Environment Variables**: `MNAASConfPath`, `MNAASLocalLogPath`, `dbname` (from common properties).
- **Referenced Config Files**: `MNAAS_CommonProperties.properties`.
- **External Scripts**: `dim_date_time_3months.sh`, `MNAAS_msisdn_jlr_rate_per_country.sh`.
- **Hive Scripts**: `MNAAS_msisdn_details_kyc_addon_script.hql`, `MNAAS_msisdn_details_kyc_bal_update_script.hql`, `move_kyc_sim_inventory_rnr_data.hql`.

# Improvements
1. **Parameterize DB name** in refresh commands using `${dbname}` consistently to avoid hard‑coding.
2. **Add lock file mechanism** (e.g., `/tmp/traffic_details_daily.lock`) to guarantee single‑instance execution and automatic cleanup on failure.