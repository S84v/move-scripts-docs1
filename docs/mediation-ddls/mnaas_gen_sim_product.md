**File:** `mediation-ddls\mnaas_gen_sim_product.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) an **external Hive table** named `gen_sim_product` that exposes pipe‑delimited SIM‑product reference data stored under the HDFS path `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/gen_sim_product`. The table is used by downstream mediation and billing jobs (e.g., `mnaas_billing_tolling_sims.hql`, `mnaas_billing_traffic_usage.hql`) to enrich CDRs, calculate tariffs, and generate invoices. Because the table is external and marked with `external.table.purge='true'`, dropping it will also delete the underlying files.

---

## 2. Core Objects Defined

| Object | Type | Responsibility |
|--------|------|-----------------|
| `gen_sim_product` | External Hive table | Provides a read‑only view of SIM‑product master data (product codes, plan dates, pricing model, etc.) to all mediation/billing pipelines. |
| Table columns (`product_name`, `plan_effective_date`, … `bill_mode`) | Schema fields | Capture the attributes required for tariff lookup, provisioning logic, and reporting. All columns are defined as `string` to match the raw source file format. |
| Table properties (`bucketing_version`, `external.table.purge`, `impala.events.*`, `transient_lastDdlTime`) | Metadata | Control storage behavior, enable automatic cleanup, and expose the table to Impala for low‑latency queries. |

*No procedural code (functions, procedures, UDFs) is present in this file.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Input data** | Raw text files located at `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/gen_sim_product`. Each line is pipe (`|`) delimited, matching the column order defined in the DDL. |
| **Output** | Hive/Impala metadata entry `mnaas.gen_sim_product`. Queries against this table return the parsed rows from the underlying files. |
| **Side‑effects** | - Registers the external table in the Hive metastore.<br>- Because of `external.table.purge='true'`, a `DROP TABLE gen_sim_product` will also delete the HDFS directory and its files.<br>- Impala catalog is notified via `impala.events.*` properties. |
| **Assumptions** | - The HDFS path is reachable from all Hive/Impala nodes (network name `NN-HA1` resolves correctly).<br>- Files are UTF‑8 encoded and conform to the pipe‑delimited schema.<br>- No column type conversion is required (all values can be safely stored as strings).<br>- The Hive metastore is configured to allow external tables and the `purge` flag. |
| **External dependencies** | - Hadoop HDFS cluster (namenode `NN-HA1`).<br>- Hive Metastore service.<br>- Impala catalog service (ID `2f97c33b15cd4d7b:926f09b4f64aaefd`).<br>- Potential downstream scripts that reference `gen_sim_product`. |

---

## 4. Integration Points with Other Scripts / Components

| Connected Component | How the Connection Occurs |
|---------------------|----------------------------|
| **Billing / Tolling scripts** (`mnaas_billing_tolling_sims.hql`, `mnaas_billing_traffic_usage.hql`, etc.) | These scripts join `gen_sim_product` with CDR or usage tables to resolve product‑specific pricing rules (`pricing_based_on`, `usage_model`, `bill_mode`). |
| **ETL pipelines** (e.g., nightly data‑load jobs) | A separate ingestion job (not shown) copies the latest SIM‑product CSV into the HDFS location before this DDL is executed, ensuring the table reflects the most recent catalog. |
| **Impala query layer** | Impala queries reference the same table for ad‑hoc reporting; the `impala.events.*` properties keep the Impala catalog in sync with Hive DDL changes. |
| **Data governance / purge processes** | Because the table is external with purge enabled, any automated cleanup that drops the table will also remove the source files, so coordination with data retention policies is required. |
| **Configuration files** | The HDFS URI (`hdfs://NN-HA1/...`) may be templated via an environment variable (e.g., `HDFS_NN`) in the deployment pipeline; verify the CI/CD script that renders this DDL. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Schema drift** – source file adds/removes columns or changes order. | Queries downstream may fail or return incorrect data. | Enforce a validation step in the ingestion job (e.g., schema‑check script) before the DDL is applied. |
| **Missing or corrupted source files** – HDFS directory empty or contains malformed rows. | Downstream billing jobs produce zero or erroneous charges. | Add a health‑check that counts files/records and validates delimiters before the table is used. |
| **Accidental table drop** – `external.table.purge='true'` will delete source files. | Permanent loss of reference data. | Restrict DROP privileges to a limited admin role and require a two‑step approval (e.g., JIRA ticket) for any table removal. |
| **Permission issues** – Hive/Impala users lack read access to the HDFS path. | Job failures at runtime. | Ensure ACLs on the directory grant `READ` to the service accounts used by Hive/Impala. |
| **Network name resolution** – `NN-HA1` not resolvable in some environments (e.g., test vs prod). | Table creation fails with “cannot locate namenode”. | Parameterize the namenode host via an environment variable and validate it during CI pipeline execution. |

---

## 6. Running / Debugging the Script

1. **Typical execution** (as part of a deployment pipeline)  
   ```bash
   hive -f mediation-ddls/mnaas_gen_sim_product.hql
   # or via Beeline:
   beeline -u jdbc:hive2://<hive-host>:10000/default -f mediation-ddls/mnaas_gen_sim_product.hql
   ```

2. **Verification steps**  
   - After execution, run `SHOW CREATE TABLE gen_sim_product;` to confirm the DDL.  
   - Query a few rows: `SELECT * FROM gen_sim_product LIMIT 10;` – verify column alignment and delimiter handling.  
   - Check HDFS location: `hdfs dfs -ls hdfs://NN-HA1/user/hive/warehouse/mnaas.db/gen_sim_product` – ensure files exist and have appropriate permissions.

3. **Debugging common issues**  
   - **“Table already exists”** – use `CREATE EXTERNAL TABLE IF NOT EXISTS` or drop/recreate with `DROP TABLE gen_sim_product PURGE;`.  
   - **Malformed rows** – inspect raw files with `hdfs dfs -cat ... | head -n 20` and confirm pipe delimiters.  
   - **Permission errors** – verify the user running Hive/Beeline has `READ` on the HDFS directory and `CREATE` on the metastore.

---

## 7. External Configurations / Environment Variables

| Config / Variable | Purpose |
|-------------------|---------|
| `NN-HA1` (namenode host) | Hard‑coded in the `LOCATION` clause; may be overridden by a CI/CD templating step (e.g., `${HDFS_NN}`). |
| Hive Metastore connection settings (e.g., `hive.metastore.uris`) | Required for the DDL to be persisted. |
| Impala catalog service ID (`2f97c33b15cd4d7b:926f09b4f64aaefd`) | Auto‑populated by the Hive‑Impala integration; not typically edited manually. |
| Optional: `GEN_SIM_PRODUCT_PATH` env var used by upstream ingestion jobs to place files into the correct HDFS directory. | Not referenced directly in this DDL but part of the overall data‑move flow. |

---

## 8. Suggested Improvements (TODO)

1. **Add explicit column types** – Convert date fields (`plan_effective_date`, `plan_termination_date`) to `DATE` and numeric codes (`product_code`, `secs_code`) to `INT`/`BIGINT` to enable proper validation and query optimization.  
2. **Introduce partitioning** – If the dataset grows (e.g., by month or product version), partition the table (e.g., `PARTITIONED BY (load_date STRING)`) to improve query performance and simplify incremental loads.  

---