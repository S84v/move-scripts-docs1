**File:** `mediation-ddls\mnaas_billing_traffic_filtered_cdr.hql`  

---

### 1. High‑Level Summary
This script creates an **external Hive/Impala table** named `mnaas.billing_traffic_filtered_cdr`. The table maps raw, filtered Call Detail Record (CDR) files stored in HDFS to a structured schema that downstream billing and analytics jobs can query. Because the table is external and marked with `external.table.purge='true'`, dropping the table will also delete the underlying files, making the definition the single point of truth for the physical data location.

---

### 2. Key Objects & Responsibilities
| Object | Type | Responsibility |
|--------|------|-----------------|
| `billing_traffic_filtered_cdr` | Hive external table | Provides a column‑wise view of filtered CDR data for billing calculations, usage aggregation, and reporting. |
| `createtab_stmt` | DDL statement (named block) | Encapsulates the `CREATE EXTERNAL TABLE` command; used by the orchestration layer to execute the DDL. |
| `ROW FORMAT SERDE` | Hive SerDe definition | Uses `LazySimpleSerDe` to parse delimited text files (default delimiter is `\001`). |
| `INPUTFORMAT / OUTPUTFORMAT` | Hadoop I/O classes | Reads plain text files (`TextInputFormat`) and writes using Hive’s ignore‑key text format (no map‑reduce output needed for external tables). |
| `TBLPROPERTIES` | Metadata | Controls external table purge, Impala catalog sync, and timestamps for DDL tracking. |

---

### 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - HDFS directory `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/billing_traffic_filtered_cdr` containing the filtered CDR files.<br>- Implicit environment: Hive metastore, Impala catalog service (`db1f91781e2d40c4:acb25cfd944febad`). |
| **Outputs** | - Table metadata registered in Hive metastore (and propagated to Impala).<br>- No data transformation; the table simply exposes existing files. |
| **Side Effects** | - Creates a new entry in the Hive metastore.<br>- Because `external.table.purge='true'`, a subsequent `DROP TABLE` will delete the HDFS files. |
| **Assumptions** | - The HDFS path exists and is readable by the Hive/Impala service accounts.<br>- Files are delimited text matching the column order and data types defined.<br>- No partitioning is required for current query patterns.<br>- The cluster’s `LazySimpleSerDe` default field delimiter (`\001`) matches the source file format. |

---

### 4. Integration Points with Other Scripts / Components  

| Connected Component | Interaction |
|---------------------|-------------|
| **Up‑stream CDR filtering jobs** (`mnaas_billing_traffic_*` scripts) | Produce the filtered CDR files that land in the HDFS location referenced by this table. |
| **Down‑stream billing calculations** (`mnaas_billing_*` DDLs and ETL jobs) | Query `billing_traffic_filtered_cdr` to compute usage, chargeable units, and generate billing records. |
| **Impala query layer** | Uses the table for ad‑hoc analysis and scheduled reporting; the `impala.events.catalog*` properties keep Impala in sync. |
| **Data retention / purge processes** | May invoke `DROP TABLE mnaas.billing_traffic_filtered_cdr` as part of a data‑lifecycle policy; the purge flag ensures physical cleanup. |
| **Monitoring / Auditing** | Metastore audit logs capture the DDL execution; the `transient_lastDdlTime` property is used by housekeeping scripts to detect stale definitions. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – source CDR files change column order or data type. | Queries fail or return corrupt data. | Add a validation step (e.g., a Hive `DESCRIBE` + sample row check) before table creation; version the DDL. |
| **Missing or inaccessible HDFS path**. | Table creation succeeds but queries return empty or error. | Pre‑flight check that the directory exists and has correct permissions (`hdfs dfs -test -d …`). |
| **Accidental data loss** due to `external.table.purge='true'`. | Dropping the table removes raw files permanently. | Restrict DROP privileges; implement a “soft‑drop” process that first copies data to a backup location. |
| **Incorrect SerDe delimiter**. | Data parsing errors, mis‑aligned columns. | Document the expected delimiter; consider adding `FIELDS TERMINATED BY '\001'` explicitly. |
| **Impala catalog out‑of‑sync** after DDL changes. | Queries in Impala see stale metadata. | Run `INVALIDATE METADATA` or `REFRESH` after DDL execution; monitor the `impala.events.catalogVersion` property. |

---

### 6. Running / Debugging the Script  

| Step | Command | Notes |
|------|---------|-------|
| **Execute** | `hive -f mediation-ddls/mnaas_billing_traffic_filtered_cdr.hql` <br>or <br>`impala-shell -f mediation-ddls/mnaas_billing_traffic_filtered_cdr.hql` | Use the same Hive/Impala version as production. |
| **Verify creation** | `SHOW CREATE TABLE mnaas.billing_traffic_filtered_cdr;` | Confirm location, SerDe, and properties. |
| **Test data visibility** | `SELECT COUNT(*) FROM mnaas.billing_traffic_filtered_cdr LIMIT 10;` | Should return a non‑zero count if files are present. |
| **Check HDFS files** | `hdfs dfs -ls /user/hive/warehouse/mnaas.db/billing_traffic_filtered_cdr` | Ensure files exist and are readable. |
| **Debug failures** | - Review Hive metastore logs (`/var/log/hive/hive-metastore.log`).<br>- Check Impala daemon logs for catalog sync errors.<br>- Use `EXPLAIN` on a sample query to see parsing plan. | |

---

### 7. External Configuration / Environment Variables  

| Config / Env | Usage |
|--------------|-------|
| `NN-HA1` (NameNode host) | Hard‑coded in the HDFS URI; may be overridden by a cluster‑wide `fs.defaultFS` property. |
| Hive metastore connection (`hive.metastore.uris`) | Required for DDL registration; not referenced directly in the script but assumed to be set in the Hive client environment. |
| Impala catalog service ID (`impala.events.catalogServiceId`) | Embedded in `TBLPROPERTIES`; matches the Impala catalog instance used in production. |
| Optional: `HIVE_CONF_DIR`, `IMPALA_HOME` | Needed when invoking the script from a CI/CD pipeline or operator console. |

If the environment changes (e.g., a new NameNode or different warehouse path), the script must be updated accordingly.

---

### 8. Suggested Improvements (TODO)

1. **Add `IF NOT EXISTS` guard** – Prevents errors on re‑run:  
   ```sql
   CREATE EXTERNAL TABLE IF NOT EXISTS mnaas.billing_traffic_filtered_cdr ( ... )
   ```
2. **Introduce partitioning** (e.g., by `calldate` or `country`) to improve query performance and enable incremental data loading.  
   ```sql
   PARTITIONED BY (calldate string)
   ```
   Follow with an `MSCK REPAIR TABLE` or `ALTER TABLE ... ADD PARTITION` step in the ingestion pipeline.

---