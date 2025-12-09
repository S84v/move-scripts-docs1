# Summary
`Sim_Inventory_Sqoop.properties` is a Bash‑style configuration wrapper for the Move mediation pipeline’s SIM inventory extraction job. It sources the global `MNAAS_CommonProperties.properties` file and defines all environment‑specific variables required by the `Sim_Inventory_Sqoop.sh` driver script, including script name, status‑file location, log file path, Hive table name, backup HDFS directory, and paths to the loader configuration, JAR, and Java log. The file enables a single source of truth for paths and filenames used throughout the SIM inventory Sqoop load process.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – sources shared Hadoop, credential, and logging settings.
- **`MNAASSimInventorySqoopScriptName`** – name of the driver shell script.
- **`MNAAS_SimInventory_Sqoop_ProcessStatusFileName`** – absolute path to the process‑status flag file.
- **`sim_inventory_SqoopLog`** – log file path (includes daily date suffix).
- **`Dname_Move_siminventory_status_tblname`** – Hive table identifier for the SIM inventory status view.
- **`Sim_inventory_backup`** – HDFS directory where raw Sqoop extracts are archived.
- **`SIMLoaderConfFilePath`** – path to the loader‑specific configuration text file.
- **`SIMLoaderJarPath`** – full HDFS/local path to `SIMLoader.jar`.
- **`SIMLoaderJavaLogPath`** – Java‑level log file for the loader (daily date suffix).

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| **Driver** (`Sim_Inventory_Sqoop.sh`) | Variables from this file + common properties | Executes Sqoop import using `SIMLoaderJarPath` and `SIMLoaderConfFilePath`; writes raw data to `Sim_inventory_backup`; updates Hive table `Dname_Move_siminventory_status_tblname`; logs to `sim_inventory_SqoopLog` and `SIMLoaderJavaLogPath`; writes status flag to `MNAAS_SimInventory_Sqoop_ProcessStatusFileName`. | HDFS files, Hive table rows, log entries, status file. |
| **External services** | Hadoop cluster (HDFS, YARN), Hive Metastore, Sqoop, underlying telecom OSS/CRM source system. | N/A | Data persisted in HDFS and Hive. |

# Integrations
- **`MNAAS_CommonProperties.properties`** – provides `$MNAASConfPath`, `$MNAASJarPath`, `$MNAASLocalLogPath`, Hadoop configuration, and credential variables.
- **`Sim_Inventory_Sqoop.sh`** – consumes all variables defined here to construct command‑line arguments for Sqoop and the Java loader.
- **Hive** – the status table referenced by `Dname_Move_siminventory_status_tblname`.
- **HDFS** – backup directory `Sim_inventory_backup` and loader JAR location.
- **Scheduler (Cron/ODI)** – typically invoked by a cron entry that sources this properties file before execution.

# Operational Risks
- **Path drift** – Hard‑coded absolute paths may become stale after filesystem re‑layout. *Mitigation*: centralize base directories in `MNAAS_CommonProperties.properties` and reference them consistently.
- **Date suffix handling** – Log filenames embed `$(date +_%F)` at source time; if the file is sourced multiple times in a single run, logs may be split unintentionally. *Mitigation*: compute date once and reuse a single variable.
- **Missing status file** – Downstream jobs may block if `MNAAS_SimInventory_Sqoop_ProcessStatusFileName` is not created/removed correctly. *Mitigation*: add existence checks and cleanup logic in the driver script.
- **Permission mismatches** – The HDFS backup directory must be writable by the Hadoop user running the job. *Mitigation*: enforce ACLs and validate before job start.

# Usage
```bash
# From a terminal on the mediation host
source /app/hadoop_users/MNAAS/move-mediation-scripts/config/Sim_Inventory_Sqoop.properties
# Verify variables (optional)
env | grep -i sim_inventory
# Run the driver script (debug mode)
bash -x $MNAASSimInventorySqoopScriptName
```
To debug a specific variable:
```bash
echo "Log file: $sim_inventory_SqoopLog"
```

# Configuration
- **Referenced files**
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
  - `$MNAASConfPath/MNAAS_Daily_SimInventory_Load_Status.txt`
  - `$MNAASJarPath/Loader_Jars/SIMLoader.jar`
- **Environment variables expected from common properties**
  - `MNAASConfPath`
  - `MNAASJarPath`
  - `MNAASLocalLogPath`
  - Hadoop client variables (`HADOOP_CONF_DIR`, `HADOOP_USER_NAME`, etc.)
- **Adjustable parameters**
  - `Sim_inventory_backup` – change if backup location moves.
  - `Dname_Move_siminventory_status_tblname` – rename if Hive schema changes.

# Improvements
1. **Parameterize date handling** – introduce a single `RUN_DATE=$(date +%F)` variable at the top of the file and reference it for all log filenames to avoid inconsistent timestamps.
2. **Add validation block** – at the end of the file, insert a Bash function that checks existence and write permission of all referenced paths, exiting with a clear error code if any check fails. This will surface configuration drift before the heavy Sqoop job starts.