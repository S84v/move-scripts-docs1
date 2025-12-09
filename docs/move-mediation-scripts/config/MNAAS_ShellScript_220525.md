**# Summary**  
`MNAAS_ShellScript_220525.properties` is the central configuration repository for the MNAAS (Move‑Network‑As‑a‑Service) daily‑processing pipeline. It defines all filesystem locations, HDFS paths, Hive/Impala connection details, Java class names, script names, partition‑retention policies, and operational parameters used by the suite of shell, Java, and Sqoop jobs that ingest, cleanse, aggregate, and back‑up telecom CDR, activation, tolling, and SIM‑inventory data for multiple downstream partners (MyRepublic, CHT, SkyRoam, etc.). The file is read by the various cron‑driven scripts under `move‑mediation‑scripts` to drive end‑to‑end data movement from landing zones to Hive tables, generate daily aggregates, perform sequence‑number checks, and produce partner‑specific reports.

---

**# Key Components**

- **Classpath definition** – `CLASSPATHVAR` points to Hive/Hadoop libraries and custom JARs.  
- **Directory roots** – `MNAASLocalPath`, `MNAASConfPath`, `MNAASJarPath`, `MNAASLocalLogPath`, `MNAASInterFilePath`, `BackupDir`, etc.  
- **Script name variables** – e.g. `MNAASDailyTollingAggregationScriptName`, `MNAASDailyTrafficDetailsLoadAggrSriptName`, `MNAASSimInventorySqoopScriptName`, `MNAASDailyReportScriptName`.  
- **HDFS staging & table locations** – `MNAASPathName`, `MNAAS_Rawtablesload_PathName`, `MNAAS_Aggregation_PathName`, partition column mappings.  
- **Hive/Impala connection parameters** – `nameNode`, `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`, `JDBC_DRIVER_NAME`, `HIVE_HOST`, `HIVE_JDBC_PORT`.  
- **Java class entry points** – e.g. `Insert_Part_Daily_table`, `MNAAS_usage_trend_aggregation_classname`, `MNAAS_MNAAS_seqno_check_classname`.  
- **Retention policies** – `mnaas_retention_period_*` values for each Hive table.  
- **Sqoop configuration** – Oracle connection blocks, queries (`Org_DetailsQuery`, `sim_inventory_SqoopQuery`, etc.), and process‑status file paths.  
- **Partner‑specific output directories** – `TrafficOutput`, `ActivesOutput`, … for MyRepublic, CHT, LinksField, SkyRoam, uCloudlink, SkTelink.  
- **Process‑status files** – one per major step (load, aggregation, backup, seq‑check) to enable idempotent cron execution.  
- **Notification e‑mail settings** – `notif_mail_*`, `siminv_mail_*`, `GTPMailId`, `ccList`.  
- **Kafka broker list** – `kafka_servers_list`.  
- **Re‑process trigger files** – paths under `Notification_ReProcess_Files` used by downstream notification handlers.

---

**# Data Flow**

| Stage | Input (source) | Processing | Output (target) | Side‑effects |
|-------|----------------|------------|----------------|--------------|
| **Landing** | Raw files in `/Input_Data_Staging/.../REP/` (traffic, activations, actives, tolling, sim‑inventory) | Shell scripts move files to staging dirs, generate “prev” snapshots | Files moved to `*_Prev` directories; status files updated |
| **Ingestion** | Files from staging dirs | Sqoop jobs (`MNAAS_Sqoop.sh`) import Org, SIM‑Inventory, GBS, Rate, Serv‑abbr data into HDFS `/user/MNAAS/sqoop/...` | HDFS tables populated; Sqoop logs written |
| **Raw‑table load** | HDFS raw files | Java `Insert_Part_Daily_table` (or `_Reject_`) loads into Hive raw tables (`*_raw_daily`) | Partition creation (`partition_date`); duplicate removal logs |
| **Sequence‑check** | Raw tables | Java `MNAAS_MNAAS_seqno_check_classname` validates monotonic seq‑numbers per partner | `*_seq_check.log` files; error notifications |
| **Aggregation** | Raw tables | Daily aggregation Java jobs (`Insert_Daily_Aggr_table`, `traffic_aggr_dups_remove`, etc.) produce hourly/daily aggregate tables (`*_aggr_daily`, `*_aggr_hourly`) | Aggregate tables refreshed; partition retention applied |
| **Block Consolidation** | Raw tables | `blk.jar` merges daily blocks, renames tables (`*_blk_tmp` → final) | Block‑load logs; renamed tables become visible |
| **Partner Export** | Aggregated tables | Shell scripts copy partitioned files to partner FTP directories (`/backup1/MNAAS/FilesforRepublic/...`) | Partner‑specific CSV/TSV files, backup copies |
| **Reporting** | Aggregated tables | `MNAAS_daily_reports.sh` generates daily summary reports, emails to `reporting_id` | Report files, email notifications |
| **Retention / Cleanup** | All Hive tables | Java `DropNthDaysPartition` / `DropNthOlderPartition` based on `mnaas_retention_period_*` | Old partitions dropped, metadata refreshed |

External services: Oracle DB (multiple schemas), HDFS, Hive/Impala, Kafka (for notifications), SMTP (mail alerts), partner FTP servers.

---

**# Integrations**

- **Sqoop → Oracle** – pulls reference data (org, SIM inventory, GBS) using the Oracle connection blocks.  
- **Hive/Impala** – all Java loading/aggregation classes use the Hive JDBC driver (`org.apache.hive.jdbc.HiveDriver`) to execute DDL/DML.  
- **Kafka** – `kafka_servers_list` is referenced by notification scripts that publish processing status/events.  
- **Mail** – `notif_mail_*` and `siminv_mail_*` are consumed by error‑handling scripts to send alerts.  
- **Partner FTP** – backup/export directories are mounted or accessed by downstream FTP push jobs (outside this repo).  
- **Cron scheduler** – each script name variable is used by a crontab entry that sources this properties file, checks the corresponding process‑status file, and executes the job.  

---

**# Operational Risks**

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale classpath** – hard‑coded CDH 5/7 jar paths may become invalid after upgrades. | Job failures, missing libraries. | Externalize classpath to a version‑agnostic variable; validate existence at start. |
| **Hard‑coded partition retention** – negative values (`-1`)