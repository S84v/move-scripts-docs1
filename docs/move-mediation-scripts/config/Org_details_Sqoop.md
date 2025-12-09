# Summary
Defines Bash‑sourced runtime constants for the **Org_details_Sqoop** job. The job extracts organization master data from the Oracle `organization` table via Sqoop, writes the CSV output to a designated HDFS staging directory, and logs execution. Constants include common property imports, log file path, HDFS target directory, and the Sqoop free‑form query.

# Key Components
- **MNAAS_CommonProperties.properties** – imported shared environment variables (e.g., `$MNAASLocalLogPath`, `$CONDITIONS`).
- **Org_Details_SqoopLog** – full path of the job‑specific log file, timestamped per run.
- **Org_SqoopDir** – HDFS target directory for the Sqoop export (`/user/MNAAS/sqoop/Org_details`).
- **Org_DetailsQuery** – Oracle SQL query executed by Sqoop; filters active organizations and respects `$CONDITIONS` for Sqoop split‑by predicates.

# Data Flow
1. **Input**: Oracle `organization` table (via JDBC). Query limited to active rows (`org_inact_dt IS NULL OR org_inact_dt >= SYSDATE`) and Sqoop split conditions.
2. **Processing**: Sqoop imports rows as delimited text (default `,`), using `$CONDITIONS` for parallelism.
3. **Output**: HDFS directory `$Org_SqoopDir` containing one or more part files.
4. **Side Effects**: Append execution details to `$Org_Details_SqoopLog`; possible generation of `_SUCCESS` marker in HDFS.
5. **External Services**: Oracle DB, HDFS NameNode/DataNodes, Hadoop/YARN resource manager.

# Integrations
- **Driver Script**: `Org_details_Sqoop.sh` (or similarly named) sources this properties file to obtain constants.
- **Common Property File**: `MNAAS_CommonProperties.properties` supplies base paths, Hadoop configuration, and `$CONDITIONS` placeholder.
- **Downstream Jobs**: Any Hive/Impala load or reporting job that consumes the CSV files placed in `$Org_SqoopDir`.

# Operational Risks
- **Query Drift**: Schema changes in `organization` table may break column list; mitigate with schema version control and unit tests.
- **Missing `$CONDITIONS`**: If not set, Sqoop may default to a single mapper, causing performance bottlenecks; ensure `$CONDITIONS` is defined in common properties.
- **Log Rotation**: Log file includes date but no rotation; large logs can fill local disk; implement logrotate or size‑based truncation.
- **HDFS Permissions**: Incorrect ACLs on `$Org_SqoopDir` can cause job failure; enforce proper directory ownership (`MNAAS` user).

# Usage
```bash
# Source common and job‑specific properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/Org_details_Sqoop.properties

# Execute Sqoop (example)
sqoop import \
  --connect "$ORACLE_JDBC_URL" \
  --username "$ORACLE_USER" \
  --password "$ORACLE_PWD" \
  --query "$Org_DetailsQuery" \
  --target-dir "$Org_SqoopDir" \
  --as-textfile \
  --split-by org_no \
  --num-mappers 4 \
  --fields-terminated-by ',' \
  --lines-terminated-by '\n' \
  --verbose \
  >> "$Org_Details_SqoopLog" 2>&1
```
To debug, run the Sqoop command with `--verbose` and inspect `$Org_Details_SqoopLog`.

# Configuration
- **Environment Variables (from common properties)**
  - `MNAASLocalLogPath` – base local log directory.
  - `CONDITIONS` – Sqoop split predicate placeholder (e.g., `AND \$CONDITIONS`).
  - `ORACLE_JDBC_URL`, `ORACLE_USER`, `ORACLE_PWD` – Oracle connection details.
- **Referenced Config Files**
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
  - This file itself (`Org_details_Sqoop.properties`).

# Improvements
1. **Parameterize Column List** – externalize column names to a property to simplify schema updates.
2. **Add Log Rotation Hook** – integrate `logrotate` configuration or script‑based size check to prevent uncontrolled log growth.