# Summary
`MNAAS_backup_SDP_ticket_validation.properties` is a shell‑sourced configuration file that provides environment‑specific constants for the `MNAAS_backup_SDP_ticket_validation.sh` driver. It defines script metadata, log and status file locations, Hive database/table references, backup root paths, email notification lists, Windows server credentials, and per‑customer backup directory and job‑mapping tables used during the SDP ticket validation and backup verification step of the Move‑Mediation pipeline.

# Key Components
- **Source of common properties** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  Loads shared variables (e.g., `MNAASConfPath`, `MNAASLocalLogPath`).
- **Script metadata variables** – `MNAAS_backup_SDP_ticket_ScriptName`, `MNAAS_backup_SDP_ticket_ProcessStatusFileName`, `MNAAS_backup_SDP_ticket_logpath`.
- **Hive integration variables** – `MNAAS_backup_SDP_ticket_db`, `MNAAS_backup_SDP_ticket_table`.
- **Backup path variables** – `MNAAS_backup_SDP_ticket_parent_path`, `MNAAS_backup_SDP_ticket_attachment`.
- **Email notification variables** – `MNAAS_WINDOWS_TO_EMAIL`, `MOVE_TEAM_EMAIL`, `MNAAS_WINDOWS_CC_EMAIL`.
- **Windows server access variables** – `MNAAS_windows_server`, `MNAAS_windows_user`.
- **Associative array `MNAAS_customer_backup_dir`** – maps logical customer identifiers to their edge‑node backup directory names.
- **Associative array `MNAAS_windows_job_mapping`** – maps customer identifiers to the job string format expected by the Windows backup job scheduler.

# Data Flow
| Stage | Input | Processing | Output / Side Effect |
|-------|-------|------------|----------------------|
| Script start | Environment (common properties, this file) | Variables are exported to the shell environment | Configuration values available to `MNAAS_backup_SDP_ticket_validation.sh`. |
| Hive validation | `MNAAS_backup_SDP_ticket_db` & `MNAAS_backup_SDP_ticket_table` | Script queries Hive to count/validate customer file records. | Logs, status file, possible email alerts. |
| File system checks | `MNAAS_backup_SDP_ticket_parent_path`, `MNAAS_customer_backup_dir` | Traverses HDFS/edge‑node directories per customer. | Success/failure metrics written to status file. |
| Windows job verification | `MNAAS_windows_server`, `MNAAS_windows_user`, `MNAAS_windows_job_mapping` | SSH/SMB calls to Windows host to verify job existence/completion. | Logs, optional CSV attachment (`MNAAS_backup_SDP_ticket_attachment`). |
| Notification | Email lists (`MNAAS_WINDOWS_TO_EMAIL`, `MNAAS_WINDOWS_CC_EMAIL`) | Sends email with log/attachment on failure or completion. | Email sent to stakeholders. |

# Integrations
- **Common properties file** – provides base paths and generic settings used across all MNAAS scripts.
- **Hive** – accessed via `beeline`/`hive` CLI using the defined DB/table.
- **HDFS / Edge‑node storage** – backup directories under `/backup1/MNAAS/`.
- **Windows backup server** – remote host `66.198.167.215` accessed with user `cdrxfr` for job status checks.
- **Email subsystem** – uses system `mail`/`sendmail` to dispatch notifications.
- **Calling script** – `MNAAS_backup_SDP_ticket_validation.sh` sources this file and relies on all defined variables.

# Operational Risks
- **Credential exposure** – plain‑text Windows user/password may be leaked. *Mitigation*: store credentials in a vault or use key‑based SSH with restricted permissions.
- **Hard‑coded email addresses** – changes require file edit and redeploy. *Mitigation*: externalize to a central mailing list config.
- **Missing customer entries** – if a new customer is added without updating the associative arrays, validation will fail. *Mitigation*: implement a dynamic loader from a master customer registry.
- **Path drift** – changes to `MNAASConfPath` or `MNAASLocalLogPath` without updating this file cause log/status file misplacement. *Mitigation*: validate paths at script start.

# Usage
```bash
# Source the configuration (normally done inside the driver script)
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
. /path/to/move-mediation-scripts/config/MNAAS_backup_SDP_ticket_validation.properties

# Run the driver script with debug output
bash -x /app/hadoop_users/MNAAS/MNAAS_Scripts/MNAAS_backup_SDP_ticket_validation.sh
```
To debug variable values, insert `echo "$VAR"` after sourcing or run the script with `set -o xtrace`.

# Configuration
- **Referenced files**  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` (common base config).  
- **Environment variables expected** (populated by common properties)  
  - `MNAASConfPath` – directory for status files.  
  - `MNAASLocalLogPath` – base log directory.  
- **Local constants defined in this file** – listed in *Key Components*.

# Improvements
1. **Externalize credential handling** – replace `MNAAS_windows_user` with a token fetched from a secrets manager (e.g., HashiCorp Vault) and use key‑based SSH for Windows host access.  
2. **Dynamic customer map generation** – load `MNAAS_customer_backup_dir` and `MNAAS_windows_job_mapping` from a centralized JSON/YAML file or a Hive table to eliminate manual edits when onboarding new customers.