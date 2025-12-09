# Summary
Defines environment‑specific variables for the **MNAAS Actives Table Load Driver** job. It sources common MNAAS properties and sets paths for status files, log files, and the driver/aggregation shell scripts used by the daily actives aggregation and Hive table load process in the Move‑Mediation pipeline.

# Key Components
- `MNAAS_Daily_Actives_Load_CombinedActivesStatuFile` – HDFS path to the combined actives status flag file.  
- `MNAAS_DailyActivesCombinedLoadLogPath` – Local filesystem path for the driver log file, timestamped per execution.  
- `MNAASDailyActivesProcessingScript` – Name of the driver shell script invoked by the scheduler (`MNAAS_Actives_tbl_load_driver.sh`).  
- `MNAASDailyActivesAggregationScriptName` – Name of the aggregation script called by the driver (`MNAAS_Actives_tbl_Load.sh`).  
- Source line `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` – imports shared configuration (e.g., `$MNAASConfPath`, `$MNAASLocalLogPath`).

# Data Flow
| Stage | Input | Transformation | Output |
|-------|-------|----------------|--------|
| Driver start | Scheduler (cron) triggers `MNAASDailyActivesProcessingScript` | Reads variables from this properties file; passes status‑file path and log path to driver script | Log file written to `$MNAASLocalLogPath`; status flag created/updated in `$MNAASConfPath` |
| Aggregation | Driver script invokes `MNAASDailyActivesAggregationScriptName` | Reads raw actives files from HDFS, aggregates, loads into Hive | Updated Hive tables; aggregation status flag (`MNAAS_Daily_Actives_Load_CombinedActivesStatuFile`) |
| Side effects | HDFS directories, local log directory | Creation of intermediate control files, error files (handled by downstream scripts) | Persistent state for downstream jobs (e.g., table load, reporting) |

# Integrations
- **MNAAS_CommonProperties.properties** – provides base paths (`$MNAASConfPath`, `$MNAASLocalLogPath`) and common Hadoop/Hive configuration.  
- **MNAAS_Actives_tbl_load_driver.sh** – consumes variables defined here; orchestrates validation, aggregation, and Hive load.  
- **MNAAS_Actives_tbl_Load.sh** – called by the driver to perform actual data aggregation and Hive `INSERT/OVERWRITE`.  
- **Cron scheduler** – triggers the driver script according to the production schedule.  
- **HDFS** – stores raw actives feeds, status flag file, and intermediate control files.  
- **Hive Metastore** – target of the load operation.

# Operational Risks
- **Missing common properties** – if `MNAAS_CommonProperties.properties` is unavailable, all derived paths resolve to empty, causing script failures. *Mitigation*: health‑check for source file at job start.  
- **Log path permission issues** – driver may fail to write log if `$MNAASLocalLogPath` is not writable. *Mitigation*: enforce directory ACLs and monitor log creation.  
- **Stale status flag** – leftover flag from previous run may cause downstream jobs to skip processing. *Mitigation*: driver script should clean or overwrite the flag at start.  
- **Date format mismatch** – log filename uses `$(date +_%F)`; locale changes could alter output. *Mitigation*: enforce `LC_ALL=C` in cron environment.

# Usage
```bash
# Load properties (usually done inside the driver script)
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source /path/to/config/MNAAS_Actives_tbl_load_driver.properties

# Manually invoke driver for debugging
bash MNAAS_Actives_tbl_load_driver.sh -debug

# Verify log creation
tail -f $MNAASLocalLogPath/MNAAS_Actives_tbl_load_driver.log_$(date +%F)
```

# Configuration
- **Environment variables** (populated by `MNAAS_CommonProperties.properties`):  
  - `MNAASConfPath` – base HDFS config directory.  
  - `MNAASLocalLogPath` – base local log directory.  
- **Referenced config files**:  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  - This properties file itself (`MNAAS_Actives_tbl_load_driver.properties`).  

# Improvements
1. **Add validation block** to confirm that `$MNAASConfPath` and `$MNAASLocalLogPath` exist and are writable before script execution.  
2. **Externalize script names** into a separate “script‑registry” properties file to allow versioned script swaps without modifying driver‑specific files.