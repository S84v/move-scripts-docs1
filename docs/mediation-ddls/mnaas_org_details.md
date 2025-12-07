**File:** `mediation-ddls\mnaas_org_details.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive **external** table `mnaas.org_details`. The table stores static reference data that maps an organization number (`orgno`) to its full name (`orgname`) and abbreviation (`org_abbr`). The data resides as a plain‑text CSV file in HDFS under the `mnaas.db/org_details` directory. Because the table is external and marked with `external.table.purge='true'`, dropping the table will also delete the underlying files, making it a controlled source of reference data for downstream mediation, billing, and reconciliation jobs (e.g., the MNP, usage‑aggregation, and billing scripts listed in the HISTORY section).

---

## 2. Key Artifacts & Responsibilities  

| Artifact | Type | Responsibility |
|----------|------|----------------|
| `createtab_stmt` | Hive DDL statement | Defines the schema, storage format, SerDe, location, and table properties for `org_details`. |
| `org_details` | External Hive table | Provides a lookup of organization metadata used by many downstream ETL/mediation jobs. |
| `LazySimpleSerDe` with `field.delim=','` | SerDe configuration | Parses the CSV source file into the three columns. |
| Table properties (`external.table.purge`, `STATS_GENERATED`, Impala catalog IDs, etc.) | Metastore metadata | Controls purge behavior, statistics generation, and Impala integration. |

*No procedural code (functions, classes) exists in this file; its sole purpose is schema definition.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Input Data** | A CSV file (or set of files) located at `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/org_details`. Each line must contain three fields: `orgno` (bigint), `orgname` (string), `org_abbr` (string). |
| **Output Artifact** | Hive external table `mnaas.org_details` registered in the Hive Metastore (and visible to Impala). |
| **Side Effects** | - Registers the table metadata in the metastore.<br>- Because `external.table.purge='true'`, a `DROP TABLE` will delete the underlying HDFS files.<br>- Updates Impala catalog version (`impala.events.catalogVersion`). |
| **Assumptions** | - HDFS path `NN-HA1` is reachable from the Hive/Impala service nodes.<br>- The CSV files conform to the defined delimiter and column order.<br>- No partitioning is required (the dataset is small and static).<br>- The Hive/Impala services have permission to read/write the target directory. |
| **External Dependencies** | - Hadoop NameNode `NN-HA1` (hostname may be supplied via environment variable or cluster config).<br>- Hive Metastore service.<br>- Impala catalog service (referenced by catalog IDs). |

---

## 4. Integration with Other Scripts & Components  

| Consuming Component | How It Uses `org_details` |
|---------------------|---------------------------|
| `mnaas_mnp_portinout_details_raw.hql` & related MNP scripts | Join on `orgno` to enrich port‑in/out records with organization names for reporting and reconciliation. |
| Billing aggregation scripts (`mnaas_month_billing.hql`, `mnaas_month_ppu_billing.hql`) | Map billing entries to the responsible organization for cost allocation. |
| SIM inventory status aggregation (`mnaas_move_sim_inventory_status_Aggr.hql`) | Resolve organization ownership of SIM assets. |
| Usage‑level aggregation views (`mnaas_msisdn_level_daily_usage_aggr_view.hql`, etc.) | Provide organization context for daily usage metrics. |
| Any downstream reporting layer (e.g., Tableau, PowerBI) | Pull `org_details` as a dimension table for drill‑down. |

*Because the table is external, any change to the underlying CSV (add/remove rows, schema change) is instantly visible to all consumers without a Hive `REFRESH` (except for Impala, which may need `INVALIDATE METADATA`).*

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Accidental data loss** – `external.table.purge='true'` causes underlying CSV files to be deleted on `DROP TABLE`. | Permanent loss of reference data. | Restrict `DROP TABLE` privileges to a limited admin role; enforce change‑control process for table recreation. |
| **Schema drift** – CSV file format changes (extra columns, different delimiter). | Query failures in downstream jobs. | Add a validation step (e.g., a Hive `EXTERNAL TABLE` with `skip.header.line.count=1` and a test query) in the CI pipeline before deploying changes. |
| **Permission issues** – Hive/Impala cannot read/write the HDFS location. | Table creation or query failures. | Verify HDFS ACLs and Kerberos tickets; document required permissions (`rwx` for the Hive service principal). |
| **Stale Impala metadata** – Impala may cache an older schema. | Inconsistent query results. | Run `INVALIDATE METADATA mnaas.org_details;` after any DDL change. |
| **Data freshness** – The CSV is static; updates require manual file replacement. | Out‑of‑date organization mapping. | Automate CSV refresh via a scheduled ingest job (e.g., Airflow task) that overwrites the directory and runs `MSCK REPAIR TABLE` (if partitioned in future). |

---

## 6. Example Execution & Debugging Workflow  

1. **Run the DDL**  
   ```bash
   hive -f mediation-ddls/mnaas_org_details.hql
   # or via Impala:
   impala-shell -i <impala-host> -f mediation-ddls/mnaas_org_details.hql
   ```

2. **Validate Creation**  
   ```sql
   SHOW CREATE TABLE mnaas.org_details;
   DESCRIBE FORMATTED mnaas.org_details;
   SELECT COUNT(*) FROM mnaas.org_details LIMIT 1;
   ```

3. **Check Data Presence**  
   ```bash
   hdfs dfs -ls /user/hive/warehouse/mnaas.db/org_details
   hdfs dfs -cat /user/hive/warehouse/mnaas.db/org_details/part-00000
   ```

4. **Debug Common Issues**  
   - *“Table not found”* → Verify Hive Metastore is reachable and that the `mnaas` database exists.  
   - *“Permission denied”* → Check HDFS ACLs for the Hive service principal.  
   - *“SerDe error”* → Ensure the CSV uses commas and no stray newline characters.  

5. **Refresh Impala (if needed)**  
   ```sql
   INVALIDATE METADATA mnaas.org_details;
   ```

---

## 7. External Configurations / Environment Variables  

| Config / Variable | Usage |
|-------------------|-------|
| `NN-HA1` (NameNode hostname) | Hard‑coded in the `LOCATION` clause; may be overridden by a cluster‑wide Hadoop configuration (`fs.defaultFS`). |
| Hive Metastore connection settings (`hive.metastore.uris`) | Required for the DDL to register the table. |
| Impala catalog service IDs (`impala.events.catalogServiceId`, `impala.events.catalogVersion`) | Populated automatically by Impala; no manual edit needed. |
| Optional: `HIVE_CONF_DIR` / `IMPALA_CONF_DIR` | Determines which configuration files are read when the script is executed. |

If the environment uses a different NameNode or a variable for the warehouse path, the script should be templated (e.g., using `${HDFS_WAREHOUSE}`) and rendered before execution.

---

## 8. Suggested TODO / Improvements  

1. **Add Table Comment & Column Descriptions** – Improves discoverability for downstream developers and reporting tools.  
   ```sql
   COMMENT 'Reference table mapping organization numbers to names and abbreviations';
   ALTER TABLE mnaas.org_details CHANGE orgno orgno BIGINT COMMENT 'Unique organization identifier';
   ```

2. **Introduce a Controlled Refresh Process** – Wrap the CSV replace and table metadata refresh in an Airflow DAG or similar scheduler to guarantee atomic updates and automatic Impala metadata invalidation.  

---