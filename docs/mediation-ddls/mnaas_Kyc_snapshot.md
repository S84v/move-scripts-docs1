**File:** `mediation-ddls\mnaas_Kyc_snapshot.hql`  

---

## 1. High‑Level Summary
This script creates an **external Hive table** named `kyc_snapshot` in the `mnaas` database. The table maps a CSV‑formatted KYC (Know‑Your‑Customer) data dump stored under a fixed HDFS location to a set of string and timestamp columns. Because it is external and has `external.table.purge='true'`, Hive only manages the metadata; the underlying files remain on HDFS and are automatically removed if the table is dropped.

---

## 2. Core Statement(s) & Responsibilities  

| Element | Responsibility |
|---------|-----------------|
| `CREATE EXTERNAL TABLE mnaas.kyc_snapshot (…)` | Registers a new external table in the Hive metastore with the specified column list. |
| Column definitions (`iccid`, `vin`, `sim_date`, …, `lastupdatedate`) | Define the schema expected from the CSV source. |
| `ROW FORMAT SERDE … LazySimpleSerDe` + `field.delim=','` | Instruct Hive to parse each line as a comma‑separated record. |
| `STORED AS INPUTFORMAT … TextInputFormat` / `OUTPUTFORMAT … HiveIgnoreKeyTextOutputFormat` | Use plain text files as the storage format (no compression, no Hive‑specific file layout). |
| `LOCATION 'hdfs://NN-HA1/.../kyc_snapshot'` | Points the table to the physical HDFS directory that holds the source CSV files. |
| `TBLPROPERTIES (…)` | Provides metadata for Impala event tracking, statistics generation, and purge behavior. |

*No procedural code, functions, or classes are present – the file is a DDL definition.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - HDFS directory `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/kyc_snapshot` containing CSV files.<br>- Implicit Hive/Impala configuration (metastore connection, namenode address). |
| **Outputs** | - Hive metastore entry for `mnaas.kyc_snapshot` (schema, location, properties).<br>- No data transformation; the raw files become queryable via Hive/Impala. |
| **Side‑Effects** | - Registers a new external table (metadata only).<br>- Because `external.table.purge='true'`, dropping the table will **delete** the underlying HDFS files. |
| **Assumptions** | - The HDFS path exists and is readable by the Hive service user.<br>- Files are UTF‑8 encoded, comma‑delimited, and match the column order/types.<br>- `lastupdatedate` values are parseable as Hive `timestamp` (default format `yyyy‑MM‑dd HH:mm:ss`).<br>- No partitioning is required for current workloads. |

---

## 4. Interaction with Other Scripts / Components  

| Connected Component | Relationship |
|---------------------|--------------|
| **Down‑stream ETL jobs** (e.g., `mnaa_Kyc_mapping_snapshot.hql`, `gbs_curr_avgconv_rate.hql`) | Likely read from `mnaas.kyc_snapshot` to enrich or aggregate KYC data for reporting or billing. |
| **Impala** | The table properties (`impala.events.*`) indicate that Impala queries this table; any Impala‑based analytics will reference it. |
| **Data Ingestion Pipelines** (outside this repo) | A separate process (e.g., a nightly SFTP or Kafka consumer) deposits CSV files into the HDFS location before this DDL is executed. |
| **Hive Metastore** | The table definition is stored here; any script that uses `SHOW TABLES` or `DESCRIBE` will see it. |
| **Monitoring / Auditing** | The `last_modified_by` and `last_modified_time` properties are used by ops tools to track changes. |

*Because the script only creates the table, the actual data load is performed elsewhere.*

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – source CSV column order changes. | Queries fail or return wrong data. | Add a validation step (e.g., a Hive `SELECT COUNT(*) FROM ... LIMIT 1`) that checks column count/types before production use. |
| **Accidental data loss** – `external.table.purge='true'` deletes files on DROP. | Permanent loss of raw KYC records. | Restrict DROP privileges; implement a backup of the HDFS directory before any schema change. |
| **Permission issues** – Hive user cannot read/write the HDFS path. | Table creation fails or queries return empty results. | Verify HDFS ACLs for the Hive service principal; include a pre‑flight `hdfs dfs -test -d` check in deployment scripts. |
| **Timestamp parsing errors** – malformed `lastupdatedate`. | Rows become NULL or cause query failures. | Enforce a strict input format upstream; optionally add a staging table with string column and a conversion UDF. |
| **Unpartitioned large dataset** – full table scans become expensive. | Performance degradation for downstream analytics. | Consider adding partitioning (e.g., by `sim_date`) in a future iteration. |

---

## 6. Running / Debugging the Script  

1. **Execution**  
   ```bash
   # Using Hive CLI
   hive -f mediation-ddls/mnaas_Kyc_snapshot.hql

   # Or using Beeline (preferred)
   beeline -u jdbc:hive2://<hive-host>:10000/default -f mediation-ddls/mnaas_Kyc_snapshot.hql
   ```

2. **Verification**  
   ```sql
   SHOW CREATE TABLE mnaas.kyc_snapshot;
   DESCRIBE FORMATTED mnaas.kyc_snapshot;
   SELECT COUNT(*) FROM mnaas.kyc_snapshot LIMIT 10;
   ```

3. **Debugging Tips**  
   - If the table creation fails, check the Hive metastore logs for permission or path errors.  
   - Use `hdfs dfs -ls <location>` to confirm that the CSV files exist and are accessible.  
   - Run a sample query with `LIMIT` to ensure the `timestamp` column parses correctly.  
   - Inspect `TBLPROPERTIES` via `SHOW TBLPROPERTIES mnaas.kyc_snapshot;` to confirm Impala event IDs are present.

---

## 7. External Configuration / Environment Dependencies  

| Item | Usage |
|------|-------|
| `NN-HA1` (namenode host) | Hard‑coded in the `LOCATION` URI; must resolve in the cluster DNS. |
| Hive/Impala service accounts | Must have read/write access to the HDFS path and permission to create external tables in the `mnaas` database. |
| Hive Metastore connection settings (e.g., `hive.metastore.uris`) | Required for the `CREATE EXTERNAL TABLE` command to succeed. |
| No explicit environment variables are referenced inside the file; any dynamic values (e.g., namenode host) are expected to be baked into the script by the deployment pipeline. |

---

## 8. Suggested Improvements (TODO)

1. **Add Partitioning** – Introduce a `PARTITIONED BY (sim_date STRING)` clause and a subsequent `MSCK REPAIR TABLE` step to improve query performance on date‑range scans.  
2. **Document Column Semantics** – Extend the DDL with `COMMENT` clauses for each column (e.g., `iccid STRING COMMENT 'Integrated Circuit Card Identifier'`) to aid downstream developers and data catalog tools.  

---