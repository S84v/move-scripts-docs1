**File:** `mediation-ddls\move_sim_inventory_status.hql`  

---

## 1. High‑Level Summary
This HiveQL script defines an **external table** named `move_sim_inventory_status` that materialises SIM‑inventory status records stored as semi‑colon‑delimited text files in HDFS (`hdfs://NN-HA1/user/hive/warehouse/mnaas.db/move_sim_inventory_status`). The table captures up to ten IMSI values, multiple MSISDNs, SIM identifiers, lifecycle dates, provisioning details, and billing‑related attributes. It is partitioned by a string column `partition_date` to enable incremental processing. In production the table is used by downstream “move” jobs (e.g., `move_sim_inventory_count`, `move_master_consolidated_1`) for reporting, reconciliation, and billing feed generation.

---

## 2. Core Artefacts & Responsibilities  

| Artefact | Type | Responsibility |
|----------|------|----------------|
| `move_sim_inventory_status` | Hive **external** table | Provides a queryable view over raw SIM‑inventory files; isolates data lifecycle from Hive metastore (no data copy on DROP). |
| Columns (`primary_imsi` … `sim_type`) | Table schema | Capture all attributes required for inventory tracking, provisioning status, and billing integration. |
| `partition_date` | Partition column | Enables day‑level incremental loads and efficient pruning for downstream jobs. |
| SerDe & Storage settings | Table definition | Parse semi‑colon delimited text (`LazySimpleSerDe`), store as plain text (`TextInputFormat` / `HiveIgnoreKeyTextOutputFormat`). |
| Table properties (`external.table.purge`, `DO_NOT_UPDATE_STATS`, …) | Metastore metadata | Control statistics collection, automatic data purge on DROP, and Impala catalog sync. |

*No procedural code (functions, classes) exists in this file; its sole purpose is schema definition.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - Raw data files placed under the HDFS location defined in `LOCATION`. <br> - Files must be UTF‑8 text, rows delimited by `\n`, fields delimited by `;`. <br> - Each file must contain a `partition_date` value (as a string) that matches the table’s partition. |
| **Outputs** | - Hive metastore entry for `move_sim_inventory_status`. <br> - Queryable dataset for downstream jobs (e.g., counts, joins, billing extracts). |
| **Side‑Effects** | - Registers an external table; no data movement occurs. <br> - Because `external.table.purge='true'`, dropping the table will **delete** the underlying HDFS files. |
| **Assumptions** | - HDFS path is reachable by Hive/Impala services (namenode alias `NN-HA1`). <br> - Proper ACLs/permissions exist for read/write on the directory. <br> - Partition values are supplied consistently (e.g., `YYYYMMDD`). <br> - No schema evolution is required after production deployment (columns are fixed). |

---

## 4. Integration Points  

| Connected Component | Interaction |
|---------------------|-------------|
| **`move_sim_inventory_count.hql`** | Reads this table to aggregate inventory counts per business unit, region, or status. |
| **`move_master_consolidated_1.hql`** | May join on `sim` or `primary_imsi` to enrich master‑consolidated view. |
| **ETL ingestion pipelines** (e.g., nightly SFTP drop, Kafka → HDFS) | Populate the HDFS directory with new files; they must respect the delimiter and schema. |
| **Impala/Presto query layer** | Executes analytical queries against the table; relies on the `impala.events.*` properties for catalog sync. |
| **Scheduler (Oozie / Airflow)** | Triggers partition creation (`ALTER TABLE … ADD PARTITION`) before data lands, or runs `MSCK REPAIR TABLE` to discover new partitions. |
| **Monitoring / Alerting** | Checks table existence, partition freshness, and HDFS storage usage. |

*Because the table is external, any process that removes or archives files from the HDFS location will immediately affect downstream consumers.*

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Schema drift** – upstream data files add/remove columns. | Queries fail; downstream jobs produce incorrect results. | Enforce schema validation in the ingestion pipeline; version‑control the DDL and run a nightly `DESCRIBE FORMATTED` diff check. |
| **Stale or missing partitions** – new `partition_date` not added. | Data for a day is invisible to consumers. | Automate partition addition (e.g., `ALTER TABLE … ADD IF NOT EXISTS PARTITION`) as part of the load job; schedule `MSCK REPAIR TABLE` after each load. |
| **Accidental table drop** – `external.table.purge=true` deletes raw files. | Permanent loss of raw inventory data. | Restrict DROP privileges; implement a pre‑drop approval workflow; keep a backup of the HDFS directory (snapshot or replication). |
| **Permission mis‑configuration** – Hive/Impala cannot read/write the HDFS path. | Job failures, data not visible. | Audit HDFS ACLs; use a dedicated service account; test access with `hdfs dfs -ls` before deployment. |
| **Performance degradation** – large number of small files (one per SIM). | Slow query planning, high metadata overhead. | Consolidate files per partition (e.g., using `Hive ACID` or `ORC` format) or enable `hive.optimize.skewjoin` if needed. |

---

## 6. Typical Execution & Debugging Workflow  

1. **Run the DDL**  
   ```bash
   hive -f mediation-ddls/move_sim_inventory_status.hql
   ```
   *If the table already exists, the command will fail – use `CREATE EXTERNAL TABLE IF NOT EXISTS` in a wrapper script.*

2. **Validate creation**  
   ```sql
   SHOW CREATE TABLE move_sim_inventory_status;
   DESCRIBE FORMATTED move_sim_inventory_status;
   SHOW PARTITIONS move_sim_inventory_status;
   ```

3. **Load a test file** (place a small semi‑colon delimited file in the HDFS location, e.g., `20231201/part-00000`) and add the partition:  
   ```sql
   ALTER TABLE move_sim_inventory_status ADD IF NOT EXISTS PARTITION (partition_date='20231201');
   ```

4. **Query a sample**  
   ```sql
   SELECT primary_imsi, sim, prod_status, activation_date
   FROM move_sim_inventory_status
   WHERE partition_date='20231201' LIMIT 10;
   ```

5. **Debugging tips**  
   - Check file encoding & delimiters (`hdfs dfs -cat … | head`).  
   - Verify that the `partition_date` column matches the directory name.  
   - Use `EXPLAIN` on a query to ensure partition pruning.  
   - Look at Hive/Impala logs for SerDe parsing errors (`LazySimpleSerDe` mismatches).  

---

## 7. External Configuration & Environment Dependencies  

| Item | Usage |
|------|-------|
| `hdfs://NN-HA1/...` | Hard‑coded HDFS namenode alias; must resolve via the cluster’s HA configuration (`hdfs-site.xml`). |
| Hive Metastore URI (`hive.metastore.uris`) | Required for table registration; typically set in `hive-site.xml`. |
| Impala catalog service ID & version (`impala.events.*`) | Auto‑populated by Impala; no manual edit needed but the properties must be present for Impala‑Hive sync. |
| Environment variables (e.g., `HADOOP_CONF_DIR`, `HIVE_CONF_DIR`) | Needed by the Hive CLI or Beeline when executing the script. |
| Scheduler variables (e.g., `${run_date}`) | Not present in this DDL, but downstream jobs often pass the partition value via a variable. |

If any of these values change (e.g., a new namenode address), the script must be updated accordingly.

---

## 8. Suggested Improvements (TODO)

1. **Add Table Comment & Column Descriptions** – Improves discoverability for downstream developers and data catalog tools.  
   ```sql
   COMMENT 'External table storing raw SIM inventory status records for billing and reporting.'
   ALTER TABLE move_sim_inventory_status CHANGE COLUMN primary_imsi primary_imsi STRING COMMENT 'Primary IMSI (International Mobile Subscriber Identity)';
   ```

2. **Introduce a Staging Format (ORC/Parquet) & Compaction** – Convert the raw text files into a columnar format after ingestion to reduce metadata overhead and improve query performance. This can be done in a separate “compact” job that writes to a managed table and swaps the external location.  

---