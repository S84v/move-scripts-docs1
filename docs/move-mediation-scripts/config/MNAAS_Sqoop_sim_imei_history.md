# Summary
`MNAAS_Sqoop_sim_imei_history.properties` is a Bash‑sourced configuration fragment that defines all runtime constants required by the **SIM IMEI History** Sqoop import step in the MNAAS daily‑processing pipeline. It supplies paths for process‑status tracking, logging, script identification, HDFS staging, target Hive table names, and the Oracle SELECT query executed by `MNAAS_Sqoop_sim_imei_history.sh` to import the `sim_imei_history` table into a staging Hive/Impala table.

# Key Components
- **Source of common properties** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- `SIM_IMEI_History_Sqoop_ProcessStatusFile` – HDFS/local file tracking job status.
- `SIM_IMEI_History_table_SqoopLogName` – Log file name (includes current date).
- `MNAAS_Sqoop_sim_imei_history_Scriptname` – Name of the invoking shell script.
- `sim_imei_history_table_Dir` – HDFS directory for staging files.
- `sim_imei_history_temp_table_name` – Temporary Hive table used during import.
- `sim_imei_history_table_name` – Final Hive table name.
- `sim_imei_history_table_Query` – Oracle SELECT statement with Sqoop `$CONDITIONS` placeholder.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| **Sqoop launch** (via `MNAAS_Sqoop_sim_imei_history.sh`) | Oracle DB `sim_imei_history` table, `$CONDITIONS` supplied by Sqoop for split‑by column | Sqoop import using `sim_imei_history_table_Query` | HDFS files in `sim_imei_history_table_Dir` → temporary Hive table `sim_imei_history_temp` → final Hive table `sim_imei_history` |
| **Status tracking** | N/A | Script writes job state (STARTED, SUCCESS, FAILED) | `SIM_IMEI_History_Sqoop_ProcessStatusFile` |
| **Logging** | N/A | Script appends stdout/stderr | `SIM_IMEI_History_table_SqoopLogName` |

External services:
- Oracle database (source)
- HDFS (staging)
- Hive/Impala (target)

# Integrations
- **Common properties file** (`MNAAS_CommonProperties.properties`) provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`, `SqoopPath`) and shared environment variables.
- **Sqoop driver script** (`MNAAS_Sqoop_sim_imei_history.sh`) sources this properties file to obtain configuration values.
- **Hive/Impala**: final tables are consumed by downstream analytics jobs in the MNAAS pipeline.
- **Monitoring/Orchestration**: status file is polled by the pipeline orchestrator to determine step completion.

# Operational Risks
- **Missing common properties** – job fails if `MNAAS_CommonProperties.properties` is unavailable or corrupted. *Mitigation*: validate file existence before sourcing.
- **Hard‑coded log date** – log rotation relies on date format; midnight roll‑over may cause duplicate logs. *Mitigation*: enforce log rotation policy or use a dedicated logging framework.
- **Credential exposure** – Oracle connection credentials are likely defined in the common properties file; improper permissions could leak secrets. *Mitigation*: restrict file permissions to the service account and audit access.
- **Query `$CONDITIONS` misuse** – incorrect split‑by column can cause data skew or job failure. *Mitigation*: enforce split‑by column validation in the driver script.
- **HDFS path changes** – any change to `SqoopPath` without updating this file breaks staging. *Mitigation*: centralize path definitions in the common properties file.

# Usage
```bash
# Source the configuration (normally done inside the driver script)
source /app/hadoop_users/MNAAS/move-mediation-scripts/config/MNAAS_Sqoop_sim_imei_history.properties

# Execute the Sqoop import script
bash $MNAAS_Sqoop_sim_imei_history_Scriptname
```
For debugging, enable Bash tracing:
```bash
set -x
source .../MNAAS_Sqoop_sim_imei_history.properties
bash -x $MNAAS_Sqoop_sim_imei_history_Scriptname
```

# Configuration
- **Environment variables / paths defined in `MNAAS_CommonProperties.properties`**  
  - `MNAASConfPath` – base directory for status files.  
  - `MNAASLocalLogPath` – base directory for log files.  
  - `SqoopPath` – base HDFS staging directory for all Sqoop jobs.  
- **Local constants (defined in this file)**  
  - `SIM_IMEI_History_Sqoop_ProcessStatusFile`  
  - `SIM_IMEI_History_table_SqoopLogName`  
  - `MNAAS_Sqoop_sim_imei_history_Scriptname`  
  - `sim_imei_history_table_Dir`  
  - `sim_imei_history_temp_table_name`  
  - `sim_imei_history_table_name`  
  - `sim_imei_history_table_Query`

# Improvements
1. **Externalize Oracle credentials** into a secure vault (e.g., Hadoop Credential Provider) and reference them via environment variables rather than plain‑text in the common properties file.  
2. **Parameterize the log filename** to include job run identifier (e.g., `$RUN_ID`) in addition to the date, preventing log collisions during rapid re‑runs or back‑fills.