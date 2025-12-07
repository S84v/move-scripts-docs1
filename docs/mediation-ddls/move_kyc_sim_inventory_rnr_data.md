**File:** `mediation-ddls\move_kyc_sim_inventory_rnr_data.hql`  

---

## 1. High‑Level Summary
This HiveQL script creates an **external Hive table** named `move_kyc_sim_inventory_rnr_data` in the `mnaas` database. The table stores KYC‑related SIM inventory records (VIN, ICCID, plan details, RNR status, vehicle information, etc.) as plain‑text files located on HDFS under `/user/hive/warehouse/mnaas.db/move_kyc_sim_inventory_rnr_data`. Because the table is external and marked with `external.table.purge='true'`, Hive will not retain data after a DROP, and the underlying files will be deleted automatically. The definition is used by downstream “move” jobs that read, enrich, or export KYC inventory data for billing, provisioning, or compliance reporting.

---

## 2. Important Objects & Their Responsibilities
| Object | Type | Responsibility |
|--------|------|----------------|
| `move_kyc_sim_inventory_rnr_data` | External Hive table | Persists raw KYC SIM inventory records in a column‑wise string schema; serves as the canonical source for downstream ETL jobs. |
| `ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'` | SerDe | Parses delimited text (default delimiter `\001`) into Hive columns. |
| `STORED AS INPUTFORMAT/OUTPUTFORMAT` | File format spec | Uses Hadoop TextInputFormat and HiveIgnoreKeyTextOutputFormat (plain text, no key). |
| `TBLPROPERTIES` (e.g., `external.table.purge`, `impala.*`) | Metadata | Controls purge behavior, Impala catalog integration, and audit timestamps. |

*No procedural code (functions, procedures) is present; the file is a pure DDL definition.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions
| Category | Details |
|----------|---------|
| **Inputs** | - No runtime input parameters. <br> - Implicitly depends on the HDFS directory `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/move_kyc_sim_inventory_rnr_data` existing and being accessible to the Hive/Impala service accounts. |
| **Outputs** | - Hive metastore entry for the external table. <br> - No data is written at creation time; downstream jobs will populate the directory with text files. |
| **Side‑Effects** | - Registers the table in the Hive metastore (visible to Impala). <br> - Because `external.table.purge='true'`, a future `DROP TABLE` will **delete** the underlying HDFS files. |
| **Assumptions** | - All columns are stored as `string`; downstream processes will cast/convert as needed. <br> - The HDFS namenode alias `NN-HA1` resolves correctly in the cluster’s network/DNS. <br> - The default field delimiter (`\001`) matches the producer’s output format. <br> - No partitioning is required for the current data volume. |

---

## 4. Integration with Other Scripts / Components
| Connected Component | Interaction |
|---------------------|-------------|
| **Downstream “move” jobs** (e.g., `move_kyc_sim_inventory_load.hql`, `move_kyc_sim_inventory_transform.py`) | Read from this table to enrich with plan pricing, generate billing records, or push to external KYC APIs. |
| **Impala** | The table is visible to Impala via the catalog service (`impala.events.catalogServiceId`), enabling low‑latency SQL queries for reporting dashboards. |
| **Data Ingestion Pipelines** (e.g., Kafka → Flume → HDFS) | Produce raw text files into the HDFS location; the external table automatically reflects new files. |
| **Data Quality / Validation Scripts** (e.g., `validate_kyc_inventory.sql`) | Query the table to verify row counts, nullability, or format compliance before downstream processing. |
| **Retention / Archival Jobs** | May `DROP TABLE` or move files; because of `purge=true`, they must be aware that dropping will erase data. |

*Note:* The exact script names are not present in the repository snapshot; you would locate them by searching for `move_kyc_sim_inventory_` in the code base.

---

## 5. Operational Risks & Recommended Mitigations
| Risk | Impact | Mitigation |
|------|--------|------------|
| **Accidental data loss** – `external.table.purge='true'` causes file deletion on DROP. | Permanent loss of raw KYC inventory. | Restrict DROP privileges; enforce change‑management approval; consider removing `purge` until a robust backup/archival process is in place. |
| **Schema drift** – All columns are `string`; downstream casts may fail if source data changes (e.g., new columns). | Job failures, silent data corruption. | Add column comments; version the table (e.g., `move_kyc_sim_inventory_rnr_data_v2`); implement schema validation step in ingestion pipeline. |
| **Incorrect delimiter** – Default `\001` may not match producer output. | Mis‑parsed rows, null fields. | Document the expected delimiter; optionally set `FIELDS TERMINATED BY ','` if CSV is used. |
| **Missing HDFS path or permission issues** – Table creation succeeds but reads fail. | Downstream jobs stall. | Verify HDFS directory existence and ACLs for Hive/Impala service accounts during deployment. |
| **Impala catalog out‑of‑sync** – Catalog version mismatch may cause stale metadata. | Queries return empty or old data. | Schedule periodic `INVALIDATE METADATA` or `REFRESH` after data load. |

---

## 6. Running / Debugging the Script
1. **Execution**  
   ```bash
   # Using Hive CLI
   hive -f mediation-ddls/move_kyc_sim_inventory_rnr_data.hql

   # Or via Beeline (recommended)
   beeline -u jdbc:hive2://<hive-host>:10000/mnaas -f mediation-ddls/move_kyc_sim_inventory_rnr_data.hql
   ```
   *No parameters are required.*

2. **Verification**  
   ```sql
   SHOW CREATE TABLE mnaas.move_kyc_sim_inventory_rnr_data;
   DESCRIBE FORMATTED mnaas.move_kyc_sim_inventory_rnr_data;
   SELECT COUNT(*) FROM mnaas.move_kyc_sim_inventory_rnr_data LIMIT 10;
   ```
   - Confirm the `Location` points to the expected HDFS path.  
   - Verify that `STORED AS` and `SERDE` match the producer format.

3. **Debugging Tips**  
   - **Metadata issues:** Run `INVALIDATE METADATA mnaas.move_kyc_sim_inventory_rnr_data;` in Impala.  
   - **File visibility:** `hdfs dfs -ls /user/hive/warehouse/mnaas.db/move_kyc_sim_inventory_rnr_data` to ensure files exist and have correct permissions.  
   - **Parsing problems:** Sample a raw file locally and test with `hive -e "SELECT * FROM mnaas.move_kyc_sim_inventory_rnr_data LIMIT 5"`; check for `NULL` columns.  

---

## 7. External Configuration / Environment Dependencies
| Item | Usage |
|------|-------|
| `NN-HA1` (namenode alias) | Part of the HDFS URI; must resolve via DNS or `/etc/hosts` on all Hive/Impala nodes. |
| Hive Metastore connection (usually via `hive-site.xml`) | Required for table registration. |
| Impala catalog service ID (`b9c115786e934562:94a82089945b48d6`) | Auto‑populated; used by Impala to track metadata changes. |
| Environment variables (e.g., `HIVE_CONF_DIR`, `HADOOP_CONF_DIR`) | Needed for CLI/Beeline to locate configuration files. |
| Optional: `hive.exec.scratchdir` | Determines temporary staging area for DDL execution. |

If any of these are overridden in a deployment script (e.g., using a `*.properties` file), the overrides must be reviewed to ensure they point to the correct cluster resources.

---

## 8. Suggested TODO / Improvements
1. **Add proper data types & partitioning** – Replace generic `string` columns with appropriate types (`bigint` for attempts, `timestamp` for dates) and consider partitioning by `plan_date` or `countryofsale` to improve query performance and manageability.  
2. **Document and enforce the field delimiter** – Explicitly set `FIELDS TERMINATED BY ','` (or the actual delimiter) in the DDL and add a comment describing the expected source file format to avoid future parsing mismatches.  

---