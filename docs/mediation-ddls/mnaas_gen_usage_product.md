**File:** `mediation-ddls\mnaas_gen_usage_product.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates the **external table** `gen_usage_product` in the `mnaas` database. The table stores the “usage‑product” reference data that downstream mediation, billing, and reporting jobs consume. Each row describes a product (e.g., voice, data, bundle) together with its commercial, technical, and billing attributes. Because the table is *external* and marked with `external.purge='true'`, Hive will automatically delete the underlying HDFS files if the table is dropped, ensuring no orphan data remains.

---

## 2. Important Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `gen_usage_product` | External Hive table | Holds static‑ish product metadata used by usage‑aggregation, billing, and reporting pipelines. |
| Columns (`product_code`, `product_name`, `secs_code`, … `bill_mode`) | Table fields | Provide a normalized view of product characteristics (identifiers, UOM, bundle composition, country profile, plan dates, etc.). |
| `ROW FORMAT SERDE` | Hive SerDe definition | Parses pipe‑delimited (`|`) text files stored in HDFS. |
| `LOCATION 'hdfs://NN-HA1/.../gen_usage_product'` | Table storage location | Physical directory where source files are read from / written to. |
| Table properties (`external.purge`, `bucketing_version`, `impala.*`) | Metadata | Control lifecycle, compatibility with Impala, and statistics generation. |

---

## 3. Interfaces  

| Aspect | Description |
|--------|-------------|
| **Inputs** | Text files (pipe‑delimited) placed in the HDFS directory `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/gen_usage_product`. Files are typically produced by upstream “product generation” jobs (e.g., `mnaas_gen_sim_product.hql`, `mnaas_gen_sim_product_split.hql`). |
| **Outputs** | The logical Hive/Impala table `mnaas.gen_usage_product`. Downstream scripts query this table to enrich CDRs, calculate bundles, or generate invoices. |
| **Side Effects** | - Registers the external table in the Hive metastore.<br>- Sets `external.purge='true'` → dropping the table removes the HDFS directory.<br>- Updates table statistics (`STATS_GENERATED='TASK'`). |
| **Assumptions** | - Hive/Impala services are running and can access the HDFS namenode `NN-HA1`.<br>- The HDFS path exists and is writable by the Hive user.<br>- Source files strictly follow the defined column order and use `|` as delimiter.<br>- No partitioning is required for this reference data set. |

---

## 4. Integration Points  

| Connected Component | How it interacts |
|---------------------|------------------|
| **`mnaas_gen_sim_product.hql` / `mnaas_gen_sim_product_split.hql`** | These scripts generate the raw product definition files that land in the same HDFS directory; they are the primary producers of the data consumed by `gen_usage_product`. |
| **Usage‑aggregation jobs** (`mnaas_billing_traffic_usage.hql`, `mnaas_billing_traffic_actusage.hql`, etc.) | Join `gen_usage_product` on `product_code` / `secs_code` to enrich CDRs with bundle and billing attributes before aggregation. |
| **Billing & invoicing pipelines** (`mnaas_cbf_gnv_invoice_generation.hql`) | Pull `bill_mode`, `uom`, and bundle fields to calculate charges per product. |
| **Reporting / analytics** (e.g., `mnaas_fiveg_nsa_count_report.hql`) | Use country profile IDs (`cntry_prof_id`) and plan dates to filter/report on product usage. |
| **Impala** | The table is visible to Impala (see `impala.*` properties) and can be queried directly by downstream analytical jobs. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – upstream producer adds/removes columns or changes order. | Queries downstream may fail or return wrong data. | Enforce schema versioning; add a validation step (e.g., `SHOW CREATE TABLE` diff) before loading new files. |
| **Incorrect delimiter / malformed rows** – parsing errors or silent data loss. | Bad rows may be dropped or mis‑interpreted, leading to billing errors. | Implement a pre‑load sanity check (e.g., `hdfs dfs -cat … | grep -c '\|'`) and reject files that don’t match expected column count. |
| **Accidental table drop** – `external.purge='true'` will delete underlying files. | Permanent loss of product reference data. | Restrict DROP privileges; use a “soft‑delete” policy (rename table instead of drop) and keep a backup of the HDFS directory. |
| **Permission / ownership mismatch** – Hive user cannot read/write the HDFS path. | Job failures at runtime. | Verify HDFS ACLs (`hdfs dfs -ls`) and ensure Hive user (`hive` or service account) has `rwx` on the directory. |
| **Stale data** – files are not refreshed after product changes. | Billing uses outdated product attributes. | Schedule a daily refresh job that truncates and reloads the table from the latest source files. |

---

## 6. Running / Debugging the Script  

1. **Execute** (typical production deployment)  
   ```bash
   hive -f mediation-ddls/mnaas_gen_usage_product.hql
   # or, for Impala:
   impala-shell -i <impala-host> -f mediation-ddls/mnaas_gen_usage_product.hql
   ```

2. **Verify creation**  
   ```sql
   SHOW CREATE TABLE mnaas.gen_usage_product;
   DESCRIBE FORMATTED mnaas.gen_usage_product;
   ```

3. **Inspect sample data** (ensure delimiter and column order)  
   ```bash
   hdfs dfs -cat /user/hive/warehouse/mnaas.db/gen_usage_product/part-00000 | head -n 5
   ```

4. **Check statistics** (helps downstream query planners)  
   ```sql
   ANALYZE TABLE mnaas.gen_usage_product COMPUTE STATISTICS;
   ```

5. **Debugging tips**  
   - If the table already exists, run `DROP TABLE IF EXISTS mnaas.gen_usage_product;` before re‑creating (be aware of purge).  
   - Look at Hive Metastore logs (`/var/log/hive/hive-metastore.log`) for permission or format errors.  
   - Use `hdfs dfs -test -e <path>` to confirm the HDFS directory exists before running the script.  

---

## 7. External Configuration / Environment Variables  

| Variable / Config | Usage |
|-------------------|-------|
| `NN-HA1` (namenode host) | Hard‑coded in the `LOCATION` URI; may be overridden by a cluster‑wide DNS alias or environment variable in the deployment wrapper. |
| Hive/Impala connection settings (e.g., `hive.metastore.uris`, `impala.service.host`) | Required for the script to reach the metastore and register the table. |
| `HIVE_CONF_DIR` / `IMPALA_CONF_DIR` | Determines which Hive/Impala configuration (including security, Kerberos) is used when the script runs. |
| Optional **property files** (e.g., `hdfs-site.xml`, `core-site.xml`) | Provide HDFS client settings; the script itself does not reference them directly but they must be present on the execution node. |

If the production environment injects these values via a wrapper (e.g., Airflow, Oozie, or a custom scheduler), ensure the wrapper passes the correct HDFS namenode address and Hive/Impala credentials.

---

## 8. Suggested Improvements (TODO)

1. **Add Partitioning** – If the product list grows or is refreshed frequently, partition by `plan_effective_date` (or a static `load_date`) to enable faster metadata operations and incremental loads.  
2. **Schema Validation Hook** – Introduce a pre‑load validation step (e.g., a small Spark/MapReduce job) that checks column count and delimiter consistency before the table is created or refreshed, reducing the risk of malformed data entering downstream billing pipelines.  

---