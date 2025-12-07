**File:** `mediation-ddls\mnaas_ tolling_raw_daily.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates the external table **`tolling_raw_daily`** that serves as the canonical landing zone for daily tolling data received from MVNO partners. The table is partitioned by `partition_date` (string) and stored as Parquet files under the HDFS path `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/tolling_raw_daily`. Down‑stream billing, mediation, and reporting jobs query this table (or views built on it) to enrich raw tolling records with product, customer, and commercial‑offer metadata.

---

## 2. Core Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `tolling_raw_daily` | **External Hive table** | Holds raw daily tolling records; provides a schema for downstream processing; partitions data by `partition_date` for efficient pruning. |
| Columns (`companyintegerid`, `companyname`, …, `eid`) | Table fields | Capture MVNO‑specific identifiers, invoice details, product & SIM information, timestamps, and source file metadata. |
| `partition_date` | Partition column | Enables daily isolation of data; used by ingestion scripts to add new partitions and by query engines for pruning. |
| Table properties (`external.table.purge`, `DO_NOT_UPDATE_STATS`, etc.) | Hive/Impala metadata | Control statistics generation, automatic purge of external data on DROP, and Impala catalog synchronization. |

*No procedural code (functions, classes) exists in this file; it is purely declarative.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - Parquet files placed in the HDFS location `.../tolling_raw_daily/partition_date=YYYY-MM-DD/` by upstream ingestion jobs (e.g., SFTP pull, Spark/MapReduce loader). <br> - Correct column ordering and data types matching the DDL. |
| **Outputs** | - Hive metastore entry for `tolling_raw_daily`. <br> - Logical view of the raw data for downstream SQL/Impala jobs (billing, mediation, reporting). |
| **Side‑Effects** | - Registers the table in Hive Metastore. <br> - Creates the HDFS directory if it does not exist (via `LOCATION`). <br> - Because of `external.table.purge='true'`, dropping the table will also delete the underlying files. |
| **Assumptions** | - The cluster runs Hive 3.x (or compatible) with Impala catalog integration. <br> - All incoming files are Parquet and conform to the defined schema. <br> - Partition folders are named exactly as the string value of `partition_date` (e.g., `partition_date=2024-01-15`). <br> - Proper HDFS permissions exist for the Hive/Impala service accounts to read/write the location. |

---

## 4. Interaction with Other Scripts & Components  

| Connected Component | Relationship |
|---------------------|--------------|
| **Ingestion pipelines** (e.g., `tolling_raw_daily_ingest.py`, Spark jobs) | Write Parquet files into the partitioned HDFS path; may invoke `MSCK REPAIR TABLE tolling_raw_daily` or `ALTER TABLE … ADD PARTITION` after file drop. |
| **Views / DDLs** such as `mnaas_vw_tcl_asset_pkg_mapping_billing.hql` (currently empty) | Expected to reference `tolling_raw_daily` to enrich raw records with asset‑package mappings. |
| **Billing & Mediation scripts** (`mnaas_v_traffic_details_*`, `mnaas_v_ictraffic_details_*`) | Join against `tolling_raw_daily` to pull invoice numbers, commercial offers, and SIM identifiers for charge calculation. |
| **Impala/Presto query engines** | Execute analytical queries on the table; rely on the `impala.events.*` properties for catalog sync. |
| **Hive Metastore** | Stores the table definition; any schema change requires a DDL alteration script. |
| **HDFS NameNode (`NN-HA1`)** | Physical storage location; data lifecycle (retention, archiving) managed by separate Hadoop admin scripts. |

*Note:* The exact script names for ingestion are not present in the repository snapshot; they are inferred from naming conventions used across the mediation‑ddls folder.

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – upstream source adds/renames columns. | Queries downstream may fail or return incorrect results. | Implement a schema‑validation step in the ingestion pipeline; version‑control DDL changes and run `DESCRIBE EXTENDED` checks before adding new partitions. |
| **Stale or missing partitions** – partitions not added after file drop. | Data for a given day is invisible to downstream jobs. | Automate partition discovery (`MSCK REPAIR TABLE`) as part of the ingest job; monitor Hive metastore partition count vs. expected dates. |
| **Uncontrolled storage growth** – raw files accumulate indefinitely. | HDFS capacity exhaustion. | Define a retention policy (e.g., delete partitions older than N days) and schedule a cleanup job; leverage `external.table.purge` for safe drops. |
| **Permission misconfiguration** – Hive/Impala service accounts lack read/write rights. | Job failures, data loss. | Audit HDFS ACLs; enforce least‑privilege policies; include permission checks in CI/CD validation. |
| **Accidental table drop** – `external.table.purge=true` will delete underlying files. | Irrecoverable data loss. | Restrict DROP privileges; enable Hive metastore backups; add a pre‑drop safeguard script that requires explicit confirmation. |

---

## 6. Running / Debugging the Table  

1. **Validate creation**  
   ```bash
   hive -e "SHOW CREATE TABLE mnaas.tolling_raw_daily\G"
   ```
   or in Impala:
   ```bash
   impala-shell -q "SHOW CREATE TABLE mnaas.tolling_raw_daily"
   ```

2. **Add a new partition (if not using MSCK)**  
   ```sql
   ALTER TABLE mnaas.tolling_raw_daily ADD IF NOT EXISTS
   PARTITION (partition_date='2024-10-01')
   LOCATION 'hdfs://NN-HA1/user/hive/warehouse/mnaas.db/tolling_raw_daily/partition_date=2024-10-01';
   ```

3. **Refresh metadata after files land**  
   ```bash
   hive -e "MSCK REPAIR TABLE mnaas.tolling_raw_daily;"
   ```

4. **Sample query**  
   ```sql
   SELECT partition_date, count(*) AS recs
   FROM mnaas.tolling_raw_daily
   GROUP BY partition_date
   ORDER BY partition_date DESC
   LIMIT 10;
   ```

5. **Debug missing data**  
   - Verify HDFS path exists and contains Parquet files (`hdfs dfs -ls …/partition_date=...`).  
   - Check Hive metastore partitions (`SHOW PARTITIONS tolling_raw_daily;`).  
   - Review ingestion logs for any file‑format errors.

---

## 7. External Config / Environment Dependencies  

| Item | Usage |
|------|-------|
| `NN-HA1` (NameNode address) | Hard‑coded in the `LOCATION` clause; must resolve in the cluster’s DNS. |
| Hive/Impala service principals | Required to read/write the HDFS directory; defined outside this script (usually via `hive-site.xml` / `impala-shell` configs). |
| Environment variables for Hadoop (`HADOOP_CONF_DIR`, `HIVE_CONF_DIR`) | Needed when executing Hive/Impala commands that reference this table. |
| Potential property files (`metastore-site.xml`) | Control `external.table.purge` behavior and statistics generation. |

If any of these values change (e.g., a new HDFS namespace), the DDL must be updated accordingly.

---

## 8. Suggested Improvements (TODO)

1. **Automate partition discovery** – Add a lightweight wrapper script (e.g., `add_tolling_partitions.sh`) that scans the HDFS directory for new `partition_date=` folders and issues `ALTER TABLE … ADD PARTITION` statements, then runs `MSCK REPAIR TABLE`. This reduces reliance on manual `MSCK` runs and prevents missing partitions.

2. **Add table comment & column-level documentation** – Include a `COMMENT` on the table itself (e.g., `'Raw daily tolling records received from MVNO partners'`) and ensure each column comment follows a consistent naming convention. This improves discoverability for downstream developers and data catalog tools.