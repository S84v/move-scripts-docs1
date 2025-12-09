# Summary
`MNAAS_GBS_Journal_Sqoop_Load.properties` supplies runtime constants for the GBS Journal Sqoop Load batch job in the Move‑Mediation production environment. It imports the shared `MNAAS_CommonProperties.properties` file and defines the local filesystem path for the Sqoop job’s log output (`gbs_Details_SqoopLog`). These constants are consumed by the corresponding shell script to drive a Sqoop import of GBS journal data into HDFS/Hive.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – source statement that injects all common configuration variables (e.g., Hadoop classpath, DB connection strings, environment mode).
- **`gbs_Details_SqoopLog`** – absolute path (`/app/hadoop_users/MNAAS/MNAASCronLogs/gbs_Details_sqooplogs`) where the Sqoop job writes its execution log.

# Data Flow
| Phase | Input | Process | Output | Side Effects |
|-------|-------|---------|--------|--------------|
| 1 | Source DB credentials & connection strings (from common properties) | Sqoop import command (executed by `MNAAS_GBS_Journal_Sqoop_Load.sh`) | Staged GBS journal files in HDFS directory (defined in common properties) | Log file written to `gbs_Details_SqoopLog` |
| 2 | HDFS staging directory | Hive `LOAD DATA` (or external table) performed by downstream scripts | Populated Hive table (`gbs_journal_tblname` – defined in common properties) | None beyond Hive metadata update |

# Integrations
- **Common Properties**: Inherits all variables required for Hadoop, Hive, Oracle, and email configuration.
- **Shell Wrapper** (`MNAAS_GBS_Journal_Sqoop_Load.sh`): Sources this file to resolve `gbs_Details_SqoopLog` and any common variables.
- **Sqoop**: Uses the log path for stdout/stderr redirection.
- **Hive**: Consumes the HDFS staging directory (from common properties) to load data into the target Hive table.

# Operational Risks
- **Missing Common Properties**: Failure to source the common file results in undefined DB credentials → job abort. *Mitigation*: Verify file existence before sourcing; exit with clear error.
- **Log Directory Permission Issues**: Inadequate write permissions on `/app/hadoop_users/MNAAS/MNAASCronLogs` cause log write failures, masking underlying errors. *Mitigation*: Enforce ACLs and monitor directory health.
- **Path Hard‑coding**: Absolute paths reduce portability across environments. *Mitigation*: Parameterize base directories via common properties.

# Usage
```bash
# From the job’s execution directory
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_GBS_Journal_Sqoop_Load.properties

# Run the Sqoop load (example command inside the wrapper script)
sqoop import \
  --connect ${oracle_jdbc_url} \
  --username ${oracle_user} \
  --password ${oracle_pass} \
  --table ${gbs_journal_tblname} \
  --target-dir ${gbs_journal_hdfs_dir} \
  > ${gbs_Details_SqoopLog}/$(date +%Y%m%d_%H%M%S).log 2>&1
```
To debug, tail the latest log file in `${gbs_Details_SqoopLog}`.

# Configuration
- **Environment Variables**: None defined locally; all required variables (e.g., `oracle_jdbc_url`, `gbs_journal_tblname`, `gbs_journal_hdfs_dir`) are provided by `MNAAS_CommonProperties.properties`.
- **Referenced Config Files**:  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` (mandatory).  

# Improvements
1. **Parameterize Log Base Path** – Introduce a variable (e.g., `MNAAS_LOG_BASE`) in common properties and construct `gbs_Details_SqoopLog` from it to avoid hard‑coded absolute paths.
2. **Add Validation Block** – Append a sanity‑check section that verifies required common variables are defined and that the log directory is writable; abort with a descriptive message if checks fail.