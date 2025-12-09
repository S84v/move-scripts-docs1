# Summary
The `mnaas_vaz_table_loading.properties` file defines configuration parameters for the VAZ (Vehicle‑Analytics‑Zone) data ingestion pipeline. It supplies script names, HDFS paths, Hive table names, partitioning logic, and logging locations used by the `mnaas_vaz_table_loading.sh` driver to load daily and hourly VAZ source files, populate intermediate Hive tables, and generate aggregated reporting tables.

# Key Components
- **`mnaas_generic_vaz_scriptname`** – Main driver script invoked by the scheduler.  
- **Path variables** (`mnaas_vaz_data_landingdir`, `mnaas_vaz_load_generic_pathname`, `generic_vaz_inter_tblname`, etc.) – Define HDFS source, staging, and intermediate locations.  
- **Log & status files** (`mnaas_vaz_table_loadinglogname`, `mnaas_vaz_table_processstatusfilename`, `mnaas_vaz_table_aggrlogname`, …) – Capture execution metadata.  
- **Hive partition insert statements** (`vaz_dm_*_daily_Insert_Partitions`) – Dynamic‑partition INSERT/OVERWRITE statements for each VAZ domain table.  
- **Hive aggregation statements** (`vaz_dm_*_aggr_report`) – Window‑function based deduplication to produce “latest‑record‑per‑key” aggregates.  
- **Backup & reporting variables** (`generic_backupdir`, `generic_update_report`, `generic_vaz_reporting_table`).  
- **File transfer settings** (`SOURCE`, `DESTINATION`, `ccList`) – Used by the script to pull raw logs from the router host.

# Data Flow
1. **Ingestion** – Raw VAZ files are rsynced from `${SOURCE}` to `${DESTINATION}` (HDFS landing dir).  
2. **Staging** – Files are moved to `${mnaas_vaz_load_generic_pathname}` for Hive `LOAD DATA`.  
3. **Intermediate tables** – Data is loaded into `${generic_vaz_inter_tblname}` (e.g., `vaz_dm_qosmap_daily_inter`).  
4. **Partitioned load** – Each `*_daily_Insert_Partitions` statement inserts data into the corresponding Hive table partitioned by `partitiondate`.  
5. **Aggregation** – `*_aggr_report` statements read the daily tables, apply `ROW_NUMBER()` over business keys, and write the latest row per key into `${generic_vaz_reporting_table}` (e.g., `vaz_dm_qosmap_aggr`).  
6. **Backup** – Processed files are copied to `${generic_backupdir}`.  
7. **Status & logging** – Process status files and logs are written to the paths defined by `${MNAASConfPath}` and `${logdir}`.  

**Side Effects**: Creation of HDFS directories, Hive table partitions, and backup copies; email notifications to `${ccList}` on failure.

**External Services**:  
- HDFS (file storage)  
- Hive/Tez execution engine (SQL processing)  
- Remote router host (`vaz@10.171.102.16`) via SCP/rsync  
- Email (SMTP) for alerts  

# Integrations
- **Property includes** – Sources common properties from `mnaaspropertiese.prop` and `MNAAS_CommonProperties.properties`.  
- **Driver script** – `mnaas_vaz_table_loading.sh` reads this file to resolve all variables.  
- **Hive metastore** – All INSERT statements target tables in the `mnaas` database.  
- **Scheduler** – Typically invoked by Oozie/Airflow with `${processname}` and `${filetype}` set per run.  
- **Backup subsystem** – Uses `${backuplocation}` defined in common properties.  

# Operational Risks
| Risk | Impact | Mitigation |
|------|--------|------------|
| Missing source files (router down) | Incomplete daily load → downstream reports stale | Add pre‑flight check for file count; retry logic with exponential back‑off |
| Hive dynamic partition limits exceeded | Job failure, partial data load | Ensure `hive.exec.max.dynamic.partitions.pernode` is sized; monitor partition count |
| Partition date mismatch (`'part_date'` placeholder not replaced) | Inserts go to wrong partition or fail | Validate that `${processdate}` is substituted before Hive execution |
| Log rotation overflow (`logname$(date +_%F)`) | Log files grow without limit | Implement log rotation or size‑based cleanup script |
| Email alert suppression (invalid `ccList`) | Failure unnoticed | Verify email address format at deployment; fallback to default admin list |

# Usage
```bash
# Export required env vars (example)
export processname=vaz
export filetype=dm
export MNAASConfPath=/app/hadoop_users/MNAAS/conf
export logdir=/var/log/mnaas
export backuplocation=/app/hadoop_users/MNAAS/backup

# Run the driver script
/app/hadoop_users/MNAAS/scripts/mnaas_vaz_table_loading.sh -c move-mediation-scripts/config/mnaas_vaz_table_loading.properties
```
*Debug*: Add `-x` to the shell script for trace; inspect `${logdir}` for the generated log file; query Hive tables for the latest `partitiondate`.

# Configuration
- **Environment variables** required by the script: `processname`, `filetype`, `MNAASConfPath`, `logdir`, `backuplocation`.  
- **Referenced config files**:  
  - `mnaaspropertiese.prop` – Global MNAAS settings (Hadoop classpath, Hive connection).  
  - `MNAAS_CommonProperties.properties` – Common paths (`$logdir`, `$backuplocation`, `$MNAASConfPath`).  
- **Key property entries** (see file): script name, HDFS directories, Hive table names, column‑to‑date mappings (`vaz_dm_*_column`), partition file names, source/destination host, email CC list.

# Improvements
1. **Parameterize `part_date`** – Replace the hard‑coded placeholder with `${processdate}` substitution to avoid manual edit and ensure correct partition targeting.  
2. **Centralize Hive settings** – Move repeated Hive session parameters (`set hive.exec.dynamic.partition…`) into a shared `.hql` header file to reduce duplication and simplify future engine changes.