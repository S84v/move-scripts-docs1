**File:** `mediation-ddls\mnaas_vaz_dm_applet_provisioning.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates an **external Parquet table** named `vaz_dm_applet_provisioning` in the `mnaas` database. The table stores provisioning‑related records (applet‑level device‑management data) that are written by upstream ingestion jobs (e.g., Spark, Flink, or Sqoop) into an HDFS directory. The table is partitioned by `partitiondate` to enable efficient daily pruning. Because it is *external*, Hive only registers the metadata; the underlying files are managed outside Hive (typically by the provisioning pipeline).

---

## 2. Core Objects Defined

| Object | Type | Responsibility / Description |
|--------|------|------------------------------|
| `vaz_dm_applet_provisioning` | External Hive table | Holds raw provisioning records as strings; each row corresponds to a single applet provisioning event. |
| Columns (`filename`, `id`, `creationdate`, …, `record_insert_time`) | `STRING` | Capture raw fields as they appear in the source feed (e.g., CSV/JSON → Parquet conversion). |
| `partitiondate` | `STRING` (partition column) | Daily partition key used by downstream jobs for incremental processing. |
| Table properties (e.g., `parquet.compression='SNAPPY'`) | Hive metadata | Define storage format, compression, and Impala catalog integration. |
| `LOCATION` | HDFS path | Physical storage location: `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/vaz_dm_applet_provisioning`. |

*No procedural code (functions, procedures) is present; the file only contains a DDL statement.*

---

## 3. Inputs, Outputs & Side Effects

| Aspect | Details |
|--------|---------|
| **Input data** | Files written by upstream provisioning pipelines (most likely Parquet files generated from source systems such as device‑management platforms, OSS/BSS, or partner APIs). |
| **Output** | Hive metadata registration; the external table becomes queryable by Hive/Impala, Presto, SparkSQL, etc. No data is moved or transformed by this script itself. |
| **Side effects** | - Creates a directory entry in the Hive metastore.<br>- May overwrite an existing table definition if the script is re‑executed (Hive will drop the old definition and re‑create it). |
| **Assumptions** | - HDFS namenode `NN-HA1` is reachable and the path exists (or Hive has permission to create it).<br>- The underlying files conform to the Parquet schema defined (all columns stored as strings).<br>- Partitioning scheme (`partitiondate`) will be populated by the ingestion job; the script does **not** add partitions automatically. |
| **External services** | - Hive Metastore (for DDL registration).<br>- HDFS (storage).<br>- Impala catalog (via table properties). |

---

## 4. Integration Points & Data Flow

1. **Upstream ingestion job** (e.g., Spark batch, Flink streaming, or a custom Java/Scala ETL) writes Parquet files to the HDFS location defined in `LOCATION`.  
   - The job must include the `partitiondate` directory structure (`.../partitiondate=YYYY-MM-DD/`) because Hive will only see data under those partition folders.  
2. **Partition management** – After files are landed, a separate script (often `MSCK REPAIR TABLE` or an `ALTER TABLE ADD PARTITION` command) is executed to make Hive aware of the new partitions. This script is **not** part of the current DDL but is required for the table to be queryable.  
3. **Downstream consumers** – Reporting, billing, or analytics jobs query `mnaas.vaz_dm_applet_provisioning` (or its views) to extract provisioning events for charging, KPI calculation, or audit trails.  
4. **Related DDLs** – The other `mnaas_*` DDL files in the repository define traffic‑detail tables; they likely share the same `mnaas` database and may be joined with this provisioning table for enriched billing or network‑performance analysis.  

*Typical orchestration tool (e.g., Airflow, Oozie, or custom scheduler) will run the DDL once during environment bootstrap, then schedule the ingestion and partition‑repair jobs on a daily cadence.*

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale metadata** – Table created but no partitions added → queries return empty result sets. | Billing/analytics pipelines miss data. | Automate `MSCK REPAIR TABLE` or `ALTER TABLE ... ADD PARTITION` after each ingestion batch; add health‑check that verifies row count > 0 for the latest partition. |
| **Schema drift** – Source system adds new columns but the external table still defines only strings. | Data loss or query failures. | Periodically run a schema‑validation job that compares source schema (e.g., from Avro/Proto) with Hive DDL; version‑control DDL changes. |
| **Permission issues** – Hive or the ingestion job lacks write access to the HDFS location. | Job failures, incomplete data. | Ensure the service account used by Hive and the ETL has `rw` on the target directory; audit ACLs after any cluster upgrade. |
| **Partition skew** – All data lands in a single partition (e.g., missing `partitiondate`). | Performance degradation, long query times. | Enforce partition column population in the ETL; add a pre‑load validation step that aborts if `partitiondate` is null. |
| **External table deletion** – Accidental `DROP TABLE` removes only metadata, but underlying files remain, leading to orphaned data. | Storage bloat, confusion. | Enable Hive metastore backup; use `DROP TABLE IF EXISTS` guarded by a confirmation flag in deployment scripts. |

---

## 6. Running / Debugging the Script

| Step | Command | Purpose |
|------|---------|---------|
| **1. Connect to Hive/Impala** | `hive -S -e "SHOW DATABASES;"` or `impala-shell -i <impala-host>` | Verify connectivity. |
| **2. Execute DDL** | `hive -f mediation-ddls/mnaas_vaz_dm_applet_provisioning.hql` | Registers the external table. |
| **3. Verify creation** | `hive -e "DESCRIBE FORMATTED mnaas.vaz_dm_applet_provisioning;"` | Confirms columns, location, and properties. |
| **4. Check HDFS path** | `hdfs dfs -ls /user/hive/warehouse/mnaas.db/vaz_dm_applet_provisioning` | Ensure the directory exists and is accessible. |
| **5. Add a test partition (optional)** | `ALTER TABLE mnaas.vaz_dm_applet_provisioning ADD IF NOT EXISTS PARTITION (partitiondate='2025-12-04');` | Allows immediate query testing. |
| **6. Query sample data** | `SELECT * FROM mnaas.vaz_dm_applet_provisioning LIMIT 10;` | Verify that data appears after ingestion. |
| **Debug tip** | If the query returns *no rows* but files exist, run `MSCK REPAIR TABLE mnaas.vaz_dm_applet_provisioning;` and re‑query. | |

*When debugging, also check Hive metastore logs (`/var/log/hive/hive-metastore.log`) for permission or schema errors.*

---

## 7. Configuration & External Dependencies

| Item | How it is used |
|------|----------------|
| **HDFS namenode address** (`NN-HA1`) | Embedded in the `LOCATION` clause; must resolve via DNS or `/etc/hosts`. |
| **Hive Metastore URI** (`hive.metastore.uris`) | Determines where the DDL is registered; typically set in `hive-site.xml`. |
| **Impala catalog service ID & version** (`impala.events.catalogServiceId`, `impala.events.catalogVersion`) | Auto‑generated by Impala; not required for Hive operation but needed for Impala to see the table instantly. |
| **Parquet compression** (`parquet.compression='SNAPPY'`) | Controls file size and read performance; downstream jobs must be able to read SNAPPY‑compressed Parquet. |
| **Environment variables** (e.g., `HIVE_CONF_DIR`, `HADOOP_CONF_DIR`) | Must point to the correct configuration directories when running the script from a CI/CD pipeline or scheduler. |
| **External ingestion scripts** (not in this repo) | Must write files matching the column order and types (all strings) into the partitioned HDFS path. |

---

## 8. Suggested TODO / Improvements

1. **Strongly‑typed columns** – Replace generic `STRING` types with appropriate Hive types (`TIMESTAMP` for dates, `BOOLEAN` for flags, `BIGINT` for IDs). This improves query performance and data validation.
2. **Automated partition discovery** – Add a post‑creation step in the deployment pipeline:  
   ```sql
   MSCK REPAIR TABLE mnaas.vaz_dm_applet_provisioning;
   ```  
   or use `ALTER TABLE ... ADD PARTITION` based on a manifest file generated by the ingestion job, to guarantee that new partitions are instantly visible.

*Both changes reduce runtime errors and improve downstream analytics.*