**File:** `mediation-ddls\gbs_curr_avgconv_rate.hql`  

---

### 1. High‑Level Summary
This Hive DDL script creates an **external** table `mnaas.gbs_curr_avgconv_rate` that stores monthly average currency‑conversion rates (`cal_month`, `from_curr`, `to_curr`, `avg_conv_rate`). The table is defined over a CSV‑style text file located in HDFS under the `mnaas.db` warehouse directory. Because the table is external and marked with `external.table.purge='true'`, Hive will not retain data after a DROP, and the underlying files are managed outside Hive (e.g., by upstream data‑move jobs that generate the CSV files). Down‑stream analytics (Impala, Hive, BI tools) query this table to obtain historical conversion rates.

---

### 2. Key Objects & Responsibilities
| Object | Type | Responsibility |
|--------|------|----------------|
| `gbs_curr_avgconv_rate` | Hive external table | Holds monthly average conversion rates; schema is `cal_month STRING, from_curr STRING, to_curr STRING, avg_conv_rate DOUBLE`. |
| `createtab_stmt` | DDL statement (named block) | Encapsulates the `CREATE EXTERNAL TABLE` command; used by orchestration tools that may parse the file for a specific block name. |
| `ROW FORMAT SERDE` | Hive SerDe definition | Parses CSV rows using `LazySimpleSerDe` with comma delimiter. |
| `LOCATION` | HDFS path | Physical storage location for the CSV files (`hdfs://NN-HA1/user/hive/warehouse/mnaas.db/gbs_curr_avgconv_rate`). |
| `TBLPROPERTIES` | Table metadata | Stores statistics, purge flag, Impala catalog IDs, and audit timestamps. |

*No procedural code (functions, procedures) is present; the file is pure DDL.*

---

### 3. Inputs, Outputs, Side Effects & Assumptions
| Category | Details |
|----------|---------|
| **Inputs** | - The script itself (DDL).<br>- Implicit: HDFS namenode `NN-HA1` must be reachable.<br>- Implicit: Hive metastore must be reachable and allow creation of external tables in database `mnaas`. |
| **Outputs** | - A new entry in the Hive metastore for `mnaas.gbs_curr_avgconv_rate`.<br>- No data is written; the table points to existing files (or an empty directory) at the specified HDFS location. |
| **Side Effects** | - Alters Hive metastore state (adds table metadata).<br>- If the location already contains files, they become instantly queryable.<br>- Because `external.table.purge='true'`, a later `DROP TABLE` will delete the underlying files. |
| **Assumptions** | - The `mnaas` database already exists.<br>- The HDFS directory exists and is writable by the Hive/Impala service user (often `hive` or `impala`).<br>- The CSV files placed there conform to the defined schema (comma‑delimited, newline‑terminated).<br>- Impala catalog IDs (`impala.events.catalogServiceId`, `catalogVersion`) are valid for the current Impala cluster. |

---

### 4. Integration Points & Connectivity
| Connected Component | How This File Relates |
|---------------------|-----------------------|
| **Upstream data‑move jobs** (e.g., Spark/MapReduce jobs that compute average conversion rates) | They write CSV files into the HDFS location defined by `LOCATION`. The DDL makes those files visible to Hive/Impala. |
| **Downstream analytics** (Impala queries, reporting dashboards) | Consume the table directly (`SELECT * FROM mnaas.gbs_curr_avgconv_rate`). The `impala.events.*` properties ensure the table appears in Impala’s catalog. |
| **Orchestration layer** (Airflow, Oozie, custom scheduler) | Executes this `.hql` as a step in a pipeline, often after the upstream job finishes. The block name `createtab_stmt` may be referenced for selective execution. |
| **Metadata lineage tools** (e.g., Apache Atlas) | Register the table and its properties; the `last_modified_by` and timestamps help lineage tracking. |
| **Related DDL files** (e.g., `dim_date.hql` from history) | `dim_date` provides a date dimension that may be joined with `gbs_curr_avgconv_rate` on `cal_month`. Both tables reside in the same `mnaas` database, enabling consistent reporting. |

---

### 5. Operational Risks & Mitigations
| Risk | Impact | Mitigation |
|------|--------|------------|
| **Location path change** (namenode rename, HDFS re‑layout) | Table becomes inaccessible; downstream jobs fail. | Parameterize the HDFS URI via a config file or environment variable; validate path existence during deployment. |
| **Schema drift** (upstream job adds a column or changes delimiter) | Queries break; data may be mis‑parsed. | Add a schema validation step after data generation; enforce strict CSV format; version the table (e.g., `gbs_curr_avgconv_rate_v2`). |
| **Permission issues** (Hive/Impala user cannot read/write the directory) | Table creation fails or data is invisible. | Ensure directory ACLs grant `rwx` to the service user; include a pre‑check script. |
| **External table purge** (`external.table.purge='true'`) | Accidental `DROP TABLE` removes raw files, causing data loss. | Restrict DROP privileges; add a safeguard script that checks for a “protected” flag before dropping. |
| **Impala catalog mismatch** (stale `catalogServiceId`/`catalogVersion`) | Impala may not see the table or may serve stale metadata. | Refresh Impala catalog after DDL (`INVALIDATE METADATA` or `REFRESH`); automate this in the pipeline. |

---

### 6. Running / Debugging the Script
1. **Execution**  
   ```bash
   # Using Hive CLI
   hive -f mediation-ddls/gbs_curr_avgconv_rate.hql

   # Or using Beeline (recommended)
   beeline -u jdbc:hive2://<hive-host>:10000/default -f mediation-ddls/gbs_curr_avgconv_rate.hql
   ```
   *If the orchestration framework supports block execution, reference the block name:*
   ```bash
   beeline -u ... -f mediation-ddls/gbs_curr_avgconv_rate.hql -e "createtab_stmt"
   ```

2. **Verification**  
   ```sql
   SHOW CREATE TABLE mnaas.gbs_curr_avgconv_rate;
   DESCRIBE FORMATTED mnaas.gbs_curr_avgconv_rate;
   SELECT COUNT(*) FROM mnaas.gbs_curr_avgconv_rate LIMIT 10;
   ```

3. **Debugging Tips**  
   - **Metastore errors**: Check Hive metastore logs (`/var/log/hive/hive-metastore.log`).  
   - **HDFS path**: `hdfs dfs -ls /user/hive/warehouse/mnaas.db/gbs_curr_avgconv_rate` to confirm directory exists and contains files.  
   - **Permissions**: `hdfs dfs -chmod -R 775 ...` and `hdfs dfs -chown -R hive:hive ...` if needed.  
   - **Impala visibility**: In Impala shell, run `INVALIDATE METADATA mnaas.gbs_curr_avgconv_rate;` then query.  

---

### 7. External Configuration & Environment Variables
| Config Item | Expected Source | Usage |
|------------|----------------|-------|
| Hive metastore URL / JDBC connection | `HIVE_JDBC_URL` env var or `hive-site.xml` | Determines where the DDL is applied. |
| HDFS namenode address (`NN-HA1`) | Could be templated via `${NAMENODE_HOST}` | Allows the same script to run in different clusters. |
| Database name (`mnaas`) | May be overridden by `${TARGET_DB}` | Enables reuse across environments (dev, test, prod). |
| Table location base path | `${WAREHOUSE_ROOT}` | Centralizes path management. |

*The current script hard‑codes these values; a production pipeline typically injects them via a templating engine (e.g., Jinja, Velocity) or environment substitution.*

---

### 8. Suggested Improvements (TODO)
1. **Parameterize critical values** – Refactor the script to use placeholders for the database name, HDFS namenode, and location path, then render via the orchestration tool. This reduces the risk of environment‑specific hard‑coding.
2. **Add `IF NOT EXISTS` guard** – Modify the DDL to `CREATE EXTERNAL TABLE IF NOT EXISTS ...` to make the script idempotent and safe for re‑runs in CI/CD pipelines.  

---