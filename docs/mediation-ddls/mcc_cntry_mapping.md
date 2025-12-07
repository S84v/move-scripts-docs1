**File:** `mediation-ddls\mcc_cntry_mapping.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) an **external Hive table** `mnaas.mcc_cntry_mapping` that stores a static mapping between Mobile Country Code (`mcc`), ISO country code (`cntry_code`), and human‑readable country name (`cntry_name`). The table points to a pre‑populated directory on HDFS, allowing downstream mediation, reporting, and analytics jobs to enrich call‑detail records (CDRs) with geographic information without copying data into the Hive warehouse.

---

## 2. Key Artifacts & Responsibilities  

| Artifact | Type | Responsibility |
|----------|------|----------------|
| `CREATE EXTERNAL TABLE mnaas.mcc_cntry_mapping` | DDL statement | Defines the schema and location of the MCC‑to‑country mapping data. |
| Columns `mcc`, `cntry_code`, `cntry_name` | Table fields | Holds the three‑field mapping used for enrichment. |
| `ROW FORMAT SERDE … LazySimpleSerDe` | SerDe definition | Parses plain‑text CSV‑style files (default field delimiter = `\001`). |
| `STORED AS INPUTFORMAT … TextInputFormat` / `OUTPUTFORMAT … HiveIgnoreKeyTextOutputFormat` | Storage format | Indicates the data is plain text; no compression or ORC/Parquet. |
| `LOCATION 'hdfs://NN-HA1/.../mcc_cntry_mapping'` | HDFS path | Physical location of the source files; external table does **not** own the data. |
| `TBLPROPERTIES` (e.g., `external.table.purge='true'`) | Table metadata | Controls purge behavior, stats, and Impala catalog integration. |

No procedural code (functions, procedures) exists in this file; it is a pure DDL definition.

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - The HDFS directory `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/mcc_cntry_mapping` must contain one or more text files matching the table schema (e.g., `mcc,cntry_code,cntry_name`). |
| **Outputs** | - Hive metastore entry for `mnaas.mcc_cntry_mapping`. <br> - No data files are created or modified by the DDL itself (external table). |
| **Side Effects** | - If the table already exists, Hive will error unless `IF NOT EXISTS` is added (currently not present). <br> - `external.table.purge='true'` means that dropping the table will also delete the underlying HDFS files – a destructive side effect. |
| **Assumptions** | - HDFS namenode alias `NN-HA1` resolves in the execution environment. <br> - The Hive/Impala services have read (and possibly write) permission on the target directory. <br> - Data files are UTF‑8 encoded, delimiter‑consistent, and match the declared column order. |

---

## 4. Integration with Other Scripts / Components  

| Connected Component | Interaction |
|---------------------|-------------|
| **Mediation ETL jobs** (e.g., CDR ingestion pipelines) | Join on `mcc` to add `cntry_code`/`cntry_name` to fact tables (`fact_cdr`, `dim_date`, etc.). |
| **Reporting scripts** (`market_management_report_sims.hql`, `gbs_curr_avgconv_rate.hql`, etc.) | Reference this table for geographic breakdowns in KPI calculations. |
| **Impala queries** | The table is visible to Impala via the catalog properties; analysts may query it directly for ad‑hoc analysis. |
| **Data loading process** | A separate upstream job (likely a Bash/Python/Airflow task) populates the HDFS directory with the latest MCC mapping CSV before this DDL is executed. |
| **Metadata management** | The Hive metastore stores the table definition; any schema change must be coordinated with downstream jobs that expect the three‑column layout. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Accidental data loss** – `external.table.purge='true'` causes underlying files to be deleted when the table is dropped. | Permanent loss of the mapping file. | Use `DROP TABLE IF EXISTS` only after confirming a backup exists; consider removing the purge property for a safety net. |
| **Schema drift** – Adding/removing columns without updating downstream jobs. | Query failures or silent data mismatches. | Enforce schema versioning; add a CI check that validates downstream SQL against the table definition. |
| **Permission mismatch** – Hive/Impala user lacks read access to the HDFS path. | Table creation succeeds but queries fail at runtime. | Verify HDFS ACLs (`hdfs dfs -ls`) and grant `READ` to the service account before deployment. |
| **Stale data** – Mapping file not refreshed after a telecom‑MCC change. | Incorrect country attribution in reports. | Schedule a periodic (e.g., weekly) data‑load job that overwrites the HDFS directory and run `MSCK REPAIR TABLE` or `REFRESH` in Impala. |
| **NameNode alias change** – `NN-HA1` may be renamed or re‑addressed. | Table creation fails with “unknown host”. | Externalize the namenode URI into a config property (e.g., `${HDFS_NN_URI}`) and reference it via Hive variables. |

---

## 6. Running / Debugging the Script  

1. **Prerequisites**  
   - Hive/Beeline client installed and configured to point at the correct metastore.  
   - Access to the HDFS path (`hdfs://NN-HA1/.../mcc_cntry_mapping`).  

2. **Execution**  
   ```bash
   hive -f mediation-ddls/mcc_cntry_mapping.hql
   # or, using Beeline:
   beeline -u jdbc:hive2://<host>:10000/default -f mediation-ddls/mcc_cntry_mapping.hql
   ```

3. **Verification**  
   ```sql
   SHOW CREATE TABLE mnaas.mcc_cntry_mapping;
   SELECT COUNT(*) FROM mnaas.mcc_cntry_mapping LIMIT 1;
   DESCRIBE FORMATTED mnaas.mcc_cntry_mapping;
   ```

4. **Debugging Tips**  
   - If Hive reports “Table already exists”, add `IF NOT EXISTS` to the DDL.  
   - Permission errors: run `hdfs dfs -ls <path>` as the Hive user to confirm access.  
   - Data format issues: query a few rows and check column alignment; adjust SerDe or delimiter if needed.  

---

## 7. External Configurations & Environment Variables  

| Config / Variable | Usage |
|-------------------|-------|
| `NN-HA1` (namenode host) | Hard‑coded in the `LOCATION` URI; should be externalized to a property file or Hive variable (`${hdfs.namenode}`) for portability. |
| Hive Metastore connection settings (`hive.metastore.uris`) | Determines where the table definition is stored. |
| Impala catalog service IDs (`impala.events.catalogServiceId`, `catalogVersion`) | Auto‑generated; not manually edited but may be referenced by monitoring tools. |
| Optional: `HDFS_USER` / Kerberos principal | If the cluster uses security, the script must be run under a principal with read/write rights on the target directory. |

---

## 8. Suggested Improvements (TODO)

1. **Add Idempotency** – Modify the DDL to `CREATE EXTERNAL TABLE IF NOT EXISTS` and include a `DROP TABLE IF EXISTS` guard with a safety check to avoid accidental data purge.
2. **Externalize HDFS URI** – Replace the hard‑coded `hdfs://NN-HA1/...` with a Hive variable (e.g., `${mcc_mapping_path}`) sourced from a central configuration file, enabling easier environment promotion (dev → prod).  

---