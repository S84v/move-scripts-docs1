**File:** `mediation-ddls\mnaas_tolling_eid_list.hql`  

---

## 1. High‑Level Summary
This script creates (or re‑creates) the Hive external table **`mnaas.tolling_eid_list`**. The table stores a flat file of tolling‑related identifiers – an `eid`, the associated `tcl_secs_id`, the owning `orgname`, and the `commercialoffer`. Data is partitioned by a `month` string column and lives in a dedicated HDFS directory (`hdfs://NN-HA1/user/hive/warehouse/mnaas.db/tolling_eid_list`). The table is marked as *external* with `external.table.purge='true'`, meaning Hive will not retain data after a DROP and the underlying files are managed outside Hive. The definition includes standard SerDe/IO formats and a set of table properties used by Impala for catalog synchronization.

---

## 2. Core Definition (the “class” of this file)

| Element | Responsibility |
|---------|-----------------|
| **CREATE EXTERNAL TABLE `mnaas`.`tolling_eid_list`** | Declares the logical schema and storage location for tolling‑EID data. |
| Columns (`eid`, `tcl_secs_id`, `orgname`, `commercialoffer`) | Holds the raw attributes required by downstream charging and reporting jobs. |
| **PARTITIONED BY (`month` string)** | Enables month‑level pruning for queries and incremental loads. |
| **ROW FORMAT SERDE** `LazySimpleSerDe` | Parses delimited text (default `\001` field delimiter). |
| **STORED AS INPUTFORMAT / OUTPUTFORMAT** | Uses plain text input and Hive‑ignore‑key output – compatible with most batch loaders (Sqoop, Spark, Flink, etc.). |
| **LOCATION** `hdfs://NN-HA1/.../tolling_eid_list` | Physical path where source files are dropped or generated. |
| **TBLPROPERTIES** (e.g., `external.table.purge`, Impala catalog IDs) | Controls lifecycle, statistics, and cross‑engine visibility. |

*No procedural code (functions, procedures) is present – the file is a pure DDL definition.*

---

## 3. Inputs, Outputs & Side Effects

| Aspect | Details |
|--------|---------|
| **Inputs** | Implicit – the HDFS directory referenced in `LOCATION`. Data files placed there (usually by upstream ETL jobs) become visible to Hive/Impala. |
| **Outputs** | A Hive metastore entry for `mnaas.tolling_eid_list`. No data is written by this script itself. |
| **Side Effects** | - Registers the external table in the Hive metastore.<br>- Updates Impala catalog metadata (via the `impala.*` properties).<br>- Because `external.table.purge='true'`, a later `DROP TABLE` will delete the underlying HDFS files. |
| **Assumptions** | - The HDFS namenode alias `NN-HA1` resolves in the execution environment.<br>- The Hive warehouse directory (`/user/hive/warehouse/mnaas.db/`) is writable by the user running the script.<br>- The cluster’s Hive/Impala versions support the listed SerDe and table properties.<br>- Up‑stream processes will write files that match the column order and delimiter expectations. |

---

## 4. Integration Points (How this file fits into the broader move system)

| Connected Component | Interaction |
|---------------------|-------------|
| **Data ingestion jobs** (e.g., Spark batch, Sqoop import, custom Python/Java loaders) | Write monthly‑partitioned CSV/TSV files into the HDFS location. They rely on the partition column (`month`) being present in the file path or added via `ALTER TABLE … ADD PARTITION`. |
| **Transformation scripts** (`*.hql` files that reference `tolling_eid_list`) | Perform joins with other mediation tables such as `mnaas_tcl_service_asset_mapping_summary`, `mnaas_ra_traffic_details_summary`, etc., to enrich charging data. |
| **Reporting / analytics pipelines** (Impala queries, Tableau/PowerBI connectors) | Query the table directly, often filtered by `month`. The Impala catalog IDs stored in `TBLPROPERTIES` ensure the table appears in the Impala schema without extra refresh steps. |
| **Retention / cleanup jobs** (e.g., `mnaas_tolling_eid_list_cleanup.hql` if it exists) | May drop old partitions or the whole table; the `external.table.purge` flag determines whether data is physically removed. |
| **Orchestration layer** (Airflow, Oozie, Control-M) | Executes this DDL as part of a “schema‑setup” task before any load jobs run. The task typically runs `hive -f mnaas_tolling_eid_list.hql`. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Accidental data loss** – `external.table.purge='true'` + a stray `DROP TABLE` | Permanent deletion of raw tolling files. | - Enforce role‑based access control (RBAC) on Hive metastore.<br>- Add a pre‑drop safeguard script that checks for recent backups.<br>- Consider setting `external.table.purge='false'` if data must be retained after a drop. |
| **Partition drift** – missing or mis‑named month partitions cause query failures or full‑table scans. | Performance degradation, incorrect billing. | - Automate partition addition (`ALTER TABLE … ADD IF NOT EXISTS PARTITION (month='2024-09') LOCATION ...`).<br>- Validate incoming file naming conventions before load. |
| **Schema mismatch** – upstream loaders produce extra columns or different order. | Query errors, silent data corruption. | - Define a strict file format (e.g., use `FIELDS TERMINATED BY ','` explicitly).<br>- Add a validation step (e.g., Spark schema enforcement) before moving files to the location. |
| **Impala catalog out‑of‑sync** – stale `impala.events.*` properties after a table recreation. | Impala queries return “Table not found” or stale metadata. | - Run `INVALIDATE METADATA mnaas.tolling_eid_list;` or `REFRESH` after DDL execution.<br>- Include the refresh as part of the orchestration task. |
| **HDFS path availability** – `NN-HA1` namenode alias changes or network partition. | Table creation fails, downstream jobs blocked. | - Use a configuration file (e.g., `hive-site.xml`) to resolve namenode addresses.<br>- Add health‑check steps before DDL execution. |

---

## 6. Running / Debugging the Script

1. **Standard execution** (from a shell or orchestration task)  
   ```bash
   hive -f mediation-ddls/mnaas_tolling_eid_list.hql
   ```
   *or* if using Impala:  
   ```bash
   impala-shell -i <impala-coordinator> -f mediation-ddls/mnaas_tolling_eid_list.hql
   ```

2. **Verification**  
   ```sql
   SHOW CREATE TABLE mnaas.tolling_eid_list;
   DESCRIBE FORMATTED mnaas.tolling_eid_list;
   ```

3. **Debugging tips**  
   - Check Hive metastore logs (`/var/log/hive/hive-metastore.log`) for permission or path errors.  
   - Verify the HDFS directory exists and is empty (or contains expected files) with `hdfs dfs -ls /user/hive/warehouse/mnaas.db/tolling_eid_list`.  
   - If Impala cannot see the table, run `INVALIDATE METADATA mnaas.tolling_eid_list;` and re‑query.  
   - Use `hive -e "SELECT * FROM mnaas.tolling_eid_list LIMIT 10;"` to confirm data visibility after a load.

---

## 7. External Configuration & Environment Dependencies

| Item | Usage |
|------|-------|
| **HDFS namenode alias** `NN-HA1` | Resolved via the cluster’s `core-site.xml` or DNS; points to the active namenode for the Hive warehouse. |
| **Hive Metastore connection** (usually via `hive-site.xml`) | Required for DDL registration. |
| **Impala catalog service IDs** (`impala.events.catalogServiceId`, `impala.events.catalogVersion`) | Populated automatically by Impala; not manually edited but may be referenced by downstream scripts that sync catalog state. |
| **Environment variables** (e.g., `HIVE_CONF_DIR`, `HADOOP_CONF_DIR`) | Must be set for the `hive`/`impala-shell` commands to locate configuration files. |
| **Security tokens / Kerberos principal** (if enabled) | The user executing the script must have a valid ticket (`kinit`) to access HDFS and Hive. |

---

## 8. Suggested Improvements (TODO)

1. **Add `IF NOT EXISTS` guard** to make the script idempotent:  
   ```sql
   CREATE EXTERNAL TABLE IF NOT EXISTS mnaas.tolling_eid_list ( ... );
   ```

2. **Explicitly define field and line delimiters** (e.g., `FIELDS TERMINATED BY ','` and `LINES TERMINATED BY '\n'`) to avoid reliance on default SerDe settings and to make the file format self‑documenting.

*(Optional extensions: implement a small “partition‑maintenance” script that automatically adds the current month partition after table creation, and schedule an `INVALIDATE METADATA` call for Impala.)*