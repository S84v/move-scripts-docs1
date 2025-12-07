**Org_details_Sqoop.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)

`Org_details_Sqoop.sh` is a production‑grade Bash driver that extracts the **organization‑details** reference data from an Oracle source system via Sqoop, stages the raw files in HDFS, and refreshes the corresponding Hive/Impala table (`$dbname.$org_details_tblname`). It first clears any previous staging files, runs a single‑mapper Sqoop import using a parametrised query, truncates the target Hive table, loads the new data, and refreshes Impala metadata. On success it logs each step; on failure it logs the error, sends an alert e‑mail to the operations team, and exits with a non‑zero status.

---

### 2. Key Components & Responsibilities

| Component | Responsibility |
|-----------|-----------------|
| **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/Org_details_Sqoop.properties`** | Loads all runtime variables (DB names, connection strings, HDFS paths, email lists, etc.). |
| **`logger -s … >>$Org_Details_SqoopLog`** | Writes timestamped messages to the dedicated log file and syslog. |
| **`hadoop fs -rm -r $Org_SqoopDir/*`** | Cleans the staging directory in HDFS before a new import. |
| **`sqoop import … --query "${Org_DetailsQuery}" … --target-dir ${Org_SqoopDir} --append -m 1`** | Pulls data from Oracle into HDFS using the supplied SQL query; `-m 1` forces a single mapper (preserves ordering). |
| **`hive -S -e "truncate table $dbname.$org_details_tblname"`** | Empties the target Hive table to prepare for a fresh load. |
| **`impala-shell -i $IMPALAD_HOST -q "refresh $dbname.$org_details_tblname"`** | Forces Impala to drop cached metadata before/after the load. |
| **`hive -S -e "load data inpath '$Org_SqoopDir' into table $dbname.$org_details_tblname"`** | Moves the staged files from HDFS into the Hive table (managed table). |
| **`mail …`** | Sends a failure notification e‑mail with log reference to the configured distribution list. |
| **`if [ $? -eq 0 ]; then … else … fi`** | Branches on Sqoop exit status to perform success or failure handling. |

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | • `Org_details_Sqoop.properties` (defines all variables).<br>• Oracle connection parameters (`$OrgDetails_ServerName`, `$OrgDetails_PortNumber`, `$OrgDetails_Service`, `$OrgDetails_Username`, `$OrgDetails_Password`).<br>• HDFS staging directory `$Org_SqoopDir`.<br>• Hive/Impala database `$dbname` and table `$org_details_tblname`.<br>• SQL query `${Org_DetailsQuery}` (must contain `$CONDITIONS` placeholder for Sqoop). |
| **Outputs** | • Log file `$Org_Details_SqoopLog` (timestamped operational log).<br>• HDFS files under `$Org_SqoopDir` (temporary staging).<br>• Populated Hive table `$dbname.$org_details_tblname` (and refreshed Impala view). |
| **Side‑effects** | • Deletes *all* files under `$Org_SqoopDir` before import (potential data loss if path is mis‑configured).<br>• Truncates the Hive table, removing previous data.<br>• Sends an e‑mail on failure. |
| **Assumptions** | • The properties file exists and is readable by the executing user.<br>• Hadoop, Hive, Impala, and Sqoop CLIs are installed and on `$PATH`.<br>• Network connectivity to the Oracle host and to the Impala daemon.<br>• The executing user has HDFS write/delete rights on `$Org_SqoopDir` and Hive INSERT privileges on the target table.<br>• `mail` command is configured for outbound SMTP. |

---

### 4. Integration Points (how it connects to other scripts/components)

| Direction | Connection |
|-----------|------------|
| **Up‑stream** | Typically invoked by a nightly orchestration wrapper (e.g., a cron job or an Oozie workflow) that runs all “reference data” loads before the main mediation pipelines. |
| **Down‑stream** | Subsequent mediation or reporting jobs read from the Hive/Impala table `$dbname.$org_details_tblname` (e.g., `MNAAS_report_data_loading.sh`, `MNAAS_table_statistics_calldate_aggr.sh`). |
| **Shared resources** | • `$Org_Details_SqoopLog` may be aggregated by a central log‑collector.<br>• `$Org_SqoopDir` is a common staging area used by other Sqoop import scripts (e.g., `MNAAS_sqoop_bareport.sh`). |
| **External services** | • Oracle DB (source).<br>• HDFS (staging).<br>• Hive Metastore & Impala Daemon (target).<br>• SMTP server (mail alerts). |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Credentials in plain text** (properties file) | Store passwords in a secret manager (e.g., HashiCorp Vault) and source them at runtime; restrict file permissions to the service account. |
| **Accidental data loss** due to `hadoop fs -rm -r $Org_SqoopDir/*` or Hive `truncate` | Validate `$Org_SqoopDir` path before deletion (e.g., check it matches a whitelist). Add a backup of the Hive table (e.g., `CREATE TABLE … AS SELECT * FROM …` before truncate). |
| **Single‑mapper import (`-m 1`)** can become a bottleneck as data volume grows | Parameterise mapper count; start with `-m 4` or higher after performance testing. |
| **Failure to send alert e‑mail** (SMTP down) → silent failure | Add a fallback to write an entry to a monitoring system (e.g., write to `/var/log/alerts` or push to Prometheus). |
| **Unbounded log growth** | Rotate `$Org_Details_SqoopLog` via logrotate or timestamped log files. |
| **Sqoop query missing `$CONDITIONS` placeholder** → job hangs | Enforce validation of `${Org_DetailsQuery}` during script start; abort with clear error if missing. |

---

### 6. Typical Execution & Debugging Steps

1. **Run the script** (usually via the orchestrator, but can be manual):
   ```bash
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/Org_details_Sqoop.sh
   ```
2. **Check the log** immediately after start:
   ```bash
   tail -f $Org_Details_SqoopLog
   ```
3. **Verify HDFS staging** (if needed):
   ```bash
   hdfs dfs -ls $Org_SqoopDir
   ```
4. **Confirm Hive table row count**:
   ```bash
   hive -S -e "SELECT COUNT(*) FROM $dbname.$org_details_tblname;"
   ```
5. **If the script fails**:  
   - Look for the “sqoop process failed” entry in the log.  
   - Review the e‑mail sent to `$ccList` for the timestamped log path.  
   - Re‑run with `set -x` already enabled; you can also add `export HADOOP_ROOT_LOGGER=DEBUG,console` for more verbose Hadoop output.  
   - Check Oracle connectivity (`tnsping`, `sqlplus`) and Impala health (`impala-shell -i $IMPALAD_HOST -q "show tables;"`).  

---

### 7. External Configuration & Environment Variables

| Variable (from properties) | Description |
|----------------------------|-------------|
| `dbname` | Hive/Impala database name. |
| `org_details_tblname` | Target table name for organization details. |
| `Org_Details_SqoopLog` | Full path to the script‑specific log file. |
| `Org_SqoopDir` | HDFS directory used as Sqoop target (staging). |
| `OrgDetails_ServerName`, `OrgDetails_PortNumber`, `OrgDetails_Service` | Oracle network endpoint. |
| `OrgDetails_Username`, `OrgDetails_Password` | Oracle credentials (plain text). |
| `Org_DetailsQuery` | Sqoop import query; must contain `$CONDITIONS`. |
| `IMPALAD_HOST` | Hostname/IP of the Impala daemon for metadata refresh. |
| `ccList` | Comma‑separated e‑mail addresses for failure notifications. |
| `GTPMailId` | Primary recipient address for failure alerts. |

---

### 8. Suggested Improvements (TODO)

1. **Secure credential handling** – Move Oracle username/password out of the properties file into a vault or use Kerberos keytab authentication for Sqoop.
2. **Make mapper count configurable** – Add a property `OrgDetails_MapperCount` and replace `-m 1` with `-m $OrgDetails_MapperCount` after performance testing, to improve scalability.  

---