**File:** `mediation-ddls\dim_date.hql`  

---

### 1. Summary  
This script creates (or replaces) an **external Hive table** named `mnaas.dim_date`. The table stores a classic date‑dimension used throughout the mediation layer – one row per calendar day with multiple derived attributes (e.g., Excel serial number, YYYYMMDD string, fiscal month/quarter). The data resides in an HDFS directory (`/user/hive/warehouse/mnaas.db/dim_date`) and is read as a semicolon‑delimited text file. Because the table is *external* and has `external.table.purge='true'`, Hive will not delete the underlying files when the table is dropped, but it will clean up metadata on purge operations.

---

### 2. Key Definitions (DDL Elements)

| Element | Responsibility |
|---------|-----------------|
| **CREATE EXTERNAL TABLE `mnaas`.`dim_date`** | Registers the table in the Hive metastore without owning the data files. |
| **Columns** | <ul><li>`date_num` (bigint) – Excel serial number (days since 1‑Jan‑1900).</li><li>`date_yyyymmdd` (string) – Date formatted as `YYYYMMDD`.</li><li>`date` (string) – Raw date string (source format).</li><li>`day_no_of_week` (bigint) – 1‑7 (Mon‑Sun).</li><li>`day_name` (bigint) – Numeric day name (likely 1=Monday … 7=Sunday).</li><li>`day_no_of_month` (bigint) – 1‑31.</li><li>`month_name` (string) – Full month name.</li><li>`month_number` (bigint) – 1‑12.</li><li>`quarter` (bigint) – 1‑4.</li><li>`year` (bigint) – Calendar year.</li><li>`fiscal_month` (bigint) – Fiscal month number.</li><li>`fiscal_quarter` (bigint) – Fiscal quarter number.</li><li>`start_of_month` (string) – First day of month (YYYY‑MM‑DD).</li><li>`end_of_month` (string) – Last day of month (YYYY‑MM‑DD).</li></ul> |
| **ROW FORMAT SERDE** | Uses `LazySimpleSerDe` with `field.delim=';'` and `line.delim='\n'`. |
| **STORED AS INPUTFORMAT / OUTPUTFORMAT** | Text‑based input (`TextInputFormat`) and Hive‑ignore‑key output format – suitable for plain CSV‑like files. |
| **LOCATION** | `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/dim_date` – the physical directory that holds the source files. |
| **TBLPROPERTIES** | Includes statistics generation flags, Impala catalog IDs, purge flag, and audit fields (`last_modified_by`, `last_modified_time`). |

---

### 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | Existing files in the HDFS location. Expected format: semicolon‑delimited rows matching the column order above. |
| **Outputs** | No data is written by this script. The result is a **metadata object** (the external table) that downstream jobs can query. |
| **Side Effects** | • Registers the table in Hive metastore.<br>• Updates Impala catalog (via `impala.events.*` properties).<br>• May trigger automatic statistics collection if enabled. |
| **Assumptions** | • HDFS path `hdfs://NN-HA1/.../dim_date` is reachable and has proper permissions for the Hive/Impala service accounts.<br>• Files conform to the defined schema (no missing columns, correct delimiter).<br>• The cluster’s Hive version supports the listed SERDE and input formats.<br>• No partitioning is required for this dimension (full scan acceptable). |

---

### 4. Integration Points (How This File Connects to the Rest of the System)

| Connected Component | Interaction |
|---------------------|-------------|
| **ETL/Load Jobs** (e.g., `dim_date_load.py`, `populate_dim_date.hql`) | These jobs generate or refresh the source files in the HDFS location before this DDL is executed (or after, if the table already exists). |
| **Fact Tables** (e.g., `fact_cdr`, `fact_usage`) | Join on `date_num` or `date_yyyymmdd` to enrich transactional data with calendar attributes. |
| **Reporting / BI Layer** (Impala, Tableau, PowerBI) | Queries reference `mnaas.dim_date` for date slicers, fiscal calculations, etc. |
| **Data Governance / Lineage** (e.g., Apache Atlas) | The external table is a node; lineage tools capture the upstream file source and downstream fact table usage. |
| **Scheduler / Orchestration** (e.g., Oozie, Airflow) | A DAG task runs `hive -f dim_date.hql` as part of the “schema provisioning” stage of a nightly pipeline. |
| **Security / Access Control** | Hive/Impala ACLs grant SELECT to analytics groups; write access is limited to the ETL service account. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or malformed source files** | Queries return empty or error rows. | Add a pre‑execution validation step (e.g., `hdfs dfs -test -e <path>/part-*` and schema sanity check). |
| **Schema drift** (column order change in source files) | Data mis‑aligned, causing incorrect attribute values. | Enforce a strict schema contract in the upstream load job; optionally enable `serde` property `skip.header.line.count='1'` and use column names in the file header. |
| **Permission changes on HDFS location** | Hive/Impala cannot read the data, leading to job failures. | Periodically audit HDFS ACLs; store required permissions in a config file and enforce via a CI check. |
| **External table purge flag** (`external.table.purge='true'`) | Accidental `DROP TABLE` will delete underlying files. | Restrict DROP privileges; use `ALTER TABLE ... SET TBLPROPERTIES ('external.table.purge'='false')` for production, or lock the table via Hive metastore policies. |
| **Impala catalog version mismatch** | Stale metadata causing query failures. | Run `INVALIDATE METADATA mnaas.dim_date;` after any file change, or schedule a regular `REFRESH` in the orchestration. |
| **Performance** (full table scans) | Large fact tables joining to dim_date may cause high I/O. | Consider adding a small partition (e.g., by year) if the dimension grows beyond a few thousand rows, or enable caching in Impala. |

---

### 6. Execution & Debugging Guide  

1. **Run the DDL**  
   ```bash
   hive -f mediation-ddls/dim_date.hql
   # or using Beeline with HiveServer2
   beeline -u jdbc:hive2://<hs2-host>:10000/default -f mediation-ddls/dim_date.hql
   ```

2. **Verify Creation**  
   ```sql
   SHOW CREATE TABLE mnaas.dim_date;
   DESCRIBE FORMATTED mnaas.dim_date;
   ```

3. **Check Data Accessibility**  
   ```sql
   SELECT COUNT(*) FROM mnaas.dim_date LIMIT 10;
   ```

4. **Debug Common Issues**  
   - *“Table not found”* → Verify Hive metastore connection and that the script executed without errors.  
   - *“File not found”* → `hdfs dfs -ls /user/hive/warehouse/mnaas.db/dim_date` to confirm files exist.  
   - *“SerDe error”* → Ensure the delimiter in source files matches `field.delim=';'`.  

5. **Logging**  
   - Hive logs are written to `$HIVE_LOG_DIR` (often `/tmp/hive.log`).  
   - For Impala, check `impalad` logs for catalog refresh messages.  

---

### 7. External Configuration & Environment Variables  

| Config / Env | Usage |
|--------------|-------|
| `HIVE_CONF_DIR` | Points to Hive configuration (metastore URI, Kerberos settings). |
| `HADOOP_CONF_DIR` | Required for HDFS client access (`hdfs://NN-HA1/...`). |
| `NN-HA1` | NameNode HA address – must resolve via DNS or `/etc/hosts`. |
| `IMPALA_CATALOG_SERVICE_ID` (implicit via `impala.events.catalogServiceId`) | Used by Impala to map the table to its catalog; not set manually but propagated from Hive. |
| **Optional**: `DIM_DATE_SOURCE_PATH` (if the script is templated) | Could be used by a wrapper script to inject a different HDFS location for test environments. |

If the production environment uses a templating engine (e.g., Jinja, Velocity) to replace placeholders, verify the final rendered HQL contains the correct `LOCATION` string.

---

### 8. Suggested Improvements (TODO)

1. **Add Partitioning by Year** – Even though the dimension is small, partitioning (`PARTITIONED BY (year INT)`) would allow Impala to prune data when joining on year, reducing I/O for large fact tables.  
2. **Introduce a Validation Step** – Create a small Hive/Impala script that reads a sample of the source file, checks column count and delimiter, and fails the pipeline early if the format deviates.  

--- 

*End of documentation.*