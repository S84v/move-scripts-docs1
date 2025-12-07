**File:** `move-mediation-scripts\bin\MNAAS_SERV_ABBR_Load.sh`

---

## 1. Purpose (One‑Paragraph Summary)

This script extracts the “service abbreviation” reference data from an Oracle database using Sqoop, stages the raw files in a dedicated HDFS directory, and then refreshes the corresponding Hive/Impala table (`$dbname.$serv_abbr_tblname`). It first removes any previous staging files, runs the Sqoop import with a configurable query, truncates the target Hive table, loads the new data, and refreshes Impala metadata. Execution details are logged to a daily log file; on failure the script sends an email alert to the configured distribution list.

---

## 2. Key Elements & Responsibilities

| Element | Type | Responsibility |
|---------|------|-----------------|
| **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties`** | External config | Supplies all runtime variables (DB names, table names, HDFS paths, Oracle connection info, Impala host, email lists, etc.). |
| **`logger`** | Command | Writes timestamped messages to the per‑day Sqoop log (`$serv_abbr_SqoopLog_YYYY-MM-DD`). |
| **`hadoop fs -rm -r $serv_abbr_HDFSSqoopDir/*`** | HDFS command | Cleans the staging directory before each import. |
| **`sqoop import … --query "${serv_abbr_SqoopQuery}"`** | Sqoop command | Pulls data from Oracle into `$serv_abbr_HDFSSqoopDir`. Uses a single mapper (`-m 1`). |
| **`hive -S -e "truncate table $dbname.$serv_abbr_tblname"`** | Hive CLI | Empties the target Hive table to prepare for fresh load. |
| **`impala-shell -i $IMPALAD_HOST -q "refresh $dbname.$serv_abbr_tblname"`** | Impala CLI | Refreshes Impala metadata after Hive truncate and after load. |
| **`hive -S -e "load data inpath '$serv_abbr_HDFSSqoopDir' into table $dbname.$serv_abbr_tblname"`** | Hive CLI | Moves the staged files into the Hive table (managed or external, as defined by the table). |
| **`mail`** | Email utility | Sends a failure notification with log reference to `$GTPMailId` and CC list `$ccList`. |
| **`if [ $? -eq 0 ]; then … else … fi`** | Bash control flow | Determines success/failure of the Sqoop import and branches to normal processing or error handling. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | • Oracle connection details (`$ora_serverNameGBS`, `$ora_portNumberGBS`, `$ora_serviceNameGBS`, `$ora_usernameGBS`, `$ora_passwordGBS`).<br>• Sqoop query (`$serv_abbr_SqoopQuery`).<br>• HDFS staging directory (`$serv_abbr_HDFSSqoopDir`). |
| **Outputs** | • Daily log file: `$serv_abbr_SqoopLog_YYYY-MM-DD` (appended throughout execution).<br>• Populated Hive table `$dbname.$serv_abbr_tblname` (and refreshed Impala view). |
| **Side Effects** | • Deletes all files under `$serv_abbr_HDFSSqoopDir` before import.<br>• Truncates the Hive table (data loss if load fails).<br>• Sends an email on failure.<br>• Refreshes Impala metadata (affects any concurrent queries). |
| **Assumptions** | • The properties file exists and defines all referenced variables.<br>• The executing user has HDFS, Hive, Impala, and Oracle access rights.<br>• `sqoop`, `hadoop`, `hive`, `impala-shell`, and `mail` binaries are in `$PATH`.<br>• Oracle password is stored in plain text (as currently coded).<br>• The target Hive table is compatible with the Sqoop output format (e.g., delimited text). |

---

## 4. Interaction with Other Scripts / Components

| Interaction | Description |
|-------------|-------------|
| **Master orchestration scripts** (e.g., `MNAAS_Monthly_*` scripts) | Likely invoke this script as part of a nightly data‑load pipeline after prerequisite validation scripts (`MNAAS_PreValidation_Checks.sh`). |
| **Property file (`MNAAS_ShellScript.properties`)** | Central configuration shared across all MNAAS load scripts (table names, directories, DB credentials). |
| **Monitoring / Alerting** | Log file is consumed by log‑aggregation tools (Splunk, ELK) and the email alert integrates with the IPX team’s incident management system. |
| **Down‑stream consumers** | Any downstream reporting or analytics jobs that query `$dbname.$serv_abbr_tblname` in Hive/Impala assume the data is refreshed after this script runs. |
| **Credential management** | Currently uses plain‑text credentials; other scripts may have migrated to a vault – this script would need to be aligned. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Sqoop import failure** (network, query error, Oracle down) → table left empty after truncate. | Add a *pre‑load validation* step: run a `SELECT COUNT(*)` on the query before truncating. Consider loading into a temporary staging Hive table and swapping after successful load. |
| **HDFS delete (`rm -r`) runs on wrong path** → loss of unrelated data. | Validate `$serv_abbr_HDFSSqoopDir` exists and matches an expected pattern before deletion. Use `-skipTrash` flag only if intentional. |
| **Plain‑text Oracle credentials** → security breach. | Move credentials to a secure vault (e.g., HashiCorp Vault, AWS Secrets Manager) and retrieve them at runtime. |
| **Single mapper (`-m 1`)** → long run time, possible timeout. | Evaluate data volume; increase parallelism (`-m <n>`) after testing. |
| **Email flood on repeated failures** → alert fatigue. | Implement rate‑limiting or aggregate alerts (e.g., only send first failure per day, then a summary). |
| **Impala metadata refresh failures** → stale data for consumers. | Capture exit status of `impala-shell` commands; retry on transient errors. |
| **Log file growth** → disk fill. | Rotate logs daily (already using date suffix) and enforce retention policy via logrotate. |

---

## 6. Running & Debugging the Script

1. **Prerequisite Check**  
   ```bash
   ls -l /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties
   ```
   Verify the file contains all required variables.

2. **Execute** (normally invoked by the scheduler, but can be run manually):
   ```bash
   cd /path/to/move-mediation-scripts/bin
   ./MNAAS_SERV_ABBR_Load.sh
   ```

3. **Debugging**  
   - The script already enables `set -x` (trace mode). Review the console output or the generated log file:
     ```bash
     tail -f ${serv_abbr_SqoopLog}$(date +_%F)
     ```
   - To isolate a step, comment out later sections and re‑run. For example, test only the Sqoop import:
     ```bash
     sqoop import --connect ... --query "${serv_abbr_SqoopQuery}" --target-dir ${serv_abbr_HDFSSqoopDir} -m 1
     ```
   - Verify HDFS content:
     ```bash
     hadoop fs -ls ${serv_abbr_HDFSSqoopDir}
     ```
   - Verify Hive table state before/after load:
     ```bash
     hive -e "SELECT COUNT(*) FROM ${dbname}.${serv_abbr_tblname};"
     ```

4. **Exit Codes**  
   - The script exits with the status of the last command executed in the `if` block. For automation, capture `$?` after the script finishes.

---

## 7. External Configurations & Environment Variables

| Variable (from properties) | Meaning |
|----------------------------|---------|
| `dbname` | Hive/Impala database name. |
| `serv_abbr_tblname` | Target table for service abbreviation data. |
| `serv_abbr_SqoopLog` | Base path/name for the daily log file. |
| `serv_abbr_HDFSSqoopDir` | HDFS directory used as Sqoop staging area. |
| `serv_abbr_SqoopQuery` | Full Sqoop `--query` string (must end with `AND \$CONDITIONS`). |
| `ora_serverNameGBS`, `ora_portNumberGBS`, `ora_serviceNameGBS`, `ora_usernameGBS`, `ora_passwordGBS` | Oracle connection details. |
| `IMPALAD_HOST` | Hostname/IP of the Impala daemon for metadata refresh. |
| `ccList` | Comma‑separated email addresses for CC on failure alerts. |
| `GTPMailId` | Primary recipient address for failure alerts. |

These variables are sourced at the top of the script; any change must be made in the properties file, not directly in the script.

---

## 8. Suggested Improvements (TODO)

1. **Secure Credential Handling** – Replace plain‑text Oracle credentials with a vault‑based retrieval mechanism (e.g., `vault kv get` or `aws secretsmanager get-secret-value`).  
2. **Transactional Load Pattern** – Load into a temporary Hive table (`${serv_abbr_tblname}_stg`) and, on successful load, `INSERT OVERWRITE` the production table. This eliminates the need to truncate before confirming data availability and reduces the risk of an empty table after a failure.  

---