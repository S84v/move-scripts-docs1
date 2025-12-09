# Summary
`MNAAS_Sqoop.properties` is a Bash‑sourced configuration fragment used by the `MNAAS_Sqoop.sh` script in the MNAAS daily‑processing pipeline. It defines script metadata, process‑status file locations, log paths, HDFS directories, database table names, and the SQL queries executed by Sqoop to import raw rate and location data into staging tables and subsequently load them into Hive.

# Key Components
- **Source statement** – loads shared constants from `MNAAS_CommonProperties.properties`.
- **Script metadata**
  - `MNAAS_Sqoop_Scriptname` – name of the executable script.
- **Process status files**
  - `Rate_Location_Sqoop_ProcessStatusFile` – file used to record step success/failure.
- **Log configuration**
  - `Rate_Location_table_SqoopLogName` – path for Sqoop execution logs.
- **HDFS directories**
  - `Rate_table_Dir` – HDFS target for rate table imports.
  - `Location_table_Dir` – HDFS target for location table imports.
- **Database table identifiers**
  - `rate_table_name`, `rate_temp_table_name`, `location_table_name`.
- **Sqoop queries**
  - `Rate_table_Query` – source query for the rate table.
  - `Location_table_Query` – source query for the location table.
  - `Rate_Temp_To_Main_Query` – Hive INSERT that moves data from the temporary staging table to the final Hive table.

# Data Flow
| Stage | Input | Processing | Output / Side Effect |
|-------|-------|------------|----------------------|
| 1 | Oracle (or other RDBMS) tables referenced in `Rate_table_Query` and `Location_table_Query` | Sqoop import using `$CONDITIONS` for incremental splits | HDFS directories `Rate_table_Dir` and `Location_table_Dir` populated with Avro/Parquet files |
| 2 | HDFS staging files (rate temp) | Hive `INSERT INTO $dbname.$rate_table_name SELECT … FROM $dbname.$rate_temp_table_name` | Populated Hive table `$dbname.$rate_table_name` |
| 3 | Process status file | Script writes success/failure flag | Enables downstream orchestration (cron, Oozie, Airflow) |

External services:
- Oracle (or other source DB) accessed via DB links (`@gbspar.world`, `@fbrstst_comtst.world`, `@opstst.world`).
- HDFS (YARN) for intermediate storage.
- Hive/Impala for final table load.

# Integrations
- **`MNAAS_CommonProperties.properties`** – provides base variables such as `$dbname`, Hadoop user, default JDBC connection strings, and common debug flags.
- **`MNAAS_Sqoop.sh`** – consumes this properties file to construct Sqoop command lines, manage logs, and update process‑status files.
- **Downstream scripts** (e.g., `MNAAS_Rate_Load.sh`) that reference the Hive tables populated by `Rate_Temp_To_Main_Query`.
- **Orchestration layer** (cron or Oozie) that monitors the process‑status file for job completion.

# Operational Risks
- **Hard‑coded DB links** – changes in source DB topology break imports. *Mitigation*: externalize DB link names to a central config.
- **Missing `$CONDITIONS` handling** – if Sqoop split logic fails, import may stall. *Mitigation*: validate split column and provide defaults.
- **Log path permission issues** – script may fail to write logs. *Mitigation*: enforce directory ACLs and monitor disk usage.
- **Schema drift** – source table column changes break the `INSERT` query. *Mitigation*: version control queries and run schema validation before import.

# Usage
```bash
# Load properties
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Sqoop.properties

# Execute the Sqoop wrapper script with debug enabled
set -x
bash /app/hadoop_users/MNAAS/MNAAS_Scripts/MNAAS_Sqoop.sh
```
To debug a specific query, export `SQOOP_DEBUG=1` before running the script.

# Configuration
- **Environment variables** (inherited from `MNAAS_CommonProperties.properties`):
  - `dbname` – Hive database name.
  - `sqoop_jdbc_url`, `sqoop_user`, `sqoop_password`.
- **Referenced config files**
  - `MNAAS_CommonProperties.properties` – base configuration.
- **File system paths**
  - Process status: `/app/hadoop_users/MNAAS/MNAAS_Configuration_Files/Rate_Location_Sqoop_ProcessStatusFile`
  - Logs: `/app/hadoop_users/MNAAS/MNAASCronLogs/RateLocationtableLog`
  - HDFS dirs: `/user/MNAAS/sqoop/Rate_table`, `/user/MNAAS/sqoop/Location_table`

# Improvements
1. **Parameterize DB links and source schemas** – move `@gbspar.world`, `@fbrstst_comtst.world`, `@opstst.world` into separate properties to avoid code changes on environment migration.
2. **Introduce schema validation step** – add a pre‑Sqoop check that compares source column metadata with the target Hive schema and aborts with a clear error if mismatched.