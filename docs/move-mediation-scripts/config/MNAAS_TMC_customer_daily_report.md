# Summary
`MNAAS_TMC_customer_daily_report.properties` is a Bash‑sourced configuration fragment for the **TMC Customer Daily Report** step of the MNAAS mediation pipeline. It imports shared defaults, defines runtime constants (status‑file, log path, script name, HDFS and local file locations, Hive connection details, and the Hive QL query) that drive the `MNAAS_TMC_customer_daily_report.sh` job. The job extracts a filtered set of active SIM records from the `mnaas.actives_raw_daily` Hive table, writes the result as a CSV file to an HDFS staging directory (`/user/mkoripal/export/`), and makes the file available on the local filesystem for downstream consumption or archival.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – sources common environment variables (e.g., `MNAASConfPath`, `MNAASLocalLogPath`).
- **`setparameter='set -x'`** – optional Bash debug flag (comment to enable verbose tracing).
- **`MNAAS_TMC_customer_daily_report_ProcessStatusFileName`** – path to the job‑status tracking file.
- **`MNAAS_TMC_customer_daily_report_AggrLogPath`** – aggregated log file name (includes execution date).
- **`MNAAS_TMC_customer_daily_report_Script`** – name of the executable shell script.
- **`MNAAS_Active_feed_hdfs_location`** – HDFS target directory for the exported CSV.
- **`MNAAS_Active_feed_location` / `MNAAS_Active_feed_location_EV`** – local filesystem destinations for the exported CSV (standard and “EV” variant).
- **`T0_email`** – static notification recipient.
- **`HIVE_IP`** – HiveServer2 endpoint (host:port).
- **`Query`** – Hive QL statement that selects distinct active SIMs for a specific customer number and business unit, formats `activationdate`, and de‑duplicates by SIM using a window function.

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| 1. Configuration | `MNAAS_CommonProperties.properties` | Variable substitution (date, paths) | Populated Bash variables | None |
| 2. Hive extraction (performed by `MNAAS_TMC_customer_daily_report.sh`) | HiveServer2 (`HIVE_IP`) + `Query` | Execute `INSERT OVERWRITE DIRECTORY` → writes CSV to HDFS (`/user/mkoripal/export/`) | HDFS directory populated with `*.csv` | HDFS write permissions required |
| 3. Local copy (handled by the script) | HDFS CSV file | `hdfs dfs -get` or similar | Local files: `MNAAS_Active_feed_location` and `MNAAS_Active_feed_location_EV` | Requires local FS write permission |
| 4. Status & logging | Script execution | Append status to `*_ProcessStatusFileName`; write logs to `*_AggrLogPath` | Status file, log file | Log rotation may be needed |
| 5. Notification | Completion/failure | Send email to `T0_email` (if implemented) | Email sent | Email service availability |

# Integrations
- **Common Properties** – Provides base paths (`MNAASConfPath`, `MNAASLocalLogPath`) and possibly Hadoop/Hive client configuration.
- **`MNAAS_TMC_customer_daily_report.sh`** – Consumes all variables defined here; orchestrates Hive query execution, HDFS interaction, local file handling, status tracking, and logging.
- **HiveServer2** – Remote service at `HIVE_IP`; receives the `INSERT OVERWRITE DIRECTORY` command.
- **HDFS** – Destination for the intermediate CSV export (`/user/mkoripal/export/`).
- **Local Filesystem** – Archival locations for the final CSV extracts.
- **Email subsystem** – Uses `T0_email` for alerting (implementation resides in the shell script).

# Operational Risks
- **Hard‑coded date in file names** – If the job runs across midnight, mismatched dates may cause missing files. *Mitigation*: Use a consistent execution timestamp variable.
- **Static email address** – Change in personnel requires manual edit. *Mitigation*: Externalize to a configurable property or distribution list.
- **Hive query failure** – Syntax errors, missing tables, or permission issues will abort the job. *Mitigation*: Validate query in Hive console; add retry logic and error capture in the script.
- **HDFS permission/space constraints** – Export may fail if the target directory is unwritable or full. *Mitigation*: Pre‑flight check for write access and available quota.
- **Debug flag (`setparameter`) left enabled** – May generate excessive log volume. *Mitigation*: Document the need to comment it out for production runs.

# Usage
```bash
# 1. Source the configuration (optional for ad‑hoc testing)
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source /path/to/move-mediation-scripts/config/MNAAS_TMC_customer_daily_report.properties

# 2. Run the processing script
bash /path/to/move-mediation-scripts/scripts/MNAAS_TMC_customer_daily_report.sh

# 3. Enable Bash tracing for debugging (comment out the line in the .properties file)
# setparameter='set -x'   # <-- comment this line to turn on tracing
```

# Configuration
- **Referenced file**: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Environment variables expected from common properties**:
  - `MNAASConfPath`
  - `MNAASLocalLogPath`
- **Local overrides** (