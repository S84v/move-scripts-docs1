**MNAAS_GBS_Journal_Sqoop_Load.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)  
`MNAAS_GBS_Journal_Sqoop_Load.sh` extracts the GBS journal data from the Oracle “OrgDetails” database using Sqoop, stages the raw files in a dedicated HDFS directory, and refreshes the corresponding Hive/Impala table (`$dbname.$gbs_details_tblname`). It is a core step in the daily MNAAS data‑move pipeline, providing the raw journal records that downstream aggregation, validation, and reporting scripts (e.g., `MNAAS_Daily_GBS_Avg_Rate.sh`, `MNAAS_Daily_Tolling_tbl_Aggr.sh`) consume.

---

### 2. Key Components & Responsibilities  

| Component | Responsibility |
|-----------|-----------------|
| **Property source**<br>`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties` | Loads all environment‑specific variables (DB connection, HDFS paths, table names, mail settings, etc.). |
| **Logging** (`logger -s … >> $gbs_Details_SqoopLog…`) | Writes start/finish/failure messages with timestamps to a daily log file. |
| **HDFS cleanup** (`hadoop fs -rm -r $gbs_HDFSSqoopDir/*`) | Removes any leftover files from previous runs to guarantee a clean import directory. |
| **Sqoop import** (`sqoop import … --query "${gbs_journal_SqoopQuery}" … -m 1`) | Pulls journal rows from Oracle into `$gbs_HDFSSqoopDir`. Uses a single mapper (`-m 1`). |
| **Success branch** (`if [ $? -eq 0 ]; then …`) | * Truncates the Hive table.<br>* Refreshes Impala metadata.<br>* Loads the HDFS files into Hive (`LOAD DATA INPATH`).<br>* Refreshes Impala again.<br>* Logs completion. |
| **Failure branch** (`else …`) | * Logs failure.<br>* Sends an email alert to `$GTPMailId` (CC `$ccList`). |
| **Exit status** | Implicitly returns the status of the last command executed (0 on success, non‑zero on failure). |

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Inputs** | • Property file variables (see Section 4).<br>• Oracle connection details (`$OrgDetails_*`).<br>• Sqoop query string (`$gbs_journal_SqoopQuery`). |
| **Outputs** | • HDFS directory `$gbs_HDFSSqoopDir` populated with Sqoop‑generated files.<br>• Hive table `$dbname.$gbs_details_tblname` refreshed with new rows.<br>• Daily log file `$gbs_Details_SqoopLog_YYYY-MM-DD`.<br>• Email alert on failure. |
| **Side Effects** | • Deletes *all* files under `$gbs_HDFSSqoopDir` before import (potential data loss if path is mis‑configured).<br>• Truncates the Hive table (removes previous day's data). |
| **Assumptions** | • The property file exists and contains valid values.<br>• Oracle JDBC driver is available on the node running the script.<br>• Hive/Impala services are reachable (`$IMPALAD_HOST`).<br>• The target Hive table already exists with a compatible schema.<br>• Network connectivity to Oracle and HDFS is stable. |

---

### 4. External Configuration & Environment Variables  

| Variable (populated from property file) | Role |
|------------------------------------------|------|
| `dbname` | Hive/Impala database name. |
| `gbs_details_tblname` | Target table for journal data. |
| `gbs_Details_SqoopLog` | Base path/name for the daily log file. |
| `gbs_HDFSSqoopDir` | HDFS staging directory for Sqoop output. |
| `OrgDetails_ServerName`, `OrgDetails_PortNumber`, `OrgDetails_Service` | Oracle host, port, service name. |
| `OrgDetails_Username`, `OrgDetails_Password` | Oracle credentials (plain text). |
| `gbs_journal_SqoopQuery` | Full Sqoop `--query` string (must end with `AND \$CONDITIONS`). |
| `IMPALAD_HOST` | Impala daemon host for metadata refresh. |
| `ccList` | Comma‑separated list of CC recipients for failure email. |
| `GTPMailId` | Primary recipient of failure notifications. |

*Note:* No command‑line arguments are used; all configuration is driven by the sourced property file.

---

### 5. Interaction with Other Scripts / System Components  

| Connected Component | Interaction Detail |
|---------------------|--------------------|
| **Downstream daily aggregation scripts** (e.g., `MNAAS_Daily_GBS_Avg_Rate.sh`, `MNAAS_Daily_Tolling_tbl_Aggr.sh`) | Consume the Hive/Impala table populated by this script to compute rates, aggregates, and generate reports. |
| **Monitoring / Scheduler** (Cron, Oozie, Airflow) | Typically invoked as a daily step in the MNAAS ETL workflow; downstream jobs depend on its successful completion. |
| **Alerting system** (mail server) | Sends failure notifications; the email subject includes the table name for easy correlation in monitoring dashboards. |
| **Hadoop ecosystem** (HDFS, Hive, Impala) | Uses `hadoop fs`, `hive`, and `impala-shell` CLI tools; assumes proper Kerberos tickets or password‑less SSH if security is enabled. |
| **Oracle source system** | Provides the raw journal data; any schema change in the source view/table referenced by `$gbs_journal_SqoopQuery` must be reflected in the Hive table definition. |

---

### 6. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Plain‑text Oracle credentials** in the property file. | Store credentials in a secure vault (e.g., HashiCorp Vault, Hadoop Credential Provider) and retrieve them at runtime. |
| **Single‑mapper Sqoop import (`-m 1`)** may become a performance bottleneck as data volume grows. | Evaluate data size; increase mapper count (`-m N`) after testing, and ensure the target HDFS directory is partitioned appropriately. |
| **Blind deletion of `$gbs_HDFSSqoopDir/*`** could erase unrelated files if the path is mis‑configured. | Add a safety check that the directory matches an expected pattern, or move old files to an archive location instead of deleting. |
| **No retry logic on transient failures** (network glitch, temporary Oracle outage). | Wrap the Sqoop command in a retry loop (e.g., up to 3 attempts with exponential back‑off) before aborting. |
| **Failure email may be suppressed** if the mail service is down, leaving operators unaware. | Integrate with a central alerting platform (e.g., PagerDuty, Opsgenie) in addition to email. |
| **Hive table truncation removes previous data**; if the load fails, data is lost. | Load into a staging table first, then swap/rename tables atomically after successful load. |
| **Lack of row‑count verification** – downstream jobs may run on empty tables. | Capture Sqoop import statistics (`--verbose` or `--target-dir` stats) and log the row count; abort if count is zero. |

---

### 7. Running & Debugging the Script  

| Step | Command / Action |
|------|------------------|
| **1. Verify environment** | Ensure the property file exists and is readable: `ls -l /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties` |
| **2. Dry‑run** (no data load) | Comment out the `sqoop import …` line and run the script; confirm that HDFS cleanup and logging work. |
| **3. Execute** | `bash /path/to/MNAAS_GBS_Journal_Sqoop_Load.sh` (usually invoked by a scheduler). |
| **4. Check logs** | Tail the daily log: `tail -f $gbs_Details_SqoopLog$(date +_%F)` |
| **5. Verify HDFS output** | `hadoop fs -ls $gbs_HDFSSqoopDir` – should show part‑files generated by Sqoop. |
| **6. Verify Hive/Impala tables** | `hive -e "SELECT COUNT(*) FROM $dbname.$gbs_details_tblname;"` and `impala-shell -i $IMPALAD_HOST -q "SELECT COUNT(*) FROM $dbname.$gbs_details_tblname;"` |
| **7. Debug failures** | - The script runs with `set -x` (trace mode). Review the console output or log file for the exact command that failed.<br>- If the email is not sent, test the mail command manually: `echo test | mail -v -s "test" -c $ccList $GTPMailId`. |
| **8. Re‑run after fix** | Ensure the HDFS staging directory is clean (`hadoop fs -rm -r $gbs_HDFSSqoopDir/*`) before re‑executing. |

---

### 8. Suggested Improvements (TODO)

1. **Secure Credential Management** – Replace plain‑text Oracle username/password in the property file with a call to a credential store (e.g., `kinit` + Hadoop Credential Provider) and modify the Sqoop command to use `--password-file`.  
2. **Staging‑Table Load Pattern** – Load the Sqoop output into a temporary Hive table (e.g., `${gbs_details_tblname}_stg`) and, after successful load, atomically swap it with the production table using `ALTER TABLE ... RENAME TO ...`. This eliminates data loss on partial failures and enables row‑count validation before promotion.  

--- 

*End of documentation.*