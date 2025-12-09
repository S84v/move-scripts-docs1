**# Summary**  
`MNAAS_Environment.properties` centralises runtime constants for the Move‑Mediation production suite. It defines environment mode, class‑path, HDFS/Hive/Impala endpoints, job scheduling parameters, data‑retention policies, Oracle and Tableau connection credentials, email distribution lists, SSH move‑job settings, Hive database/table identifiers, and ancillary service configurations. All batch scripts source this file to obtain consistent configuration across the daily SIM inventory, tolling, validation, and related pipelines.

**# Key Components**  

- **ENV_MODE** – execution profile (`PROD`).  
- **CLASSPATHVAR** – Hadoop/Hive/Impala JAR class‑path (multiple versions, final active definition).  
- **HDFS / Hive / Impala endpoints** – `nameNode`, `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`, `HIVE_HOST`, `HIVE_JDBC_PORT`, `JDBC_DRIVER_NAME`.  
- **Job runtime parameters** – `hour_var`, `min_var`, `No_of_files_to_process`, `No_of_backup_files_to_process`.  
- **Retention period variables** – `mnaas_retention_period_*` for each table/partition.  
- **Oracle DB connection blocks** – `ora_*` for GBS, DEV, MOVE, REPC and generic `OrgDetails_*`.  
- **Tableau security block** – `tableau_security_*`.  
- **Email distribution lists** – `GTPMailId`, `reporting_id`, `mxrelay_vsnl_co_in`, `SDP_*`, `MOVE_DEV_TEAM`, `ccList`.  
- **Move‑files SSH block** – `SSH_MOVE_USER`, `SSH_MOVE_DAILY_SERVER`, `SSH_MOVE_DAILY_SERVER_PORT`, Telena feed variables.  
- **MyRep alert block** – alert sender/recipient/CC lists.  
- **Time‑zone constants** – `SNG`, `HOL`.  
- **Hive database & table name mappings** – `dbname`, `org_details_tblname`, `traffic_details_daily_tblname`, … (full list of >70 table identifiers).  
- **KYC weekly tables** – `kyc_weekly_inter_tblname`, `kyc_iccid_wise_country_hist_tblname`.

**# Data Flow**  

| Category | Input | Output / Side‑Effect | External Service |
|----------|-------|----------------------|-----------------|
| Job scheduling | `hour_var`, `min_var` | Determines cron trigger time for daily jobs | OS scheduler (cron) |
| Data ingestion | HDFS `nameNode`, source file locations (derived from other scripts) | Populates raw Hive tables (e.g., `siminventory_daily_tblname`) | HDFS, Hive/Impala |
| Retention cleanup | Retention variables | `ALTER TABLE … DROP PARTITION` statements executed via Hive/Impala | Hive Metastore, Impala |
| Oracle sync | `ora_*` credentials | Data exported/imported via Sqoop or custom JDBC calls | Oracle DB |
| Email notifications | Email lists | Sends success/failure alerts | SMTP relay (`mxrelay_vsnl_co_in`) |
| Remote file move | SSH variables, Telena feed vars | Files transferred to Windows host via `scp`/`sftp` | Remote Windows server (`MNAAS_Telena_feed_windows_server`) |
| Alerts | MyRep alert lists | Sends alert emails on replication issues | SMTP |

**# Integrations**  

- **Shell scripts** (`*.sh` in `move-mediation-scripts/config/…`) source this file via `. /path/MNAAS_Environment.properties`.  
- **Hive/Impala jobs** read table name constants to build dynamic DDL/DML.  
- **Sqoop/Custom JDBC** modules reference Oracle blocks for data staging.  
- **Email modules** use the defined recipient/CC variables for reporting.  
- **SSH move utility** (`MNAAS_move_files_from_staging.sh`) consumes SSH block for remote copy.  
- **MyRep alert subsystem** reads `MyRep_alert_*` lists for failure notifications.  

**# Operational Risks**  

- **Credential exposure** – passwords stored in clear text. *Mitigation*: migrate to a secret manager (e.g., HashiCorp Vault) and reference via token.  
- **Class‑path drift** – multiple commented CLASSPATH definitions may cause confusion. *Mitigation*: retain only the active definition and version‑control changes.  
- **Hard‑coded hostnames/IPs** – static Impala/Hive endpoints. *Mitigation*: externalize to DNS or service discovery.  
- **Retention negative values** (`-1`) may be misinterpreted. *Mitigation*: document semantics (e.g., “retain indefinitely”).  
- **Email list duplication** – overlapping CC lists can cause spam. *Mitigation*: centralise list generation.  

**# Usage**  

```bash
# Example: source the properties before running a batch script
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Environment.properties
# Verify a variable
echo "$CLASSPATHVAR"
# Run a specific job (debug mode)
bash -x MNAAS_Daily_SimInventory_tbl_Aggr.sh
```

**# Configuration**  

- **Primary file**: `move-mediation-scripts/config/MNAAS_Environment.properties`.  
- **Imported common file** (via other property files): `MNAAS_CommonProperties.properties`.  
- **Environment variable**: `ENV_MODE` (must be `PROD` for production).  
- **External config references**: none; all values are defined inline.  

**# Improvements**  

1. **Secure secrets** – replace plain‑text passwords with references to an encrypted vault or OS‑level key store.  
2. **Modularise class‑path** – generate `CLASSPATHVAR` programmatically based on installed CDH version to avoid manual edits and commented dead code.  