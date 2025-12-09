# Summary
`MNAAS_backup_file_retention_bkp.properties` is a shell‑sourced configuration file that supplies the **MNAAS_backup_file_retention.sh** job with environment‑specific constants: script metadata, log/status file locations, edge‑node backup host details, per‑customer retention periods (in days), file‑type glob patterns, and per‑customer edge‑2 backup directory names. The job uses these definitions to prune backup files on the edge‑2 storage node according to customer‑defined retention windows.

# Key Components
- **Source statement** – loads common properties from `MNAAS_CommonProperties.properties`.
- **Scalar variables**
  - `MNAAS_backup_file_retention_scriptName` – driver script name.
  - `MNAAS_file_retention_period_logpath` – full path of the job log (includes date suffix).
  - `MNAAS_file_retention_period_ProcessStatusFileName` – HDFS status‑file path used for coordination.
  - `MNAAS_edge2_backup_path` – root HDFS directory on edge‑2 where customer backups reside.
  - `MNAAS_edge2_user_name` – HDFS user for remote operations.
  - `MNAAS_edge_server_name` – hostname of the edge‑2 node.
- **Associative arrays**
  - `MNAAS_file_retention_period[customer]=days` – retention window per customer.
  - `MNAAS_dtail_extn[category]=glob` – filename patterns for each CDR category.
  - `MNAAS_edge2_customer_backup_dir[customer]=subdir` – sub‑directory under `MNAAS_edge2_backup_path` for each customer.

# Data Flow
| Phase | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| 1 | `MNAAS_CommonProperties.properties` (via `.`) | Populate shared variables (e.g., `MNAASLocalLogPath`, `MNAASConfPath`). | Global environment variables available to the driver. |
| 2 | `MNAAS_file_retention_period` & `MNAAS_edge2_customer_backup_dir` | For each customer, compute absolute backup directory: `<MNAAS_edge2_backup_path>/<subdir>`. | Paths used by `find`/`hdfs dfs -rm` to delete files older than `<days>` . |
| 3 | `MNAAS_dtail_extn` | Build glob patterns for each CDR type. | Pattern strings passed to `find -name` to restrict deletions. |
| 4 | HDFS edge‑2 node (`MNAAS_edge_server_name`) | Remote execution via `ssh`/`hdfs dfs` commands. | Deleted files, updated status file (`MNAAS_file_retention_period_ProcessStatusFileName`). |
| 5 | Log path (`MNAAS_file_retention_period_logpath`) | Append operational messages. | Persistent log file for audit. |

# Integrations
- **`MNAAS_CommonProperties.properties`** – provides base paths, Hadoop configuration, and common environment variables.
- **`MNAAS_backup_file_retention.sh`** – driver script that sources this file and implements the retention logic.
- **HDFS edge‑2 node** (`mtthdoped02`) – remote filesystem where backup directories reside.
- **HDFS status‑file mechanism** – coordination with other Move‑Mediation jobs (e.g., `MNAAS_backup_files_process`).
- **Potential downstream Hive/Impala tables** – retention may affect tables that reference backup files; not directly invoked but implied by pipeline contracts.

# Operational Risks
- **Incorrect retention values** – Over‑deletion of recent backups. *Mitigation*: Validate `MNAAS_file_retention_period` against a master data source before deployment.
- **Duplicate keys in associative arrays** (e.g., `SkTelink` appears twice). *Mitigation*: Consolidate definitions; enforce uniqueness via linting.
- **Hard‑coded hostnames/user** – Breaks in case of node rename or credential change. *Mitigation*: Externalize to a central properties file or use DNS aliases.
- **Glob pattern mismatches** – Files not matched and therefore never purged. *Mitigation*: Unit‑test patterns against a sample directory; log unmatched files.
- **Missing common properties** – Failure to source `MNAAS_CommonProperties.properties` leads to undefined variables. *Mitigation*: Add guard clause to abort if source fails.

# Usage
```bash
# From the Move‑Mediation scripts directory
cd /app/hadoop_users/MNAAS/MNAAS_Property_Files
# Source the config (normally done inside the driver)
. ./MNAAS_backup_file_retention_bkp.properties

# Run the driver in debug mode
bash -x /app/hadoop_users/MNAAS/move-mediation-scripts/MNAAS_backup_file_retention.sh
```
To test a single customer’s retention logic:
```bash
CUSTOMER=MyRep
RETENTION=${MNAAS_file_retention_period[$CUSTOMER]}
DIR="${MNAAS_edge2_backup_path}/${MNAAS_edge2_customer_backup_dir[$CUSTOMER]}"
ssh ${MNAAS_edge2_user_name}@${MNAAS_edge_server_name} \
  "hdfs dfs -ls $DIR | grep -E '${MNAAS_dtail_extn[traffic]}' | \
   awk -v d=$RETENTION '{ if ( (system(\"date +%s\") - $6) > d*86400 ) print $8 }'"
```

# Configuration
- **Environment variables** (populated by `MNAAS_CommonProperties.properties`):
  - `MNAASLocalLogPath`
  - `MNAASConfPath`
- **External config files**:
  - `MNAAS_CommonProperties.properties` – base Hadoop and logging settings.
  - `MNAAS_Create_New_Customers.properties` – may extend associative arrays at runtime.
- **Hard‑coded values**:
  - `MNAAS_edge2_backup_path=/backup1/MNAAS/Customer_CDR`
  - `MNAAS_edge2_user_name=hdfs`
  - `MNAAS_edge_server_name=mtthdoped02`

# Improvements
1. **Externalize mutable maps** – Store `MNAAS_file_retention_period`, `MNAAS_dtail_extn`, and `MNAAS_edge2_customer_backup_dir` in a JSON/YAML file and load via `jq` or `yq` to simplify updates and enable validation.
2. **Add validation routine** – Implement a function that checks for duplicate keys, missing required keys, and that retention days are positive integers; abort early with a clear error message.