**File:** `mediation-ddls\move_master_consolidated_1.hql`  
**Type:** HiveQL DDL – creates an **external** table `mnaas.move_master_consolidated_1`.

---

## 1. High‑level Summary
The script defines a wide‑column external Hive table that serves as the **master consolidation layer** for customer, credit‑limit, GAAP‑revenue, sponsor, and supplier data used throughout the mediation‑billing pipeline. Data is stored as a delimited text file (`~` field delimiter) on HDFS under the `mnaas.db` warehouse. Down‑stream billing, reporting, and analytics jobs read from this table; upstream ingestion jobs populate the underlying HDFS files (or write via `INSERT OVERWRITE`).

---

## 2. Core Objects Defined

| Object | Responsibility |
|--------|-----------------|
| **Table `mnaas.move_master_consolidated_1`** | Holds a flattened, denormalised view of multiple master data domains (customer mapping, credit limits, GAAP revenue, sponsor & supplier rate information). |
| **SerDe `LazySimpleSerDe`** | Parses the `~`‑delimited text files into Hive columns. |
| **Location `hdfs://NN-HA1/.../move_master_consolidated_1`** | Physical storage path for the external data files. |
| **Table properties** (`external.table.purge='true'`, `STATS_GENERATED='TASK'`, Impala catalog metadata) | Controls purge behaviour on DROP, enables statistics collection, and integrates with Impala. |

*No procedural code (functions, procedures) is present; the file’s sole purpose is schema definition.*

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Input data** | Text files placed (or overwritten) in the HDFS location. Expected column order matches the DDL; fields are delimited by `~`. |
| **Output** | Hive metadata for the external table; downstream queries return rows from the underlying files. |
| **Side effects** | - Registers the table in the Hive metastore.<br>- Because `external.table.purge='true'`, dropping the table will also delete the underlying HDFS files.<br>- Statistics may be generated automatically (see `STATS_GENERATED`). |
| **Assumptions** | - HDFS path exists and is writable by the Hive/Impala service accounts.<br>- Data files conform exactly to the column order and types (e.g., numeric fields are parsable as `double`).<br>- No partitioning is required for current workloads. |
| **External services** | HDFS namenode (`NN-HA1`), Hive Metastore, Impala catalog service (IDs shown in table properties). |

---

## 4. Integration Points (How It Connects to the Rest of the System)

| Direction | Connected Component | Interaction |
|-----------|---------------------|-------------|
| **Up‑stream** | Scripts that generate master data (e.g., `mnaas_vaz_dm_*`, `mnaas_v_traffic_details_*`, `move_kyc_sim_inventory_rnr_data.hql`) | These jobs write CSV/TSV files to the same HDFS directory or perform `INSERT OVERWRITE mnaas.move_master_consolidated_1 SELECT …` after joining/transforming source tables. |
| **Down‑stream** | Billing & reporting jobs (e.g., `mnaas_vw_tcl_asset_pkg_mapping_billing.hql`, revenue aggregation scripts) | Queries read from `move_master_consolidated_1` to enrich CDRs, calculate credit‑limit checks, and produce GAAP‑aligned invoices. |
| **Impala** | Impala daemons (catalog service ID shown) | The table is visible to Impala for low‑latency analytics; any schema change must be propagated to Impala via `INVALIDATE METADATA`. |
| **Data Quality / ETL orchestration** | Airflow / Oozie DAGs that schedule the “master‑consolidated” load | The DDL is executed once (or on schema change) before the DAG that populates the table runs. |
| **Security / Governance** | Ranger / Sentry policies | Access to the table and underlying HDFS path is controlled by role‑based policies; the DDL itself does not embed security settings. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – upstream jobs add/remove columns without updating this DDL. | Query failures, data mis‑alignment. | Enforce schema versioning; add CI test that compares column counts/types between source tables and this DDL. |
| **Data format mismatch** – delimiter or type errors in the raw files. | Rows rejected, NULLs, or job crashes. | Validate files with a lightweight Hive `SELECT * LIMIT 10` after each load; use schema‑aware ingestion (e.g., Spark) to enforce types before writing. |
| **External table purge** – accidental `DROP TABLE` removes raw files. | Permanent data loss. | Restrict DROP privileges; enable Hive metastore backup; consider setting `external.table.purge='false'` if data must be retained. |
| **HDFS storage pressure** – large flat file grows unchecked. | Slow queries, OOM in Hive/Impala. | Implement periodic compaction or partitioning (e.g., by month) and archive old data to cold storage. |
| **Impala catalog staleness** – schema changes not reflected. | Stale metadata, query errors. | Run `INVALIDATE METADATA` or `REFRESH` after any DDL change; automate via orchestration. |

---

## 6. Running / Debugging the Script

1. **Execute the DDL**  
   ```bash
   hive -f mediation-ddls/move_master_consolidated_1.hql
   # or via Beeline:
   beeline -u jdbc:hive2://<hs2-host>:10000 -f move_master_consolidated_1.hql
   ```

2. **Verify creation**  
   ```sql
   SHOW CREATE TABLE mnaas.move_master_consolidated_1;
   DESCRIBE FORMATTED mnaas.move_master_consolidated_1;
   ```

3. **Check data accessibility**  
   ```sql
   SELECT COUNT(*) FROM mnaas.move_master_consolidated_1 LIMIT 10;
   SELECT * FROM mnaas.move_master_consolidated_1 LIMIT 5;
   ```

4. **Debug common issues**  
   *Missing HDFS path*: `hdfs dfs -ls <location>` – create directory if absent.  
   *SerDe errors*: Verify that the file uses `~` as delimiter and no stray escape characters.  
   *Impala visibility*: `impala-shell -q "INVALIDATE METADATA mnaas.move_master_consolidated_1;"`

5. **Log locations**  
   Hive logs are under `$HIVE_LOG_DIR`; Impala catalog logs contain the IDs shown in table properties.

---

## 7. Configuration / Environment Dependencies

| Item | How It Is Used |
|------|----------------|
| **HDFS namenode address** (`NN-HA1`) | Determines the physical storage location of the external table. |
| **Hive Metastore URI** (`hive.metastore.uris`) | Required for the DDL to register the table. |
| **Impala catalog service ID / version** | Populated automatically; needed for Impala queries. |
| **SerDe properties** (`field.delim='~'`, `line.delim='\n'`) | Must match the delimiter used by upstream data generators. |
| **`external.table.purge`** | Controls whether dropping the table also deletes the underlying files. |
| **User permissions** (`last_modified_by='root'`) | The script assumes the executing user has write access to the HDFS path and metastore. |

If any of these values are overridden by environment variables (e.g., `HIVE_CONF_DIR`, `HADOOP_CONF_DIR`), ensure they point to the production configuration.

---

## 8. Suggested Improvements (TODO)

1. **Add Partitioning** – Introduce a logical partition (e.g., `load_date STRING` or `year_month STRING`) to improve query performance and enable incremental loads/archiving.  
2. **Document Column Semantics** – Add column comments (via `COMMENT '...'`) and a separate data‑dictionary markdown file; this will aid downstream developers and reduce schema‑drift risk.  

---