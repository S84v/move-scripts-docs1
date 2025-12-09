# Summary
`mnaas_generic_recon_loading.properties` defines runtime constants for the **generic reconciliation loading** batch job in the Move‑Mediation production environment. It imports shared property files, declares script‑level variables, HDFS/Hive paths, partition‑management SQL statements, and email distribution lists. The values are consumed by `mnaas_generic_recon_loading.sh` to orchestrate daily/monthly reconciliation of multiple telecom processes (eSIMHub, VoLTE, Pre‑Active, MNP Port‑In/Out, Interconnect).

# Key Components
- **Property Imports**  
  - `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/mnaaspropertiese.prop`  
  - `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Script Metadata**  
  - `mnaas_generic_recon_scriptname` – name of the driver shell script.  
  - `mnaas_generic_recon_processstatusfilename` – status‑file location.  
  - `mnaas_generic_recon_loadinglogname` – log file (timestamped).  
- **Intermediate File Paths**  
  - Daily/Monthly recon detail directories (`mnaasinterfilepath_*`).  
- **HDFS Load Destination**  
  - `mnaas_recon_load_generic_pathname` – target raw‑load directory.  
- **Hive Table Names**  
  - Temporary insert table (`dname_mnaas_load_generic_recon_tbl_temp`).  
  - Intermediate recon table (`generic_recon_inter_tblname`).  
  - Consolidated report table (`generic_recon_report`).  
- **Partition Management Queries**  
  - `*_partition_date_query` – INSERT‑OVERWRITE to extract distinct partition dates per process.  
  - `*_Drop_Partitions` – ALTER statements to drop existing partitions before reload.  
  - `*_Insert_Partitions` – Dynamic‑partition INSERT statements that populate `generic_recon_report`.  
- **Report Generation**  
  - CSV output path (`mnaas_generic_recon_report_files`).  
  - Header definition (`mnaas_generic_header`).  
  - Process‑specific recon report queries (`*_recon_report_query`).  
- **Backup & Notification**  
  - `generic_backupdir` – HDFS backup location per process/filetype.  
  - `ccList` – email recipients for job notifications.  

# Data Flow
| Stage | Input | Transformation | Output / Side‑Effect |
|-------|-------|----------------|----------------------|
| **Pre‑load** | Shared property files, environment variables (`MNAASConfPath`, `logdir`, `backuplocation`) | Variable interpolation, date‑stamp generation | Populated shell variables |
| **Intermediate Files** | Raw reconciliation files placed in `${processname}_daily_Recon` or `${processname}_monthly_Recon` | Files copied/moved to `${mnaasinterfilepath_*}` | HDFS staging directories |
| **Partition Date Extraction** | Files in `${mnaas_recon_load_generic_pathname}` | Hive `INSERT OVERWRITE DIRECTORY` queries (`*_partition_date_query`) | `/user/recon/export/` directory containing distinct partition dates |
| **Drop Existing Partitions** | `generic_recon_report` Hive table | `ALTER TABLE … DROP IF EXISTS PARTITION` statements (`*_Drop_Partitions`) | Old partitions removed |
| **Insert New Partitions** | Intermediate recon tables (`*_recon_inter`) | Dynamic‑partition INSERT (`*_Insert_Partitions`) | Populated `generic_recon_report` with file metadata and partition keys |
| **Report Generation** | `generic_recon_report` | Process‑specific `*_recon_report_query` executed via Hive CLI | CSV file `${mnaas_generic_recon_report_files}` |
| **Backup** | HDFS load directory | `hdfs dfs -cp` (implicit in script) to `${generic_backupdir}` | Backup copy retained |
| **Notification** | Job status, log file | Email sent to `ccList` (handled by driver script) | Stakeholder alert |

External services: Hadoop HDFS, Hive Metastore, email (SMTP), optional logging infrastructure.

# Integrations
- **Driver Script**: `mnaas_generic_recon_loading.sh` sources this file to obtain all runtime constants.
- **Common Property Files**: `mnaaspropertiese.prop` and `MNAAS_CommonProperties.properties` provide base paths (`MNAASConfPath`, `logdir`, `backuplocation`, etc.).
- **Hive Tables**: `mnaas.generic_recon_report`, `mnaas.<process>_recon_inter` (daily & monthly variants) are read/written.
- **HDFS Directories**: `/user/recon/export/`, `${processname}_daily_Recon`, `${processname}_monthly_Recon`, `${generic_backupdir}`.
- **Email System**: `ccList` used by the shell script’s mail utility for success/failure notifications.
- **Other Batch Jobs**: The same property pattern is used by GBS, eSIMHub, etc., ensuring consistent naming and path conventions across the Move‑Mediation suite.

# Operational Risks
- **Missing/Incorrect Environment Variables** – `MNAASConfPath`, `logdir`, `backuplocation` must be defined; otherwise path resolution fails. *Mitigation*: Validate variables at script start, abort with clear error.
- **Date‑Parsing Errors** – Partition queries rely on fixed filename token positions (`split(filename,'_')[n]`). Changes to upstream file naming break extraction. *Mitigation*: Centralize filename parsing logic or use regex with validation.
- **Partition Drop Collisions** – Concurrent runs could drop partitions needed by another process. *Mitigation*: Serialize jobs per `processname` or use lock files.
- **Hive Dynamic Partition Limits** – Large numbers of partitions may exceed Hive limits. *Mitigation*: Tune `hive.exec.max.dynamic.partitions` or prune old partitions regularly.
- **Email Delivery Failure** – Notification list may become stale. *Mitigation*: Externalize email list to a config service and monitor bounce rates.

# Usage
```bash
# From the Move‑Mediation environment
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/mnaaspropertiese.prop
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source move-mediation-scripts/config