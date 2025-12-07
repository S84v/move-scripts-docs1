**File:** `mediation-ddls\mnaas_gen_sim_product_split.hql`  

---

## 1. High‑Level Summary
This script defines an **external Hive/Impala table** named `gen_sim_product_split`. The table maps a flat‑file dataset stored under the HDFS path `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/gen_sim_product_split` to a set of 13 columns describing SIM‑product attributes (e.g., product name, plan dates, pricing codes, usage model, grace period, billing mode). Because the table is *external* and has `external.table.purge='true'`, Hive metadata is created without moving data, and dropping the table will also delete the underlying files. The definition is part of the broader “MNAAS” mediation layer that prepares SIM‑product data for downstream billing, reporting, and provisioning pipelines.

---

## 2. Key Objects & Responsibilities
| Object | Type | Responsibility |
|--------|------|----------------|
| `gen_sim_product_split` | External Hive/Impala table | Provides a structured view over raw CSV‑like files containing SIM‑product records; used by downstream scripts (e.g., `mnaas_gen_sim_product.hql`) for joins, transformations, and loading into billing tables. |
| `LazySimpleSerDe` | SerDe implementation | Parses delimited text (default delimiter `\t` unless overridden by table properties) into the defined column types. |
| `TextInputFormat` / `HiveIgnoreKeyTextOutputFormat` | Input/Output formats | Reads plain text files from HDFS; output format is a no‑op because the table is external (data is never written by Hive). |
| Table properties (`bucketing_version`, `external.table.purge`, `impala.events.*`, `transient_lastDdlTime`) | Metadata | Controls bucket versioning, purge behavior on DROP, and Impala catalog synchronization. |

*No procedural code (functions, procedures, UDFs) is present in this file.*

---

## 3. Interfaces (Inputs, Outputs, Side‑Effects, Assumptions)

| Aspect | Details |
|--------|---------|
| **Inputs** | - Raw text files located at `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/gen_sim_product_split`.<br>- Files must conform to the column order and data types defined (e.g., `secs_code` as `int`). |
| **Outputs** | - Hive/Impala metadata entry for `gen_sim_product_split`.<br>- Queryable table for downstream jobs (SELECT, JOIN, INSERT … SELECT). |
| **Side‑Effects** | - Registers the external table in the Hive metastore.<br>- Because `external.table.purge='true'`, a subsequent `DROP TABLE gen_sim_product_split` will **delete** the underlying HDFS files. |
| **Assumptions** | - HDFS namenode alias `NN-HA1` resolves correctly in the execution environment.<br>- The target directory exists and is readable/writable by the Hive/Impala service accounts.<br>- No custom delimiters are needed; default tab delimiter is acceptable.<br>- No partitioning is required for current data volume. |
| **External Services** | - HDFS (storage).<br>- Hive Metastore (metadata service).<br>- Impala catalog (via `impala.events.*` properties). |

---

## 4. Integration Points with Other Scripts / Components
| Connected Component | How the Connection Occurs |
|---------------------|----------------------------|
| `mediation-ddls\mnaas_gen_sim_product.hql` | Likely **reads** from `gen_sim_product_split` to enrich or aggregate SIM‑product data before inserting into a production billing table. |
| Downstream billing pipelines (e.g., `mnaas_billing_*` scripts) | May **join** this table with usage or subscriber tables to apply product‑specific pricing rules. |
| Data ingestion jobs (ETL frameworks such as Sqoop, Flume, or custom Spark jobs) | Populate the HDFS directory with the raw CSV files that this external table points to. |
| Monitoring / data quality jobs | Use `DESCRIBE EXTENDED gen_sim_product_split` or `SHOW CREATE TABLE` to verify schema consistency. |
| Impala UI / Hue / other query tools | Provide ad‑hoc access for analysts to validate the data before it is consumed downstream. |

*Because the table is external, any process that writes files to the target HDFS path automatically makes the data visible to Hive/Impala without further DDL changes.*

---

## 5. Operational Risks & Recommended Mitigations
| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – source files change column order or delimiter. | Queries downstream may fail or produce corrupted results. | Enforce a schema validation step (e.g., a Spark job that checks column count/types) before files are landed. |
| **Accidental data loss** – `DROP TABLE` removes underlying files due to `external.table.purge='true'`. | Permanent loss of raw SIM‑product data. | Restrict DROP privileges; add a pre‑drop approval workflow; consider setting `external.table.purge='false'` if data must be retained. |
| **Permission issues** – Hive/Impala service accounts lack read/write rights on the HDFS directory. | Table creation succeeds but queries return “Permission denied”. | Verify HDFS ACLs and POSIX permissions; include a health‑check script that runs `hdfs dfs -test -e <path>`. |
| **Missing data files** – directory empty or partially populated. | Downstream jobs produce zero rows or incomplete billing. | Implement a file‑arrival watchdog (e.g., Oozie or Airflow sensor) that confirms expected file count before enabling downstream DAGs. |
| **Performance degradation** – large unpartitioned table leads to full scans. | Increased query latency on billing reports. | Evaluate partitioning (e.g., by `plan_effective_date`) or bucketing if data volume grows. |

---

## 6. Execution & Debugging Guide
1. **Run the DDL**  
   ```bash
   hive -f mediation-ddls/mnaas_gen_sim_product_split.hql
   # or via Impala
   impala-shell -f mediation-ddls/mnaas_gen_sim_product_split.hql
   ```
2. **Validate Creation**  
   ```sql
   SHOW CREATE TABLE gen_sim_product_split;
   DESCRIBE FORMATTED gen_sim_product_split;
   ```
3. **Inspect Sample Data**  
   ```sql
   SELECT * FROM gen_sim_product_split LIMIT 10;
   ```
4. **Check Underlying Files** (on the Hadoop edge node)  
   ```bash
   hdfs dfs -ls /user/hive/warehouse/mnaas.db/gen_sim_product_split
   hdfs dfs -cat /user/hive/warehouse/mnaas.db/gen_sim_product_split/part-00000
   ```
5. **Debug Common Issues**  
   - *Empty result set*: Verify files exist and are not zero‑byte.  
   - *Column parsing errors*: Look for delimiter mismatches; consider adding `ROW FORMAT DELIMITED FIELDS TERMINATED BY ','` if CSV.  
   - *Permission errors*: Check HDFS ACLs (`hdfs dfs -getfacl <path>`).  

---

## 7. Configuration / Environment Dependencies
| Item | Usage |
|------|-------|
| `NN-HA1` (HDFS namenode alias) | Hard‑coded in the `LOCATION` clause; must resolve via the cluster’s DNS or `/etc/hosts`. |
| Hive Metastore connection | Implicit; the script assumes the Hive client is correctly configured (`hive-site.xml`). |
| Impala catalog IDs (`impala.events.catalogServiceId`, `catalogVersion`) | Auto‑generated by Impala; not manually edited but must be present for Impala to sync metadata. |
| Optional: `hive.exec.scratchdir`, `hive.metastore.warehouse.dir` | Not referenced directly but affect where temporary files are written during DDL execution. |

If the environment uses parameterised paths (e.g., `${WAREHOUSE_DIR}`), they are **not** present in this file; any change would require editing the `LOCATION` string.

---

## 8. Suggested Improvements (TODO)
1. **Add explicit delimiter and null handling** – e.g.,  
   ```sql
   ROW FORMAT DELIMITED
     FIELDS TERMINATED BY ','
     COLLECTION ITEMS TERMINATED BY '\002'
     MAP KEYS TERMINATED BY '\003'
   STORED AS TEXTFILE;
   ```
   This removes ambiguity about the default tab delimiter and improves compatibility with CSV sources.

2. **Introduce partitioning** (e.g., by `plan_effective_date`) to reduce scan size for downstream queries and enable incremental loads:
   ```sql
   PARTITIONED BY (plan_effective_date STRING)
   ```
   Follow with an `MSCK REPAIR TABLE` or `ALTER TABLE … ADD PARTITION` step after data landing.

---