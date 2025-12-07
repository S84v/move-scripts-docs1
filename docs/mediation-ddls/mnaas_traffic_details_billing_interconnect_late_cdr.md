**File:** `mediation-ddls\mnaas_traffic_details_billing_interconnect_late_cdr.hql`  

---

## 1. High‑Level Summary
This script creates a Hive **external** table named `traffic_details_billing_interconnect_late_cdr` in the `mnaas` database. The table stores “late” Call Detail Records (CDRs) that belong to inter‑connect billing flows. Data are stored as plain‑text files on HDFS under a fixed warehouse path and are partitioned by `bill_month`. Down‑stream analytics, billing, and reconciliation jobs query this table to incorporate delayed CDRs into monthly inter‑connect invoices.

---

## 2. Core Definition (the only “class”/object in this file)

| Object | Responsibility |
|--------|-----------------|
| **External Table `mnaas.traffic_details_billing_interconnect_late_cdr`** | Provides a schema‑enforced view over raw CDR files that arrive late (outside the normal ingestion window). The table is **external**, so Hive only manages metadata; the underlying files remain under direct HDFS control. |
| **Columns** | Capture all fields required for billing: identifiers (`cdrid`, `chargenumber`), partner/customer info (`partnername`, `customernumber`, `tadigcode`), network context (`mcc`, `mnc`, `country`, `operator`), call details (`calltype`, `traffictype`, `calldatetime`, `duration`, `roundedduration`, `icbillableduration`, `cost`), routing (`routingprefixin/out`, `terminatingnumber`, `smsc_msc`, `roaming`, `partnertype`), file provenance (`filename`, `record_insertime`, `timebandindex`, `file_date`). |
| **Partition** | `bill_month` (string) – enables efficient month‑wise pruning for billing runs. |
| **Storage Format** | Text files (`TextInputFormat`) with Hive’s `LazySimpleSerDe`. No compression or ORC/Parquet is used. |
| **Location** | `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/traffic_details_billing_interconnect_late_cdr` – the physical directory that upstream processes drop late‑CDR files into. |
| **Table Properties** | `OBJCAPABILITIES='EXTREAD,EXTWRITE'` (allows external read/write) and `transient_lastDdlTime` (auto‑generated timestamp). |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Aspect | Details |
|--------|---------|
| **Inputs** | - Raw late‑CDR text files placed in the HDFS location above (usually by an upstream ETL or SFTP ingest job). <br> - Partition value (`bill_month`) supplied as part of the file path or via `ALTER TABLE … ADD PARTITION`. |
| **Outputs** | - Hive metadata entry for the external table (visible to any Hive/Presto/Trino query engine). <br> - No data movement; the table simply references existing files. |
| **Side‑Effects** | - Registers the table in the Hive Metastore. <br> - May create the HDFS directory if it does not exist (depending on Hive configuration). |
| **Assumptions** | - Hive Metastore is reachable and the `mnaas` database already exists. <br> - HDFS namenode alias `NN-HA1` resolves in the cluster network. <br> - Files placed in the location conform to the column order and delimiter expectations of `LazySimpleSerDe` (default tab/ctrl‑A). <br> - `bill_month` partition values are supplied consistently (e.g., `2024-09`). |

---

## 4. Integration with Other Scripts / Components

| Connected Component | Relationship |
|---------------------|--------------|
| **`mnaas_traffic_details_billing.hql`** | Likely creates the *primary* billing table for on‑time CDRs. The late‑CDR table is queried together with it during month‑end reconciliation. |
| **Ingestion jobs (e.g., SFTP pullers, Spark/MapReduce loaders)** | Drop late‑CDR files into the HDFS path and issue `ALTER TABLE … ADD PARTITION (bill_month='YYYY-MM')` so Hive can see them. |
| **Aggregation scripts (`mnaas_traffic_details_aggr_daily.hql`, `mnaas_traffic_Details_raw_daily.hql`)** | May read from this table to produce daily/weekly aggregates that include late records. |
| **Billing engine (custom Java/Python service, or Spark job)** | Executes SELECTs against `traffic_details_billing_interconnect_late_cdr` to adjust inter‑connect invoices. |
| **Monitoring / Alerting** | External scripts may query `SHOW PARTITIONS` to verify that each expected `bill_month` partition exists after the late‑CDR load window. |
| **Configuration files** | Typically a `hive-site.xml` (for metastore URL, default FS) and possibly an environment variable `HDFS_NAMENODE=NN-HA1`. The script itself does not reference them directly but relies on the Hive client’s configuration. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing HDFS directory or permission error** | Table creation succeeds but queries return “File not found”. | Pre‑deployment check: `hdfs dfs -test -d <path>`; ensure the service account has `rw` on the directory. |
| **Schema drift (incoming files with extra/missing columns)** | Queries fail with “cannot deserialize” errors. | Enforce schema validation in the upstream loader (e.g., Spark schema enforcement) and add a Hive `ROW FORMAT SERDE` property `serialization.null.format=''` to tolerate missing fields. |
| **Late‑CDR files placed in wrong `bill_month` partition** | Data appears in the wrong billing cycle, causing revenue leakage. | Require the loader to add the partition explicitly; add a post‑load verification job that checks file timestamps vs. partition value. |
| **Uncontrolled growth of raw text files** | Storage pressure, slower scans. | Periodically compact files (e.g., using Hive `INSERT OVERWRITE` into ORC/Parquet) or enable HDFS archiving. |
| **Concurrent `ALTER TABLE … ADD PARTITION` causing metadata race** | Duplicate partitions or missing data. | Serialize partition‑add operations via a lock file or use Hive’s `IF NOT EXISTS` clause. |
| **Incorrect delimiter handling** | Mis‑aligned columns, corrupt data. | Document the expected delimiter (default `\001`) and enforce it in the loader; optionally set `FIELD TERMINATOR` explicitly in the DDL. |

---

## 6. Running / Debugging the Script

1. **Execute the DDL**  
   ```bash
   hive -f mediation-ddls/mnaas_traffic_details_billing_interconnect_late_cdr.hql
   ```
   *If Hive is configured with Beeline:*  
   ```bash
   beeline -u jdbc:hive2://<hs2-host>:10000 -f mediation-ddls/mnaas_traffic_details_billing_interconnect_late_cdr.hql
   ```

2. **Verify table creation**  
   ```sql
   SHOW CREATE TABLE mnaas.traffic_details_billing_interconnect_late_cdr;
   DESCRIBE FORMATTED mnaas.traffic_details_billing_interconnect_late_cdr;
   ```

3. **Check HDFS location**  
   ```bash
   hdfs dfs -ls /user/hive/warehouse/mnaas.db/traffic_details_billing_interconnect_late_cdr
   ```

4. **Add a test partition (if not done by loader)**  
   ```sql
   ALTER TABLE mnaas.traffic_details_billing_interconnect_late_cdr
     ADD IF NOT EXISTS PARTITION (bill_month='2024-09')
     LOCATION 'hdfs://NN-HA1/user/hive/warehouse/mnaas.db/traffic_details_billing_interconnect_late_cdr/bill_month=2024-09';
   ```

5. **Sample query**  
   ```sql
   SELECT cdrid, cost, bill_month
   FROM mnaas.traffic_details_billing_interconnect_late_cdr
   WHERE bill_month='2024-09' LIMIT 10;
   ```

6. **Debugging tips**  
   - If the query returns zero rows, verify that files exist under the partition directory and that they are readable by the Hive service account.  
   - Use `hdfs dfs -cat <file>` to inspect a raw line and confirm column order/delimiter.  
   - Check Hive Metastore logs for “Table not found” or “Partition not found” errors.  

---

## 7. External Config / Environment Variables

| Config / Variable | Usage |
|-------------------|-------|
| `hive.metastore.uris` (in `hive-site.xml`) | Determines where the Metastore service lives; required for table creation. |
| `fs.defaultFS` (in `core-site.xml`) | Provides the default HDFS namenode; the script references `hdfs://NN-HA1/...` directly, so the alias must resolve in the cluster DNS. |
| Service account permissions (e.g., `hdfs` user) | Must have `WRITE` on the external table location. |
| Optional: `HIVE_CONF_DIR` environment variable | Points to the directory containing the above XML files for the Hive CLI/Beeline. |

The script itself does **not** read any custom variables; all paths are hard‑coded.

---

## 8. Suggested Improvements (TODO)

1. **Make the DDL idempotent** – prepend `CREATE EXTERNAL TABLE IF NOT EXISTS` and use `ADD IF NOT EXISTS` for partitions to avoid failures on re‑run.  
2. **Parameterize the HDFS location** – replace the hard‑coded `NN-HA1` with a Hive variable (e.g., `${hdfs_namenode}`) so the same script can be used across environments (dev/test/prod) without modification.  

---