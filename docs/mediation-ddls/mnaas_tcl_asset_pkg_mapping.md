**File:** `mediation-ddls\mnaas_tcl_asset_pkg_mapping.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates (or replaces) the `tcl_asset_pkg_mapping` table in the `mnaas` database. The table stores a mapping between a TCL‑specific asset identifier (`tcl_secs_id`) and the corresponding package information (`packagename`, `vin`, `devicetype`, `services`). Data is partitioned by `year_month` and persisted in a dedicated HDFS directory under the Hive warehouse. The table is used downstream by mediation jobs that enrich or validate asset‑package relationships during the processing of TCL‑originated usage files.

---

## 2. Key Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `tcl_asset_pkg_mapping` | Hive **managed** table | Holds asset‑to‑package mapping rows; partitioned by `year_month` for incremental loads. |
| Columns (`tcl_secs_id`, `packagename`, `vin`, `devicetype`, `services`) | String fields | Capture raw identifiers and service list as supplied by upstream TCL data feeds. |
| Partition column `year_month` | String | Enables month‑level data isolation; used by downstream jobs for time‑bounded processing. |
| `ROW FORMAT SERDE` (`LazySimpleSerDe`) | SerDe definition | Parses CSV‑style text files (default field delimiter `\001`). |
| `STORED AS INPUTFORMAT/OUTPUTFORMAT` | Text file format | Stores data as plain text; compatible with Hadoop MapReduce and Spark readers. |
| `LOCATION` | HDFS path | Physical storage location: `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/tcl_asset_pkg_mapping`. |
| `TBLPROPERTIES` | Metadata | Tracks DDL metadata (`bucketing_version`, `last_modified_by`, timestamps). |

*No procedural code (functions, procedures) is present; the file is pure DDL.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | None at execution time. The script relies on Hive metastore configuration and HDFS connectivity. |
| **Outputs** | - A new Hive table definition registered in the metastore.<br>- A physical directory created (if not existing) at the specified HDFS location.<br>- Table metadata (`TBLPROPERTIES`). |
| **Side‑Effects** | - If the table already exists, Hive will **replace** it (default `CREATE TABLE` behavior may fail; check `IF NOT EXISTS` usage in the production wrapper).<br>- Existing data in the location may be overwritten or become orphaned if the table is dropped/re‑created. |
| **Assumptions** | - Hive metastore is reachable and the user executing the script has `CREATE`/`ALTER` privileges on the `mnaas` database.<br>- HDFS namenode alias `NN-HA1` resolves correctly in the cluster’s `core-site.xml`.<br>- The target HDFS directory is writable by the Hive user.<br>- Downstream jobs expect the table to be **partitioned** on `year_month` and stored as plain text. |
| **External Services** | - **HiveServer2 / Beeline** (or `hive` CLI) for DDL execution.<br>- **HDFS** for data storage.<br>- Potential downstream **ETL jobs** (Spark, MapReduce, or custom scripts) that read this table. |

---

## 4. Integration Points  

| Connected Component | Interaction |
|---------------------|-------------|
| **Ingestion scripts** (e.g., `mnaas_tcl_asset_pkg_load.hql` or Spark jobs) | Load raw TCL asset files into the `tcl_asset_pkg_mapping` table, usually via `INSERT INTO … PARTITION (year_month=…) SELECT …`. |
| **Transformation jobs** (e.g., `mnaas_ra_traffic_details_summary.hql`) | Join on `tcl_secs_id` to enrich traffic records with package information. |
| **Reporting / Analytics** (e.g., BI dashboards) | Query the table for asset‑package statistics, often filtered by `year_month`. |
| **Data quality / validation** (e.g., `mnaas_file_rej_geneva.hql`) | May reference this table to flag records with missing or mismatched package data. |
| **Orchestration layer** (e.g., Oozie, Airflow, or custom scheduler) | Executes this DDL as a **pre‑step** before any load jobs that depend on the table’s existence. |
| **Configuration files** | The script may be invoked via a wrapper that injects environment variables such as `HIVE_CONF_DIR`, `HADOOP_USER_NAME`, or a custom `MNAAS_DB` variable. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Table recreation overwrites data** | Loss of historical mapping data if the script runs unintentionally on a populated table. | Use `CREATE TABLE IF NOT EXISTS` in the production wrapper; protect the table with ACLs; add a pre‑run check (`SHOW TABLES LIKE 'tcl_asset_pkg_mapping'`). |
| **Partition mis‑alignment** | Downstream jobs may miss data if `year_month` values are inconsistent (e.g., wrong format). | Enforce a strict `YYYYMM` format via validation in load jobs; add a Hive constraint or comment in the DDL. |
| **HDFS path permission errors** | Job failure at table creation or data load. | Verify Hive user has `rwx` on the target directory; include a health‑check step (`hdfs dfs -test -d <path>`). |
| **SerDe incompatibility** | Incorrect parsing of source files leading to malformed rows. | Confirm source files use the default field delimiter (`\001`) or explicitly set `FIELDS TERMINATED BY` in the load statement. |
| **Metadata drift** | Table properties (e.g., `last_modified_time`) become out‑of‑sync with actual schema changes. | Automate DDL versioning (e.g., store a checksum in a `metadata_version` table) and compare before applying changes. |

---

## 6. Running & Debugging the Script  

1. **Preparation**  
   ```bash
   export HIVE_CONF_DIR=/etc/hive/conf
   export HADOOP_USER_NAME=hive
   # (Optional) source environment that defines NN-HA1 alias
   ```

2. **Execute** (via Beeline or Hive CLI)  
   ```bash
   beeline -u jdbc:hive2://<hiveserver2-host>:10000/default -n hive -p <pwd> -f mediation-ddls/mnaas_tcl_asset_pkg_mapping.hql
   ```

3. **Verify Creation**  
   ```sql
   SHOW CREATE TABLE mnaas.tcl_asset_pkg_mapping;
   DESCRIBE FORMATTED mnaas.tcl_asset_pkg_mapping;
   ```

4. **Check HDFS Directory**  
   ```bash
   hdfs dfs -ls /user/hive/warehouse/mnaas.db/tcl_asset_pkg_mapping
   ```

5. **Debugging Tips**  
   - If the script fails with *Table already exists*, add `IF NOT EXISTS` or drop the table first (`DROP TABLE IF EXISTS mnaas.tcl_asset_pkg_mapping;`).  
   - Permission errors: run `hdfs dfs -chmod -R 775 <path>` or adjust Hive ACLs.  
   - Partition visibility: after loading data, run `MSCK REPAIR TABLE mnaas.tcl_asset_pkg_mapping;` to refresh partitions.  

---

## 7. External Config / Environment Dependencies  

| Item | Usage |
|------|-------|
| `HIVE_CONF_DIR` | Points to Hive configuration (metastore URI, execution engine). |
| `HADOOP_USER_NAME` | Determines the HDFS user under which the table location is created. |
| `NN-HA1` (namenode alias) | Resolved via Hadoop’s `core-site.xml` to the active NameNode; must be reachable from the Hive server. |
| Optional wrapper variables (e.g., `MNAAS_DB`) | May be used by orchestration scripts to inject the target database name; not required by the DDL itself. |

---

## 8. Suggested Improvements (TODO)

1. **Add `IF NOT EXISTS` guard** – Prevent accidental table recreation and data loss.  
   ```sql
   CREATE TABLE IF NOT EXISTS mnaas.tcl_asset_pkg_mapping ( ... );
   ```

2. **Explicitly define field delimiter and optional compression** – Improves compatibility with downstream loaders and reduces storage cost.  
   ```sql
   ROW FORMAT DELIMITED
     FIELDS TERMINATED BY '\t'
   STORED AS ORC
   TBLPROPERTIES ('orc.compress'='ZLIB');
   ```

   *(Choose delimiter/compression that matches the upstream data format.)*

---