**File:** `mediation-ddls\mnaas_interconnect_traffic_details_raw.hql`  

---

## 1. High‑Level Summary
This HQL script creates an **external Hive/Impala table** named `mnaas.interconnect_traffic_details_raw`. The table stores raw inter‑connect CDR (Call Detail Record) data that is landed on HDFS under the `interconnect_traffic_details_raw` directory. It is partitioned by `partition_date` (string) and defined with a flat‑file (text) SerDe. Down‑stream mediation jobs (e.g., usage aggregation, billing, roaming reports) read from this table, so it is the entry point for raw inter‑connect traffic ingestion in the MNAAS data‑move pipeline.

---

## 2. Core Definition (No Classes/Functions)

| Element | Responsibility |
|---------|-----------------|
| **`createtab_stmt`** | Variable that holds the DDL statement; the script is typically sourced by a Hive/Impala runner that executes the content of this variable. |
| **`CREATE EXTERNAL TABLE mnaas.interconnect_traffic_details_raw`** | Registers the table metadata without moving data; the underlying files remain on HDFS and are not deleted on DROP (due to `external.table.purge='true'`). |
| **Column list** | Captures every field present in the raw CDR file (e.g., `cdrid`, `chargenumber`, `partnername`, …, `timebandindex`). Types are primarily `string`, with numeric fields (`chargenumber`, `duration`, `roundedduration`, `icbillableduration`, `timebandindex`) defined as `int` or `decimal`. |
| **`PARTITIONED BY (partition_date string)`** | Enables daily partitioning for efficient pruning; downstream jobs will add partitions as new files arrive. |
| **`ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'`** | Parses delimited text (default delimiter is `\001` unless overridden elsewhere). |
| **`STORED AS INPUTFORMAT ... TEXTINPUTFORMAT` / `OUTPUTFORMAT ... HiveIgnoreKeyTextOutputFormat`** | Indicates plain‑text files are expected. |
| **`LOCATION 'hdfs://NN-HA1/.../interconnect_traffic_details_raw'`** | Physical HDFS path where raw files are landed by the ingestion process (e.g., SFTP, FTP, or direct HDFS copy). |
| **`TBLPROPERTIES`** | Controls table behaviour: disables stats updates, enables external purge, and provides Impala catalog metadata. |

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | - Raw CDR files placed in the HDFS location defined by `LOCATION`. <br> - Files must conform to the column order and delimiter expected by the LazySimpleSerDe. |
| **Outputs** | - Hive/Impala metadata object `mnaas.interconnect_traffic_details_raw`. <br> - No data movement; the table simply points to existing files. |
| **Side Effects** | - Registers the table in the Hive metastore and Impala catalog. <br> - Creates a partition skeleton (no data until partitions are added). |
| **Assumptions** | - HDFS namenode alias `NN-HA1` resolves correctly in the execution environment. <br> - The directory exists and is writable by the ingestion process. <br> - Down‑stream jobs will add partitions via `ALTER TABLE … ADD PARTITION`. <br> - Column `cost` is stored as string in source files; downstream conversion is expected. |

---

## 4. Integration Points with Other Scripts / Components

| Connected Component | Interaction |
|---------------------|-------------|
| **Ingestion jobs** (e.g., SFTP pullers, HDFS copy scripts) | Drop raw files into the `LOCATION` path, possibly naming them with the `partition_date` suffix. |
| **Partition management scripts** (often separate `.hql` files) | Run `ALTER TABLE mnaas.interconnect_traffic_details_raw ADD IF NOT EXISTS PARTITION (partition_date='YYYYMMDD') LOCATION '…/YYYYMMDD'`. |
| **Transformation / aggregation scripts** (e.g., `mnaas_billing_traffic_usage.hql`, `mnaas_interconnect_traffic_details_*` scripts) | SELECT from this table, join with partner reference data, compute billable durations, etc. |
| **Impala/Presto query engines** | Used by analysts or downstream batch jobs to read the raw data for reporting or further ETL. |
| **Metastore / Catalog services** | The table properties (`impala.events.catalogServiceId`, `catalogVersion`) tie the definition to Impala’s catalog for fast metadata propagation. |
| **Monitoring / Alerting** | Jobs that verify the presence of today’s partition and file count will reference this table’s location and partition list. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Schema drift** – source files change column order or add new fields. | Implement a validation step in the ingestion pipeline that parses a sample file and compares column count/types to the table definition; raise an alert on mismatch. |
| **Missing or stale partitions** – downstream jobs query a date that has no partition. | Automate partition discovery after each file drop (e.g., a cron that runs `MSCK REPAIR TABLE` or explicit `ADD PARTITION`). |
| **Uncontrolled data growth** – raw files accumulate indefinitely. | Apply a retention policy (e.g., delete HDFS directories older than N days) and schedule a Hive `DROP PARTITION` with `PURGE`. |
| **Incorrect delimiter / malformed rows** – LazySimpleSerDe may silently produce nulls. | Enforce a strict delimiter (e.g., `FIELDS TERMINATED BY '\t'`) in the table DDL if known, and add a quality‑check job that counts rows with null critical fields. |
| **Cost stored as string** – downstream numeric calculations may fail. | Convert `cost` to a numeric type in downstream scripts; consider altering the table to `decimal(12,4)` after confirming source format. |
| **External table purge flag** – accidental `DROP TABLE` will delete underlying files. | Restrict DROP privileges to a limited admin role and enforce change‑management approvals. |

---

## 6. Running / Debugging the Script

1. **Execution**  
   ```bash
   hive -f mediation-ddls/mnaas_interconnect_traffic_details_raw.hql
   # or, if the script only defines a variable:
   hive -e "$(cat mediation-ddls/mnaas_interconnect_traffic_details_raw.hql | grep -v '^+' )"
   ```
   In Impala:
   ```bash
   impala-shell -f mediation-ddls/mnaas_interconnect_traffic_details_raw.hql
   ```

2. **Verification**  
   ```sql
   SHOW CREATE TABLE mnaas.interconnect_traffic_details_raw;
   DESCRIBE FORMATTED mnaas.interconnect_traffic_details_raw;
   SELECT COUNT(*) FROM mnaas.interconnect_traffic_details_raw LIMIT 1;
   ```

3. **Debugging Tips**  
   - If the table already exists, run `DROP TABLE IF EXISTS mnaas.interconnect_traffic_details_raw;` before re‑creating.  
   - Check HDFS path permissions: `hdfs dfs -ls /user/hive/warehouse/mnaas.db/interconnect_traffic_details_raw`.  
   - Verify that the Hive metastore service is reachable (`netstat -an | grep 9083`).  
   - Use `MSCK REPAIR TABLE mnaas.interconnect_traffic_details_raw;` to load any new partitions after files are landed.  

---

## 7. External Config / Environment Dependencies

| Item | Usage |
|------|-------|
| **HDFS namenode alias `NN-HA1`** | Resolved via Hadoop core-site.xml; must point to the active NameNode. |
| **Hive Metastore URI** (`hive.metastore.uris`) | Required for the `CREATE EXTERNAL TABLE` command to register metadata. |
| **Impala Catalog Service ID / Version** | Populated automatically; no manual configuration needed but must match the running Impala cluster. |
| **Kerberos / LDAP credentials** (if enabled) | Needed for the user executing the script to have `CREATE` privileges on the `mnaas` database. |
| **Configuration for delimiter** (optional) | If the source files use a delimiter other than the default `\001`, a `ROW FORMAT DELIMITED FIELDS TERMINATED BY` clause would be added; currently absent, so default applies. |

---

## 8. Suggested Improvements (TODO)

1. **Add explicit delimiter and column comments** – Improves readability and guards against accidental delimiter changes in source files.  
   ```sql
   ROW FORMAT DELIMITED
     FIELDS TERMINATED BY '\t'
     ESCAPED BY '\\'
   ```

2. **Convert `cost` to a numeric type** – Prevents downstream casting errors and enables proper aggregation.  
   ```sql
   ALTER TABLE mnaas.interconnect_traffic_details_raw CHANGE cost cost DECIMAL(12,4);
   ```

   (Only after confirming all source files contain parsable numeric values.)

---