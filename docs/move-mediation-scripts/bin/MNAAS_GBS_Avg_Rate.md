**File:** `move-mediation-scripts\bin\MNAAS_GBS_Avg_Rate.sh`  

---

## 1. High‑Level Summary
This script extracts the **GBS average‑rate** data from an Oracle GBS database via Sqoop, stages the raw files on HDFS, and refreshes the corresponding Hive/Impala table (`$dbname.$gbs_rate_tblname`). It is part of the daily “MNAAS” data‑move pipeline that populates analytical tables used by downstream reporting and billing processes.

---

## 2. Core Logic & Responsibilities  

| Element | Type | Responsibility |
|---------|------|-----------------|
| **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_ShellScript.properties`** | source file | Provides all environment variables (DB credentials, HDFS paths, table names, mail lists, etc.). |
| **`logger -s … >> $gbs_rate_SqoopLog$(date +_%F)`** | command | Writes start/finish/failure messages to a daily log file (named per date). |
| **`hadoop fs -rm -r $rate_HDFSSqoopDir/*`** | command | Cleans the staging HDFS directory before a new Sqoop import. |
| **`sqoop import … --query "${gbs_rate_SqoopQuery}" --target-dir ${rate_HDFSSqoopDir} --append -m 1`** | command | Pulls the GBS rate rows from Oracle into HDFS. |
| **`if [ $? -eq 0 ]; then … else … fi`** | control flow | Branches on Sqoop success: on success, truncates Hive table, loads data, refreshes Impala; on failure, logs, sends email alert. |
| **`hive -S -e "truncate table …"`** | command | Empties the target Hive table before loading fresh data. |
| **`impala-shell -i $IMPALAD_HOST -q "refresh …"`** | command | Forces Impala metadata refresh after Hive changes. |
| **`hive -S -e "load data inpath … into table …"`** | command | Moves the staged HDFS files into the Hive table (partition‑aware if defined). |
| **`mail …`** | command | Sends a failure notification to the configured distribution list. |

No custom functions or classes are defined; the script is a linear procedural flow.

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Inputs** | • Oracle connection parameters (`$ora_serverNameGBS`, `$ora_portNumberGBS`, `$ora_serviceNameGBS`, `$ora_usernameGBS`, `$ora_passwordGBS`). <br>• Sqoop query string (`$gbs_rate_SqoopQuery`). <br>• HDFS staging directory (`$rate_HDFSSqoopDir`). |
| **Outputs** | • HDFS files under `$rate_HDFSSqoopDir` (raw export). <br>• Populated Hive table `$dbname.$gbs_rate_tblname`. <br>• Daily log file `$gbs_rate_SqoopLog_YYYY-MM-DD`. |
| **Side Effects** | • Deletes any pre‑existing files in the staging directory. <br>• Truncates the Hive table (data loss if run unintentionally). <br>• Sends an email on failure. |
| **Assumptions** | • Oracle JDBC driver is on the classpath. <br>• Hive/Impala services are reachable (`$IMPALAD_HOST`). <br>• The target Hive table already exists with a compatible schema. <br>• The properties file supplies all required variables; missing values cause script failure. |

---

## 4. Integration with the Wider MNAAS System  

| Connected Component | Interaction |
|---------------------|-------------|
| **Other daily load scripts** (e.g., `MNAAS_Daily_Tolling_tbl_Aggr.sh`, `MNAAS_Daily_SimInventory_tbl_Aggr.sh`) | Run in the same nightly window; they each load separate analytical tables. Coordination is typically via a cron or Oozie workflow that sequences these scripts. |
| **Oozie / Airflow DAG** (if used) | This script is a task node that depends on successful completion of upstream Oracle extraction jobs and precedes downstream reporting jobs. |
| **Reporting layer** (BI tools, dashboards) | Consumes the Hive/Impala table populated here; any delay or failure propagates to reporting latency. |
| **Alerting/Monitoring** | Email alerts feed into the operations on‑call pager; logs are ingested by a log‑aggregation system (e.g., Splunk). |
| **Properties file** | Shared across all MNAAS scripts; changes affect multiple pipelines. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Staging directory purge** (`hadoop fs -rm -r …`) accidentally removes files needed by another concurrent job. | Data loss / job failure. | Serialize jobs that share the same `$rate_HDFSSqoopDir` or use unique per‑run directories (e.g., include timestamp). |
| **Hive table truncate** – if the script crashes after truncate but before load, the table remains empty. | Missing data for downstream reports. | Wrap truncate/load in a Hive transaction (if using ACID) or load into a temporary table and swap after successful load. |
| **Hard‑coded single mapper (`-m 1`)** – may cause performance bottlenecks on large extracts. | Long run times, missed SLA. | Tune mapper count based on data volume; expose as a configurable variable. |
| **Plain‑text Oracle credentials** in properties file. | Security breach. | Store credentials in a secret manager (e.g., HashiCorp Vault) and source them at runtime. |
| **Email alert content includes raw log path** – may expose internal paths. | Information leakage. | Redact sensitive paths or use templated alerts. |

---

## 6. Running / Debugging the Script  

1. **Prerequisites** – Ensure the properties file is up‑to‑date and the user has HDFS, Hive, Impala, and Oracle access.  
2. **Dry‑run** – Add `set -n` (no‑exec) after the `set -x` line to verify variable expansion without side effects.  
3. **Execute**  
   ```bash
   cd /app/hadoop_users/MNAAS/MNAAS_Scripts/bin
   ./MNAAS_GBS_Avg_Rate.sh > /tmp/GBS_AvgRate_stdout.log 2>&1
   ```
   The `set -x` flag already prints each command; capture the output for post‑mortem.  
4. **Check logs** – Review the daily log file:  
   ```bash
   tail -f $gbs_rate_SqoopLog$(date +_%F)
   ```  
5. **Validate load** – After success, run a quick Hive/Impala query:  
   ```bash
   hive -e "SELECT COUNT(*) FROM $dbname.$gbs_rate_tblname;"
   impala-shell -i $IMPALAD_HOST -q "SELECT COUNT(*) FROM $dbname.$gbs_rate_tblname LIMIT 10;"
   ```  
6. **Failure path** – If the script emails you, inspect the log file referenced in the email; also check the exit code of the Sqoop command (`echo $?`).  

---

## 7. External Configuration & Environment Variables  

| Variable (populated in `MNAAS_ShellScript.properties`) | Purpose |
|------------------------------------------------------|---------|
| `dbname` | Hive/Impala database name. |
| `gbs_rate_tblname` | Target table name for the GBS rate data. |
| `gbs_rate_SqoopLog` | Base path for the daily log file. |
| `rate_HDFSSqoopDir` | HDFS directory where Sqoop writes raw files. |
| `ora_serverNameGBS`, `ora_portNumberGBS`, `ora_serviceNameGBS` | Oracle connection endpoint. |
| `ora_usernameGBS`, `ora_passwordGBS` | Oracle credentials. |
| `gbs_rate_SqoopQuery` | The SELECT statement (with `$CONDITIONS`) used by Sqoop. |
| `IMPALAD_HOST` | Hostname of the Impala daemon for metadata refresh. |
| `ccList`, `GTPMailId` | Email recipients for failure notifications. |

If any of these are missing or malformed, the script will abort with a non‑zero exit status.

---

## 8. Suggested Improvements (TODO)

1. **Idempotent Load** – Implement a “load‑into‑temp‑table + swap” pattern to avoid data loss if the script fails after truncation.  
2. **Parameterise Mapper Count** – Add a configurable variable (e.g., `SQOOP_MAPPERS`) and expose it in the properties file to allow performance tuning without code change.  

---