# Summary
`MNAAS_Java_Batch.properties` is a central configuration file for the Move‑Mediation production environment. It defines classpaths, script names, filesystem locations, HDFS directories, Hive table identifiers, Sqoop queries, Java class references, Oracle/Impala connection details, email routing, and ancillary service endpoints. All batch jobs that ingest, transform, or export Move‑Mediation data (SIM inventory, Geneva billing, PPU reports, traffic‑details, etc.) source this file to obtain runtime constants, ensuring consistent pathing, logging, and external‑system connectivity across the Java‑based and shell‑based pipelines.

# Key Components
- **Classpath Definition** – `CLASSPATHVAR` points to Hive, Hadoop, and generic JAR libraries required by Java batch jobs.  
- **Script Identifiers** – `MNAASSimInventorySqoopScriptName`, `MNAASDailyGenevaLoadScriptName`, `ppu_daily_report_Scriptname` used by cron wrappers.  
- **Filesystem Roots** – `MNAASLocalPath`, `MNAASJarPath`, `MNAASLocalLogPath`, `MNAASConfPath`, `BackupDir`, `MNAASInterFilePath`, etc.  
- **HDFS Staging & Load Paths** – `MNAASPathName`, `MNAAS_Rawtablesload_PathName`, `MNAAS_Daily_Rawtablesload_SimInventory_PathName`, `SqoopPath`, `HDFSSqoopDir`.  
- **Hive Table Metadata** – `dbname`, `Move_siminventory_status_inter_tblname`, `Move_siminventory_status_tblname`, `traffic_details_api_tblname`, etc.  
- **Java Class References** – `Load_nonpart_table`, `drop_partitions_in_table`, `DropNthDaysPartition`, etc. (used via `java -cp`).  
- **Sqoop Queries** – `sim_inventory_SqoopQuery`, `customer_master_SqoopQuery`.  
- **Database Connectivity** – Oracle (`ora_serverNameMOVE`, `ora_portNumberMOVE`, `ora_serviceNameMOVE`, `ora_usernameMOVE`, `ora_passwordMOVE`) and Impala (`IMPALAD_HOST`, `IMPALAD_JDBC_PORT`).  
- **Email Notification Settings** – multiple `*_mail_*` groups for SIM inventory, Geneva, error, reconciliation, export, and RA reports.  
- **External Service Endpoints** – Kafka (`kafka_server`, `kafka_port`), Geneva (`Geneva_Host`, `Geneva_port`, `Geneva_path`), DataLake (SSH credentials & destination paths).  
- **Process‑Status Files** – paths such as `MNAASDailyGenevaLoad_ProcessStatusFile`, `ppu_daily_report_ProcessStatusFileName`, `SIMLoaderConfFilePath`.  

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| **SIM Inventory Load** | Raw CSV files in `$MNAASMainStagingDirDaily` | Sqoop import using `sim_inventory_SqoopQuery` → Hive staging table (`move_sim_inventory_status_inter`) | HDFS files under `$MNAAS_Daily_Rawtablesload_SimInventory_PathName`; status file updated; logs written to `$sim_inventory_SqoopLog`. |
| **Geneva Billing Load** | Geneva files in `$MNAAS_Billing_Geneva_FilesPath` | Java `GenevaLoader.jar` (or PPU variant) reads files, transforms, writes to Hive tables (`geneva_*`) | Hive tables populated; logs to `$GenevaLoaderJavaLogPath`; email notifications via `geneva_mail_*`. |
| **PPU Daily Report** | Hive tables `gen_daily_subs_ppu_summary` & `gen_daily_usage_ppu_summary` | Sqoop export using queries defined in `ppu_daily_report_*_Query` | Exported files to `$SqoopPath/...`; status file updated; email to `export_mail_*`. |
| **Traffic Details Aggregation** | Raw Hive tables (`api_med_data`) | Java classes (`LoadNonPartitionTable`, `DropNthDaysPartition`, etc.) executed via `$MNAAS_Main_JarPath` | Partitioned Hive tables refreshed; logs to `$GenevaLoaderJavaLogPath`; email on errors. |
| **Reconciliation** | CSVs in `$recon_source_path` | Hive load scripts (referenced by other property files) ingest into `$recon_csv_path` → final tables | Reconciliation tables updated; alerts via `recon_mail_to`. |
| **Backup & Retention** | Files in `$MNAASLocalPath` and HDFS | Periodic copy to `$BackupDir`, removal of empty/rejected files, partition drop scripts | Backup directories populated; HDFS space reclaimed. |
| **Messaging** | Processed events | Kafka producer (configured via `kafka_server:port`) | Messages on topic(s) for downstream consumers. |
| **DataLake Transfer** | Final report files | SCP using `$Datalake_user`/`$Datalake_password` to `$Datalake_*_path` | Files landed in Pentaho inbound directories. |

# Integrations
- **Shell Cron Wrappers** (`*.sh` scripts) source this properties file to resolve all paths and variables.  
- **Hive / Impala** – Table names and partition management classes are invoked from Java jobs; Impala JDBC used for direct DML (`traffic_details_api_tblname_truncate/load`).  
- **Sqoop** – Queries defined here are passed to Sqoop import/export commands executed by the scripts.  
- **Java Batch Jobs** – `MNAAS_Main_JarPath` contains the driver class that reads the property file (`Mnaas_java_propertypath`) to locate class names and table identifiers.  
- **Oracle DB** – Connection parameters used by Sqoop for source data extraction (`ora_*`).  
- **Kafka** – Producer configuration referenced by Java jobs for event streaming.  
- **Geneva Billing System** – File transfer via SCP to `$Geneva_path`; subsequent Java loader consumes those files.  
- **DataLake (Pentaho)** – Final reports are pushed via SCP using the DataLake credentials.  
- **Email System** – All notification groups use the same SMTP host (`mail_host`).  

# Operational Risks
- **Hard‑coded credentials** (`ora_passwordMOVE`, `Datalake_password`) – risk of credential leakage.