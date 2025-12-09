# Summary
`MNAAS_CommonProperties.properties` is the central configuration repository for the MNAAS (Move‑Network‑As‑A‑Service) production pipeline. It defines filesystem paths, environment‑specific includes, naming conventions, Hive/Impala refresh statements, Sqoop queries, Java class references, and customer‑specific mappings used by all cron‑driven ingestion, validation, aggregation, and backup scripts that process daily CDR, activation, tolling, SIM‑inventory, IPVProbe and KYC feeds.

# Key Components
- **Environment Includes** – Sources `MNAAS_Removalble.properties` and `MNAAS_Environment.properties`.
- **Path Variables** – Base directories (`MNAASLocalPath`, `MNAASConfPath`, `MNAASLocalLogPath`, etc.) and derived sub‑paths for staging, backups, intermediate files, and HDFS locations.
- **File‑Pattern Variables** – Glob patterns for each feed type (e.g., `Traffic_Details_extn`, `Activations_extn`, `Holland01_Traffic_Details_extn`).
- **Hive/Impala Refresh Statements** – `*_refresh` strings for all tables (traffic, activations, actives, siminventory, etc.).
- **Sqoop Configuration** – Connection strings, query templates (`sim_inventory_SqoopQuery`, `gbs_journal_SqoopQuery`, etc.) and script name `MNAAS_Sqoop.sh`.
- **Java Class & JAR References** – Loader, aggregation, validation, and retention classes (e.g., `Insert_Part_Daily_table`, `MNAAS_MNAAS_seqno_check_classname`, `MNAAS_Main_JarPath`).
- **Customer Mappings** – Associative arrays for SECS IDs, sub‑customers, rating groups, feed‑to‑customer mapping, backup directories, Windows job identifiers, and feed enable flags.
- **Control Files & Status Files** – Paths for sequence‑check status, backup process status, and aggr control property files.
- **Script Name Constants** – Centralized names for reusable scripts (e.g., `MNAASDailyTrafficDetails_daily_Aggr_SriptName`).

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| **Ingestion** | Raw files in `/Input_Data_Staging/*` matching defined glob patterns | Cron scripts source this properties file → use path & pattern vars → move files to `$MNAASInterFilePath` | Files staged for Hive loading; duplicate handling dirs created |
| **Validation** | Staged files, previous sequence files (`MNASS_SeqNo_Check_*`) | Java validation classes (`MNAAS_MNAAS_seqno_check_classname`, `MNAAS_MyRep_timestamp_check_classname`) invoked via scripts | Sequence‑check logs (`seqlog_SimInventory`), status files updated |
| **Loading** | Staged CSVs | Sqoop jobs (`MNAAS_Sqoop.sh`) using query templates → HDFS tables (`$SqoopPath/*`) | HDFS parquet/ORC tables, raw‑load status logs |
| **Aggregation** | Hive tables (raw) | Impala `REFRESH` statements (e.g., `traffic_details_daily_tblname_refresh`) → Java aggregation jobs (`Insert_Daily_Aggr_table`, `MNAAS_business_trend_aggregation`) | Aggregated tables, daily reports, KYC feeds |
| **Backup** | Processed files | Copy to `$BackupDir` hierarchy (`Daily_Tolling_BackupDir`, etc.) | Backup files on edge node, duplicate tracking dirs |
| **Reporting** | Aggregated tables | API scripts (`traffic_details_daily_api_tblname_load`) → external dashboards | Updated BI dashboards, Tableau security sync |

External services: HDFS, Impala, Hive Metastore, Sqoop (Oracle/DB2), Java JARs, email notifications.

# Integrations
- **Other Property Files** – `MNAAS_Removalble.properties`, `MNAAS_Environment.properties`, `MNAAS_CommonProperties_dynamic.properties`.
- **Cron Scripts** – All MNAAS cron jobs source this file for path/variable resolution.
- **Java JARs** – Loaded via `$MNAASJarPath` (e.g., `Raw_Aggr_tables_processing/mnaas.jar`).
- **Sqoop** – Queries built from variables; results land in HDFS directories defined here.
- **Impala/Hive** – Refresh statements referenced by aggregation scripts.
- **Email** – `T0_email` used by notification modules.
- **Windows Job Scheduler** – Mapping via `MNAAS_windows_job_mapping` for cross‑platform triggers.

# Operational Risks
- **Stale Path Variables** – Hard‑coded absolute paths can break after filesystem re‑layout. *Mitigation*: externalize base paths to environment‑specific property files.
- **Glob Pattern Drift** – Adding new feed types requires updating multiple pattern variables. *Mitigation*: consolidate patterns into a single associative array.
- **Sequence‑Check Inconsistency** – Separate folder variables (`MNASS_SeqNo_Check_*`) may diverge. *Mitigation*: generate them programmatically from a base map.
- **Credential Leakage** – Sqoop queries embed table names but not credentials; ensure credentials are sourced from secure vaults, not this file.
- **Large Variable Scope** – Global export may cause name collisions in subshells. *Mitigation*: prefix all variables with `MNAAS_` consistently (already mostly done).

# Usage
```bash
# Load configuration in a bash script
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Example: list all traffic files for Holland01
ls ${MNAASInterFilePath_Daily}/${Holland01_Traffic_Details_extn}

# Debug a specific variable
echo "Backup dir for MyRep: ${MNAAS_customer_backup_dir[MyRep]}"
```
To run a full daily pipeline:
```bash
bash $MNAAS_CronFilePath/MNAAS_Daily_Run.sh   # script sources this properties file internally
```

# Configuration
- **Primary Files**: `MNAAS_Removalble.properties`, `MNAAS_Environment.properties`, `MNAAS_CommonProperties_dynamic.properties`.
- **Environment Variables**: `MNAASLocalPath`, `MNAASJarPath`, `MNAASPathName`, `dbname`, `tolling_daily_tblname`, etc. (populated by the included files).
- **Customer‑Specific Maps**: `MNAAS_Customer_SECS`, `CUSTOMER_MAPPING`, `MNAAS_customer_backup_dir`.
- **Email**: `T0_email` (notification recipient).

# Improvements
1. **Mod