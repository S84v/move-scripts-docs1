**File:** `mediation-ddls\mnaas_mmr_sponsor_master.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates (or re‑creates) the external table **`mnaas.mmr_sponsor_master`**. The table maps sponsor‑related metadata (currency, supplier name/prefix, TADIG code, file timestamps, and source filename) to raw CSV‑style files stored under an HDFS location. In production the table is used by downstream mediation jobs (e.g., invoice generation, usage aggregation, inter‑connect reporting) to enrich transaction records with sponsor information.

---

## 2. Core Artifact(s)

| Artifact | Type | Responsibility |
|----------|------|-----------------|
| `createtab_stmt` | Hive DDL statement | Defines the external table schema, serialization format, storage location, and table properties. |
| `mmr_sponsor_master` | Hive external table | Provides read‑only access to sponsor master data files located in HDFS. |

*No procedural code (functions, classes) exists in this file; the entire file is a single DDL statement.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - Raw sponsor master files (semicolon‑delimited) placed in `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/mmr_sponsor_master`.<br>- Hive metastore (for table metadata). |
| **Outputs** | - Table metadata registered in the Hive metastore under database `mnaas`.<br>- No data movement; the table points directly to the external files. |
| **Side Effects** | - If the table already exists, Hive will **replace** it (default behavior without `IF NOT EXISTS`).<br>- Permissions on the HDFS directory affect queryability. |
| **Assumptions** | - Files are UTF‑8 text, each record ends with `\n` and fields are separated by `;`.<br>- Timestamp columns (`fileupdateddatetime`, `lastinsrttime`) are in a format Hive can parse (e.g., `yyyy-MM-dd HH:mm:ss`).<br>- The HDFS namenode alias `NN-HA1` resolves correctly in the execution environment. |

---

## 4. Integration with Other Scripts / Components  

| Connected Component | How It Links |
|---------------------|--------------|
| **Downstream mediation jobs** (e.g., `mnaas_cbf_gnv_invoice_generation.hql`, `mnaas_gen_usage_product.hql`) | These scripts join transaction tables with `mmr_sponsor_master` on `supplier_name` / `supplier_prefix` / `tadig` to enrich billing records. |
| **Data ingestion pipelines** (ETL jobs that land sponsor files) | External processes (often a shell or Spark job) copy/append CSV files into the HDFS location defined above. The DDL does not manage file creation; it only exposes them to Hive. |
| **Hive Metastore** | The table definition is stored here; any tool that reads Hive metadata (e.g., Hue, Spark SQL, Presto) can query the table. |
| **Operational monitoring** | Alerting scripts may query `SHOW TABLES LIKE 'mmr_sponsor_master'` or `DESCRIBE FORMATTED` to verify existence and location. |
| **Security / ACL** | HDFS ACLs and Hive Ranger policies must grant read access to the `mmr_sponsor_master` directory for the service accounts that run downstream jobs. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Table recreation overwrites metadata** (e.g., loss of comments, partitions) | Downstream jobs may break if they rely on table properties that disappear. | Add `IF NOT EXISTS` and separate `ALTER TABLE` statements for schema changes. |
| **Stale or malformed source files** (wrong delimiter, bad timestamps) | Queries return errors or incorrect sponsor data. | Implement a validation step in the ingestion pipeline (e.g., Spark job that checks schema before dropping files). |
| **HDFS path changes / namenode alias mismatch** | Table becomes unreadable, causing job failures. | Externalize the HDFS URI into a config property (e.g., `${MMR_SPONSOR_PATH}`) and version‑control it. |
| **Insufficient permissions** | Hive queries fail with “permission denied”. | Document required HDFS and Ranger permissions; include a provisioning script. |
| **Schema drift** (new columns added upstream) | Hive table schema out‑of‑sync, causing missing data. | Use a schema‑evolution process: `ALTER TABLE ... ADD COLUMNS` rather than recreating the table. |

---

## 6. Running / Debugging the Script  

1. **Execute**  
   ```bash
   hive -f mediation-ddls/mnaas_mmr_sponsor_master.hql
   ```
   *or* run via an Airflow/ODI task that invokes Hive.

2. **Verify creation**  
   ```sql
   SHOW TABLES LIKE 'mmr_sponsor_master';
   DESCRIBE FORMATTED mnaas.mmr_sponsor_master;
   ```

3. **Sample data check**  
   ```sql
   SELECT * FROM mnaas.mmr_sponsor_master LIMIT 10;
   ```

4. **Debugging tips**  
   - If Hive reports “Table already exists”, add `IF NOT EXISTS` or drop the table first (`DROP TABLE IF EXISTS mnaas.mmr_sponsor_master;`).  
   - Check HDFS path existence: `hdfs dfs -ls /user/hive/warehouse/mnaas.db/mmr_sponsor_master`.  
   - Review Hive logs (`/tmp/hive.log` or YARN application logs) for serialization errors.  

---

## 7. External Configuration / Environment Variables  

| Config Item | Usage |
|------------|-------|
| `NN-HA1` (namenode host alias) | Hard‑coded in the `LOCATION` clause; should be resolvable via DNS or `/etc/hosts`. |
| Hive metastore URI (`hive.metastore.uris`) | Required for the `CREATE EXTERNAL TABLE` command to register metadata. |
| Optional: `MMR_SPONSOR_PATH` (if introduced) | Would replace the hard‑coded HDFS URI, allowing environment‑specific paths (dev, test, prod). |

If the organization uses a central configuration repository (e.g., Ambari, Cloudera Manager, or a custom `.properties` file), the HDFS location should be sourced from there rather than being inlined.

---

## 8. Suggested Improvements (TODO)

1. **Make the DDL idempotent** – prepend `DROP TABLE IF EXISTS mnaas.mmr_sponsor_master;` or use `CREATE EXTERNAL TABLE IF NOT EXISTS`. This prevents accidental metadata loss during repeated deployments.  
2. **Externalize the HDFS location** – replace the hard‑coded URI with a variable (e.g., `${mmr_sponsor_path}`) read from a properties file or environment variable, enabling smoother promotion across environments.  

---