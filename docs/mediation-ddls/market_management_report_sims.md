**File:** `mediation-ddls\market_management_report_sims.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates an **external Parquet table** named `market_management_report_sims` in the `mnaas` database. The table stores SIM‑level usage and provisioning data that feeds the *Market Management Report* downstream analytics (Spark, Impala, reporting dashboards). It is partitioned by `partition_year_month` and points to a pre‑defined HDFS location (`hdfs://NN-HA1/user/hive/warehouse/mnaas.db/market_management_report_sims`). Because the table is external and has `external.table.purge='TRUE'`, dropping it will also delete the underlying files.

---

## 2. Core Object(s) Defined

| Object | Type | Responsibility |
|--------|------|-----------------|
| `market_management_report_sims` | Hive **EXTERNAL TABLE** | Holds raw SIM‑level event records (usage, activation, status, cost) for market‑management reporting. |
| `partition_year_month` | Partition column (string) | Enables efficient pruning by year‑month when querying. |
| SerDe / Input‑/Output‑Format | `ParquetHiveSerDe`, `MapredParquetInputFormat`, `MapredParquetOutputFormat` | Guarantees column‑type fidelity and compression for large‑scale analytics. |
| Table properties (e.g., `external.table.purge`, `spark.sql.*`, `impala.*`) | Metadata | Controls lifecycle, statistics handling, and compatibility with Spark/Impala engines. |

*No procedural code (functions, procedures) is present in this file.*

---

## 3. Schema Overview  

| Column | Hive Type | Comment / Meaning |
|--------|-----------|-------------------|
| `tcl_secs_id` | int | – |
| `orgname` | string | – |
| `businessunit` | string | – |
| `calldate` | string | **date that event occurred (CET)** |
| `supplier_name` | string | – |
| `supplier_prefix` | string | – |
| `country` | string | **name country MCC** |
| `tadig` | string | **Transferred Account Data Interchange Group (network)** |
| `mcc` | string | **Mobile Country Code (3 digits)** |
| `mnc` | string | **Mobile Network Code (1‑3 digits)** |
| `globaltitle` | string | **Address of mobile network element used when roaming** |
| `operator` | string | **Name operator (MNC)** |
| `commercial_offer` | string | – |
| `usage_data_mb` | double | – |
| `usage_sms_mo` | bigint | – |
| `usage_sms_mt` | bigint | – |
| `usage_voice_mo_min` | double | – |
| `usage_voice_mt_min` | double | – |
| `sponsor_currency` | string | – |
| `wscost1_data_mb` | decimal(24,4) | – |
| `wscost1_sms_mo` | decimal(24,4) | – |
| `wscost1_sms_mt` | decimal(24,4) | – |
| `wscost1_voice_mo_min` | decimal(24,4) | – |
| `wscost1_voice_mt_min` | decimal(24,4) | – |
| `profile_with_version` | string | – |
| `prov_prod_type` | string | – |
| `rate_plan_name` | string | – |
| `rate_plan_code` | int | – |
| `zone_type` | string | – |
| `whitelisted_zones` | string | – |
| `blacklisted_zones` | string | – |
| `accountnumber` | string | **account number (internal)** |
| `msisdn` | string | – |
| `sim` | string | – |
| `sim_batch` | string | – |
| `activation_date` | timestamp | – |
| `deactivation_date` | timestamp | – |
| `last_prod_status_chng_dt` | timestamp | – |
| `prod_status` | string | – |
| `last_traffic_date` | timestamp | – |
| `productid` | string | **Unique identifier for product of the user** |
| `sim_type` | string | – |
| `sim_vendor` | string | – |
| `euiccid` | string | **EUICCID (empty for traditional SIM, EID for eSIM)** |
| `imsi` | string | – |
| `mappedimsi` | string | **Mapped IMSI** |
| **Partition** `partition_year_month` | string | Year‑Month string used for partitioning (e.g., `202312`). |

---

## 4. Storage, SerDe, and Table Properties  

| Setting | Value | Purpose |
|---------|-------|---------|
| **ROW FORMAT SERDE** | `org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe` | Parquet columnar storage. |
| **INPUTFORMAT** | `org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat` | Reads Parquet files. |
| **OUTPUTFORMAT** | `org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat` | Writes Parquet files. |
| **LOCATION** | `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/market_management_report_sims` | Physical HDFS path. |
| **external.table.purge** | `TRUE` | Drop → delete underlying files (hard delete). |
| **DO_NOT_UPDATE_STATS** | `true` | Prevents automatic stats collection on DDL. |
| **spark.sql.* / impala.* properties** | Various version / catalog IDs | Enables Spark & Impala to discover the table without re‑reading metadata. |
| **transient_lastDdlTime** | `1763046394` | Hive internal timestamp. |

---

## 5. Integration Points (How This Table Connects to the Rest of the System)

| Connected Component | Interaction |
|---------------------|-------------|
| **Ingestion pipelines** (e.g., `market_management_report_sims_load.hql` or Spark jobs) | Write Parquet files into the HDFS location; may use `INSERT OVERWRITE DIRECTORY` or `spark.write.parquet`. |
| **Dimension tables** (`dim_date.hql`, `gbs_curr_avgconv_rate.hql`) | Joined on `calldate` / `partition_year_month` for time‑based aggregations. |
| **Reporting layer** (Impala, Tableau, PowerBI) | Queries the table directly for market‑management dashboards. |
| **Costing / Billing jobs** | Consume `wscost1_*` columns to calculate revenue/expense. |
| **Data quality / validation scripts** | Run `SELECT COUNT(*)`, `DESCRIBE FORMATTED`, or custom Spark checks against this table. |
| **Archival / purge processes** | May drop the table for a month and rely on `external.table.purge` to clean up old partitions. |

---

## 6. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – upstream jobs add/remove columns without updating DDL. | Query failures, data loss. | Version‑control DDL; run automated schema‑diff checks before deployment. |
| **Partition mis‑alignment** – files placed in wrong `partition_year_month` folder. | Incorrect query results, full table scans. | Enforce partition path in ingestion jobs; add a post‑load validation step that checks folder names vs. column values. |
| **`external.table.purge=TRUE`** – accidental `DROP TABLE` removes raw data. | Irrecoverable data loss. | Require a change‑approval workflow; protect the table with Hive ACLs; keep a nightly backup of the HDFS directory. |
| **Permission / HDFS availability** – `NN-HA1` namenode down or ACLs missing. | Ingestion or query failures. | Monitor HDFS health; embed retry logic in ingestion jobs; document required Hive/FS permissions. |
| **Statistical metadata stale** – `DO_NOT_UPDATE_STATS=true` may cause poor query plans. | Slow queries. | Schedule periodic `ANALYZE TABLE … COMPUTE STATISTICS` jobs or set `DO_NOT_UPDATE_STATS` to `false` after initial load. |
| **Data type mismatch** – decimal precision vs. source system. | Truncation or overflow. | Validate source data ranges; adjust column definitions if needed. |

---

## 7. Running / Debugging the Script  

1. **Execute**  
   ```bash
   hive -f mediation-ddls/market_management_report_sims.hql
   ```
   *If the table already exists, prepend `DROP TABLE IF EXISTS mnaas.market_management_report_sims;` or add `IF NOT EXISTS` to the `CREATE` statement.*

2. **Validate Creation**  
   ```sql
   SHOW CREATE TABLE mnaas.market_management_report_sims;
   DESCRIBE FORMATTED mnaas.market_management_report_sims;
   ```

3. **Check Data Presence** (after ingestion)  
   ```sql
   SELECT COUNT(*) FROM mnaas.market_management_report_sims LIMIT 10;
   SELECT partition_year_month, COUNT(*) FROM mnaas.market_management_report_sims GROUP BY partition_year_month;
   ```

4. **Debug Common Issues**  
   * **Missing HDFS path** – Verify `hdfs dfs -ls /user/hive/warehouse/mnaas.db/market_management_report_sims`.  
   * **Permission errors** – Ensure the Hive user has `READ/WRITE` on the directory.  
   * **Parquet schema mismatch** – Run `spark.read.parquet(...).printSchema()` on a sample file and compare to Hive DDL.

---

## 8. External Configuration / Environment Dependencies  

| Item | How It Is Used |
|------|----------------|
| **HDFS Namenode** – `NN-HA1` | Provides the physical storage location for the external table. |
| **Hive Metastore** | Persists the table definition; must be reachable from the Hive client. |
| **Database `mnaas`** | Logical namespace; other scripts reference this DB. |
| **Environment variables** (commonly used in the broader pipeline) – e.g., `HIVE_CONF_DIR`, `HADOOP_USER_NAME` | Not referenced directly in this file but required for Hive CLI execution. |
| **Spark/Impala version IDs** (e.g., `spark.sql.create.version`) | Embedded as table properties to aid engine compatibility; they are static strings, not dynamically resolved. |

If the deployment environment uses a templating system (e.g., Jinja, Liquibase), the hard‑coded HDFS URI may be replaced by a variable such as `${hdfs_root}` – verify the surrounding CI/CD pipeline.

---

## 9. Suggested Improvements (TODO)

1. **Make the DDL idempotent** – add `IF NOT EXISTS` to the `CREATE EXTERNAL TABLE` statement and optionally a preceding `DROP TABLE IF EXISTS` guarded by a safety flag.  
2. **Automate partition discovery** – create a small Hive/Beeline script (or Spark job) that scans the HDFS directory, extracts distinct `partition_year_month` values, and runs `ALTER TABLE … ADD IF NOT EXISTS PARTITION (partition_year_month='YYYYMM') LOCATION '…/partition_year_month=YYYYMM'`. This prevents missing partitions after bulk loads.

---