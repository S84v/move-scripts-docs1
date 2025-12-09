**# Summary**  
`MNAAS_ShellScript.properties` is the central configuration repository for the MNAAS (Move‑Network‑As‑A‑Service) daily‑processing pipeline. It defines classpaths, HDFS and local filesystem locations, script names, Hive table metadata, Java class entry points, Sqoop queries, retention policies, and notification settings used by all shell‑scripts, Java jobs, and Sqoop jobs that ingest, transform, aggregate, and back‑up telecom CDR, activation, actives, tolling and SIM‑inventory data.

**# Key Components**  

- **Classpath definition (`CLASSPATHVAR`)** – Hadoop, Hive, and custom JAR locations for all Java jobs.  
- **Directory variables** – `MNAASLocalPath`, `MNAASConfPath`, `MNAASJarPath`, `MNAASLocalLogPath`, `BackupDir`, etc., centralising all file‑system roots.  
- **Script name mappings** – e.g. `MNAASDailyTollingAggregationScriptName`, `MNAASDailyTrafficDetailsLoadAggrSriptName`, `MNAAS_SimInventory_seq_check_ProcessStatusFilename`.  
- **Hive table metadata** – database name (`dbname`), table names, partition columns, and refresh statements.  
- **Java job class names** – e.g. `Insert_Part_Daily_table`, `MNAAS_usage_trend_aggregation_classname`.  
- **Sqoop configuration** – source DB credentials, queries (`Org_DetailsQuery`, `sim_inventory_SqoopQuery`), target HDFS paths.  
- **Retention policies** – `mnaas_retention_period_*` values for each Hive table.  
- **Notification settings** – email recipients, SMTP host, and mail templates for alerts.  
- **System properties** – Hadoop NameNode (`nameNode`), Impala/ Hive hosts and ports, JDBC driver.  
- **Process‑status files** – one file per logical step to enable idempotent orchestration.  

**# Data Flow**  

| Stage | Input | Processing | Output / Side‑effects |
|-------|-------|--------------|------------------------|
| **Ingestion** | Raw CSV files from `/Input_Data_Staging/...` (traffic, activations, actives, tolling, SIM inventory) | Shell scripts invoke Java loaders (`Insert_Part_Daily_table`, etc.) and Sqoop imports (`Org_DetailsQuery`, `sim_inventory_SqoopQuery`) | HDFS raw tables under `/user/MNAAS/RawtablesLoad/Daily/...`; process‑status files updated |
| **Deduplication / Validation** | Raw tables | Java classes (`traffic_aggr_dups_remove`, `MNAAS_MNAAS_seqno_check_classname`) | Cleaned tables, duplicate‑removal logs, seq‑check logs |
| **Aggregation** | Cleaned raw tables | Java aggregation jobs (`Insert_Daily_Aggr_table`, `MNAAS_usage_trend_aggregation_classname`, etc.) | Hive aggregation tables under `/user/MNAAS/Aggregation/Daily/...`; refresh statements executed |
| **Backup / Export** | Aggregated tables | Shell scripts copy files to `/backup1/MNAAS/...` and FTP directories per partner (MyRepublic, CHT, SkyRoam, etc.) | Physical backup files, partner‑specific output dirs, backup‑status process files |
| **Reporting** | Aggregated tables | `MNAAS_daily_reports_ProcessStatusFile` driven script generates CSV/HTML reports and emails them | Report files in log path, email notifications |
| **Retention** | All Hive tables | Java `DropNthDaysPartition` / `DropNthOlderPartition` jobs run using retention period constants | Old partitions dropped, metadata refreshed |

External services: Oracle (via Sqoop), Impala/Hive (via JDBC), Kafka (for notifications), SMTP mail server.

**# Integrations**  

- **Shell orchestration** – All daily cron jobs source this properties file to resolve script names, paths, and status files.  
- **Java JARs** – Paths defined here are passed to `java -cp $CLASSPATHVAR` invocations.  
- **Sqoop** – Credentials and queries are read from this file; Sqoop jobs write to HDFS locations defined here.  
- **Impala/Hive** – `nameNode`, `IMPALAD_HOST`, `HIVE_HOST` used by JDBC connections in Java jobs.  
- **Kafka** – `kafka_server`/`kafka_port`/`kafka_servers_list` consumed by notification scripts.  
- **Email** – `GTPMailId`, `ccList`, `notif_mail_*` used by alert scripts after each stage.  
- **Backup scripts** – Use `BackupDir` hierarchy to stage partner‑specific files.  

**# Operational Risks**  

1. **Stale classpath** – Hard‑coded parcel paths may become invalid after CDH upgrade. *Mitigation*: externalise classpath to a version‑agnostic symlink or use environment modules.  
2. **Process‑status file contention** – Multiple concurrent jobs may overwrite status files. *Mitigation*: enforce lock files or atomic `mv` updates.  
3. **Retention mis‑configuration** – Negative retention values (`-1`) could cause indefinite growth. *Mitigation*: periodic audit of `mnaas_retention_period_*` and automated cleanup scripts.  
4. **Credential leakage** – Plain‑text Oracle passwords in the file. *Mitigation*: move to a secure vault (e.g., HashiCorp Vault) and reference via environment variables.  
5. **Hard‑coded email lists** – Changes in team composition require file edits, risking missed alerts. *Mitigation*: maintain mailing groups in LDAP and reference group address.  

**# Usage**  

```bash
# Load properties (example in a driver script)
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties

# Run a specific step (e.g., traffic details load)
bash $MNAASDailyTrafficDetailsLoadAggrSriptName   # uses variables from this file

# Debug: print a variable
echo "Raw traffic HDFS path: $MNAAS_Daily_Rawtablesload_TrafficDetails_PathName"
```

All scripts expect the properties file to be sourced before execution; missing variables cause immediate termination.

**# Configuration**  

- **Primary file**: `MNAAS_ShellScript.properties` (path referenced by `MNAAS_Property_filename`).  
- **Dependent files**: `MNAAS_Adhoc_Queries_for_users.properties`, various control files (`*_ProcessStatusFile`, `*_CntrlFile.properties`).  
- **Environment variables**: `JAVA_HOME`, `HADOOP_CONF_DIR`, `HIVE_CONF_DIR` must be set externally.  
- **External services**: Oracle DBs (`ora_*`), Hadoop NameNode (`nameNode`), Impala (`