**File:** `mediation-ddls\mnaas_vw_tcl_asset_pkg_mapping_billing.hql`  
**Type:** HiveQL DDL (expected to define a view) – **currently empty**  

---

## 1. High‑Level Summary
This script is intended to create the Hive view **`mnaas_vw_tcl_asset_pkg_mapping_billing`**, which would expose a normalized mapping between TCL‑managed network assets (e.g., routers, switches, virtual functions) and the billing‑relevant product packages they belong to. In production the view is consumed by downstream traffic‑detail and billing‑generation scripts (e.g., `mnaas_v_traffic_details_*` and `mnaas_v_ictraffic_details_*`) to enrich CDRs with asset‑level pricing information.

*At present the file contains no DDL; therefore the view does not exist in the metastore, and any downstream job that references it will fail with “Table not found”.*  

---

## 2. Expected Objects & Responsibilities  

| Object | Responsibility (expected) |
|--------|----------------------------|
| **View `mnaas_vw_tcl_asset_pkg_mapping_billing`** | Joins the raw asset inventory (`mnaas_raw_asset`) with the package catalog (`mnaas_pkg_catalog`) to expose columns such as `asset_id`, `asset_type`, `package_id`, `billing_rate`, `effective_date`, `expiry_date`. |
| **Supporting tables (referenced by the view)** | *`mnaas_raw_asset`* – raw asset dump from the network‑management system.<br>*`mnaas_pkg_catalog`* – master list of billable packages and rates.<br>*`mnaas_dim_time`* – optional time dimension for effective dating. |

*No functions or procedures are defined in this file.*

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Inputs** | - Source tables: `mnaas_raw_asset`, `mnaas_pkg_catalog` (and possibly `mnaas_dim_time`).<br>- Hive configuration: default database, `hive.exec.dynamic.partition` settings if the view uses partitioned tables. |
| **Outputs** | - A **Hive view** (`mnaas_vw_tcl_asset_pkg_mapping_billing`) registered in the metastore. No physical data is written. |
| **Side Effects** | - Metastore metadata change (creation/overwrite of the view). |
| **Assumptions** | - Source tables are refreshed daily by upstream ETL jobs (e.g., `mnaas_traffic_pre_active_raw_sim`).<br>- Column names and data types are stable across releases.<br>- The view is referenced by downstream scripts that expect the columns listed above. |

---

## 4. Integration Points  

| Downstream Component | How it uses the view |
|----------------------|----------------------|
| `mnaas_v_traffic_details_billing.hql` | Joins on `asset_id` to fetch `package_id` and `billing_rate`. |
| `mnaas_v_ictraffic_details_ppu_billing.hql` | Uses the view to resolve per‑unit pricing for on‑demand services. |
| Billing orchestration jobs (e.g., nightly Spark/MapReduce pipelines) | Reads the view as part of the “enrich CDRs” step. |
| Monitoring dashboards | May query the view for asset‑level revenue reporting. |

*If any of the above scripts contain a `FROM mnaas_vw_tcl_asset_pkg_mapping_billing` clause, they will currently error out.*

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing view** – downstream jobs fail with “Table not found”. | Production billing run aborts; revenue leakage. | Deploy a **placeholder view** (e.g., `SELECT NULL AS asset_id … LIMIT 0`) until the real definition is ready; add a health‑check that validates view existence before job start. |
| **Schema drift** – source tables change column names/types. | View creation fails or returns wrong data. | Version‑control the view DDL and include unit tests that compare expected schema against source tables. |
| **Stale asset data** – view reflects outdated asset inventory. | Incorrect billing rates applied. | Ensure upstream asset load (`mnaas_raw_asset`) runs before this view is refreshed; add a timestamp column in the view to verify freshness. |
| **Permission issues** – user executing the script lacks `CREATE VIEW` rights. | Deployment fails silently. | Document required Hive privileges and enforce via role‑based access control. |

---

## 6. Running / Debugging the Script  

1. **Validate Hive environment**  
   ```bash
   hive -e "SHOW DATABASES;"   # confirm you are in the correct DB
   hive -e "DESCRIBE FORMATTED mnaas_raw_asset;"   # sanity‑check source schema
   ```

2. **Execute the script** (once the DDL is added)  
   ```bash
   hive -f mediation-ddls/mnaas_vw_tcl_asset_pkg_mapping_billing.hql
   ```

3. **Verify view creation**  
   ```bash
   hive -e "DESCRIBE EXTENDED mnaas_vw_tcl_asset_pkg_mapping_billing;"
   hive -e "SELECT COUNT(*) FROM mnaas_vw_tcl_asset_pkg_mapping_billing LIMIT 10;"
   ```

4. **Debugging**  
   - If the view fails to compile, capture the Hive error log (`/tmp/hive.log`) and check for missing columns.  
   - Use `EXPLAIN` on a downstream query that references the view to see how Hive resolves it.  
   - If downstream jobs still error, run a minimal query: `SELECT * FROM mnaas_vw_tcl_asset_pkg_mapping_billing LIMIT 5;` to confirm data shape.

---

## 7. External Configuration / Environment Variables  

| Variable / File | Purpose |
|-----------------|---------|
| `HIVE_CONF_DIR` (or `HADOOP_CONF_DIR`) | Points to Hive configuration (metastore URI, authentication). |
| `DB_NAME` (if used in the script) | Target database for view creation; otherwise defaults to the current session DB. |
| `ASSET_TABLE` / `PKG_TABLE` (potential placeholders) | Could be injected via `--hivevar` to allow the same script to run against test vs. prod tables. |

*The current file contains no references; when the DDL is added, verify any `${}` placeholders against the deployment documentation.*

---

## 8. Suggested TODO / Improvements  

1. **Implement the view definition** – derive the required columns from `mnaas_raw_asset` and `mnaas_pkg_catalog`, include effective dating, and add `COMMENT` metadata for each column.  
2. **Add a unit‑test Hive script** that creates mock source tables, runs the view DDL, and asserts the expected schema and row counts. This will catch schema drift early in CI pipelines.

--- 

*If you need the exact column list or business rules for the mapping, consult the data‑model repository (e.g., `data-models/mnaas_asset_pkg.xlsx`) or ask the asset‑management team.*